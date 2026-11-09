# Модуль 8: Реалтайм, push, работа в фоне

**[← Назад к индексу](../index.md)** | **[Модуль 7: Гео и карты](../Module_07_Geo_Maps/content.md)** | **Далее: [Модуль 9: Платежи](../Module_09_Payments/content.md) →**

---

## 🎯 Цель модуля

- **Socket.IO**: живое отслеживание водителя, статусы поездки, reconnect
- **Машинa состояний** поездки (finite state machine)
- **Push** (`expo-notifications`): foreground-баннер + terminated → deep link
- **Фоновый трекинг**: Android foreground service, iOS background location, батарея
- **Плавность**: интерполяция маркера через Reanimated (60fps)

**Длительность: ~30 часов. Самый «живой» модуль.**

---

## 📚 Теория

### 8.1. WebSocket + Socket.IO: почему и как

**Чистый WebSocket** — вы сами: reconnect, heartbeat, rooms, auth. **Socket.IO** — библиотека поверх WS: из коробки **reconnect с backoff**, **rooms**, **acknowledgements**, **heartbeat**, **fallback** (long-polling). Для ride-app — Socket.IO.

```bash
npx expo install socket.io-client
```

**Бэкенд (M4 Express):** `socket.io` сервер. **Комната** на каждый заказ: `order:{id}`. Подписываются **и пассажир, и водитель** (в ммоке — один клиент, но комната всё равно).

### 8.2. Подключение с auth + reconnect

```ts
// src/features/ride/socket.ts
import { io, Socket } from 'socket.io-client';

let socket: Socket | null = null;

export function getSocket(): Socket {
  if (socket) return socket;

  socket = io(API_URL, {
    auth: { token: () => tokenManager.getAccess() },  // токeн на handshake
    transports: ['websocket', 'polling'],
    reconnection: true,
    reconnectionAttempts: Infinity,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 10_000,   // экспоненциальный backoff
    randomizationFactor: 0.5,
  });

  socket.on('connect', () => { /* log: подключено */ });
  socket.on('disconnect', (reason) => { /* log: потеряно, показываем «переподключаемся» */ });
  socket.on('connect_error', (e) => { /* 401 → refresh и переподключение */ });

  return socket;
}

// Подписка на заказ (M8.3): idempotent, с cleanup
export function subscribeToOrder(rideId: string, handlers: OrderHandlers): () => void {
  const s = getSocket();
  s.emit('order:join', { rideId });
  const onStatus = (d: StatusEvent) => handlers.onStatus?.(d);
  const onDriver = (d: DriverPosEvent) => handlers.onDriver?.(d);
  s.on(`order:${rideId}:status`, onStatus);
  s.on(`order:${rideId}:driver`, onDriver);
  return () => {
    s.off(`order:${rideId}:status`, onStatus);
    s.off(`order:${rideId}:driver`, onDriver);
    s.emit('order:leave', { rideId });
  };
}
```

**Принципы:**
- **Один сокет на приложение** (не по экрану!) — переиспользуем между заказами
- **Auth на handshake** (токeн) — сервер знает, кто вы
- **Reconnect** — из коробки; при переподключении **запрашиваем свежий стейт** (`order:sync`)
- **Cleanup подписок** при unmount (иначе утечка)
- **Heartbeat** (ping/pong) — Socket.IO держит; мёртвое соединение детектируется

### 8.3. Машина состояний поездки (FSM)

**Ядро модуля.** Статусы поездки:

```
CREATED → ACCEPTED → ARRIVED → STARTED → COMPLETED
   │         │          │          │
   └─────────┴──────────┴──────────┴──→ CANCELLED (из любого)
```

```ts
// src/features/ride/state.ts — ЧИСТАЯ ФУНКЦИЯ, тестируется
export type RideStatus = 'CREATED' | 'ACCEPTED' | 'ARRIVED' | 'STARTED' | 'COMPLETED' | 'CANCELLED';

const TRANSITIONS: Record<RideStatus, RideStatus[]> = {
  CREATED:   ['ACCEPTED', 'CANCELLED'],
  ACCEPTED:  ['ARRIVED', 'CANCELLED'],
  ARRIVED:   ['STARTED', 'CANCELLED'],
  STARTED:   ['COMPLETED', 'CANCELLED'],
  COMPLETED: [],   // терминальное
  CANCELLED: [],   // терминальное
};

export function canTransition(from: RideStatus, to: RideStatus): boolean {
  return TRANSITIONS[from]?.includes(to) ?? false;
}

export function applyEvent(current: RideStatus, event: RideStatus): RideStatus {
  // Игнорируем невалидные/ретранслированные события (идемпотентность)
  if (current === event) return current;
  return canTransition(current, event) ? event : current;
}
```

**Зачем FSM (а не `if status === ...`):**
- **Одно место** с правилами → легко читать и тестировать
- **Идемпотентность**: повторное событие (ретрансляция после reconnect) не «ломает» стейт
- **Предсказуемость**: «из COMPLETED никуда нельзя» — зафиксировано

```ts
// Хук: состояние + подписка + reconnect-sync
export function useRide(rideId: string | undefined) {
  const [status, setStatus] = useState<RideStatus>('CREATED');
  const [driverPos, setDriverPos] = useState<LatLng | null>(null);
  const statusRef = useRef(status);
  statusRef.current = status;

  useEffect(() => {
    if (!rideId) return;
    const unsub = subscribeToOrder(rideId, {
      onStatus: (e) => setStatus((s) => applyEvent(s, e.status)),   // через FSM
      onDriver: (e) => setDriverPos({ lat: e.lat, lng: e.lng, heading: e.heading }),
    });
    // При reconnect — синхронизация (M8.2)
    getSocket().on('connect', () => {
      getSocket().emit('order:sync', { rideId }, (snap: OrderSnapshot) => {
        setStatus(applyEvent(statusRef.current, snap.status));
        setDriverPos(snap.driverPos);
      });
    });
    return unsub;
  }, [rideId]);

  return { status, driverPos };
}
```

### 8.4. Плавный маркер водителя (Reanimated, 60fps)

Водитель шлёт позицию **раз в 1–3 с** (не каждый кадр!). Чтобы маркер ехал **плавно**, **интерполируем** между точками:

```tsx
// src/features/ride/DriverMarker.tsx
import Animated, { useSharedValue, useAnimatedStyle, withTiming, runOnJS } from 'react-native-reanimated';

export function DriverMarker({ target }: { target: LatLng | null }) {
  const lat = useSharedValue(0);
  const lng = useSharedValue(0);
  const animating = useRef(false);

  useEffect(() => {
    if (!target) return;
    // Плавно двигаем к новой точке за время ≈ (расстояние / скорость)
    const duration = estimateMoveDuration(lat.value, lng.value, target); // 800мс–3с
    // Анимация в UI-потоке (worklet) → 60fps, не блокируем JS
    Animated.parallel([
      Animated.timing? // через useSharedValue + withTiming для lat и lng
    ]);
    // (упрощённо: withTiming на координаты; карта обновляет marker)
  }, [target]);

  // MapView-маркер, привязанный к анимированным координатам
}
```

> 💡 **Ключ:** движение маркера — **worklet** (UI-поток). JS-поток свободен для логики. Между «скачками» реальных GPS-точек — плавная интерполяция. Это и есть «водитель едет», а не «телепортируется».

### 8.5. Push: expo-notifications

```bash
npx expo install expo-notifications
```

**Три состояния приложения при push (запомнить):**

| Состояние | Что делаем |
|-----------|-----------|
| **Foreground** (открыто) | Баннер **наш** (внутри UI), не системный |
| **Background** (свёрнуто) | Системный баннер → тап = открыть в нужный экран |
| **Terminated** (убито) | Системный баннер → тап = **запустить** и открыть экран (deep link) |

```ts
// src/features/notifications/handler.ts
import * as Notifications from 'expo-notifications';

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: false,
    shouldSetBadge: false,
    // Foreground: НЕ системный баннер — покажем свой (M8.6)
    shouldShowBanner: false,
    shouldShowList: true,
  }),
});
```

**Токeн → бэкенд:**
```ts
export async function registerPushToken() {
  const { status } = await Notifications.getPermissionsAsync();
  if (status !== 'granted') {
    const req = await Notifications.requestPermissionsAsync();
    if (req.status !== 'granted') return;
  }
  const token = await Notifications.getExpoPushTokenAsync();
  await api.post('/notifications/token', { token: token.data, platform: Platform.OS });
}
```

**Приём** (foreground-свой баннер + terminated-deep link):
```ts
// Свой баннер (foreground):
const sub = Notifications.addNotificationResponseReceivedListener((resp) => {
  const data = resp.notification.request.content.data;   // { type: 'driver', rideId, ... }
  // data.rideId → на навигацию в Order (deep link M8.6)
  if (resp.origin === 'notification') {
    navigationRef.current?.navigate('Order', { id: data.rideId });
  }
});
// Foreground-событие (показали баннер handlerом):
Notifications.addNotificationReceivedListener((n) => {
  if (appInForeground) showInAppBanner(n.request.content.data);   // M8.6
});
```

### 8.6. Deep link из push → экран заказа

```ts
// app/navigation links: Order уже имеет href '/order/:id' (M3.8)
// push-data → deep link:
export function pushToLink(data: PushData): string {
  // { type: 'driver_assigned', rideId: '123' } → 'ridego://order/123'
  return `ridego://${pathFor(data.type)}/${data.rideId}`;
}
```
**Три сценария** (M8.5) → все три должны вести на `Order?rideId`:
1. Foreground → свой баннер → тап → навигация
2. Background → системный баннер → тап → открыть → navigation (cold/warm)
3. Terminated → запуск → `onOpenURL` → navigation.reset на Order

> ⚠️ **navigationRef** (M3) — теперь реально используется: deep link/push → навигация извне дерева.

### 8.7. Фоновый трекинг: когда и почему

**Задача:** после `STARTED` (поехали) — **отслеживать позицию пассажира** в фоне (безопасность, ETA, трек поездки). Приложение может быть свёрнуто.

**Android — foreground service** (обязательно, иначе OS убьёт):
```
app.json → android → "backgroundMode": ["location"]
+ foreground service с notification «RideGo отслеживает вашу поездку» (обязатeльна!)
```
(В Expo — через config plugin / `react-native-foreground-service`. Для курса: включаем `backgroundMode: location` + показываем служебное уведомление.)

**iOS — background location:**
```
app.json → ios → infoPlist → "UIBackgroundModes": ["location"]
+ `Location.hasEnabledBackgroundLocation` + `allowsBackgroundLocationUpdates: true`
```
**Ограничение iOS:** фоновая геолокация — **лимиты** (система может «уснуть»). Не полагаться полностью — дублируем через push (M8) как fallback.

**Батарея — главный враг:**
- Фоновый GPS — **дорого**. Включаем **только** на `STARTED → COMPLETED` (не всегда!)
- `distanceInterval` крупный (20–50м) в фоне
- По `COMPLETED` — **немедленно** останавливаем фоновый режим (`Location.stop...`/service stop)

### 8.8. Battery-осознанность (чек-лист)

| Действие | Батарея |
|----------|---------|
| watchPosition High, всегда | 🔴 высокая |
| watchPosition High, distanceInterval 5м, только при активной поездке | 🟡 умеренно |
| Фоновый GPS 20–50м, только STARTED→COMPLETED | 🟡🟢 приемлемо |
| WS heartbeat | 🟢 низкая |
| Push (системный, без собственного GPS) | 🟢 минимально |

**Принцип:** фоновые вещи (GPS, WS) — **только когда нужны** (поездка активна), **немедленно останавливаем** после.

### 8.9. Экран заказа (OrderScreen) — собираем M7+M8

```tsx
export function OrderScreen({ route: { params: { id } } }) {
  const { status, driverPos } = useRide(id);            // M8.3/8.4
  const { data: ride } = useQuery({ queryKey: ['ride', id], queryFn: () => api.get(`/rides/${id}`) });

  return (
    <SafeAreaView style={flex1}>
      <MapView ...>
        <Marker coordinate={driverPos ?? myPos} /* плавный, M8.4 */ icon={car} />
        <Marker coordinate={to} />
      </MapView>

      {/* Статус-панель (по FSM) */}
      <StatusPanel status={status} driver={ride?.driver} />
      {/* ACCEPTED → «Водитель едет к вам» + ETA; ARRIVED → «Прибыл»; STARTED → «В пути» */}

      {/* Reconnect-индикатор: если socket disconnected → плашка «Переподключаемся…» */}

      {status === 'STARTED' && <BackgroundTrackingBanner />}   // M8.7
      {status === 'COMPLETED' && navigation.navigate('Payment', { rideId: id })}  // M9
    </SafeAreaView>
  );
}
```

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Мок-бэкенд: реалтайм (8ч)
- [ ] Socket.IO сервер; room `order:{id}`; auth на handshake
- [ ] **Генератор водителя**: при `ACCEPTED` — «водитель» едет к passenger по полилинии (setInterval, 1–2с/точка), шлёт `driver` события; затем `ARRIVED` → ждёт → `STARTED` → везёт к `to` → `COMPLETED`
- [ ] Статусы: `order:sync` (свежий срез), ротация: `CREATED→ACCEPTED` через ~5с
- [ ] **Push**: `POST /notifications/token`; при событии → `POST https://api.expo.dev/v2/push` (Expo push service)
- [ ] Сценарии-кнопки (dev): «продвигать статус», «оборвать WS» (тест reconnect)

### Шаг 2. Socket + FSM (8ч)
- [ ] `getSocket` (M8.2) + auth handshake + reconnect backoff
- [ ] `useRide` (M8.3) + FSM (M8.3) + reconnect-sync
- [ ] **Unit-тесты FSM**: все переходы + невалидные + идемпотентность (повтор)
- [ ] **Reconnect-тест**: «оборвать WS» (кнопка dev) → плашка → переподключение → sync → стейт сохранён

### Шаг 3. Плавный маркер (4ч)
- [ ] `DriverMarker` (M8.4): интерполяция, 60fps (Reanimated worklet)
- [ ] ETA (обновляется по статусу/позиции)

### Шаг 4. Push (6ч)
- [ ] `registerPushToken` (M8.5) при auth (M6) → бэкенд
- [ ] Handler: foreground → **свой** баннер (M8.6); background/terminated → системный → deep link → Order
- [ ] **Три сценария** на живом устройстве (foreground/background/terminated)
- [ ] `pushToLink` + `navigationRef` → Order (M8.6)

### Шаг 5. Фоновый трекинг (4ч)
- [ ] Фоновый GPS **только** STARTED→COMPLETED (M8.7), banner, остановка
- [ ] Android foreground service + iOS background location
- [ ] Battery-чек: замер «до/после» (опционально, M10)

### Шаг 6. Отчёт (2ч) — `docs/progress/M8.md`
- Скриншоты: live-трекинг, свой баннер, reconnect-плашка
- FSM-тесты (результаты)
- Три push-сценария (видео/скрины)
- Ошибки: 3+ (включая reconnect + «iOS background лимиты»)

---

## ✅ Практическое задание модуля

### Задача: «Живой трекинг + push + фон»

**1. Мок-bэкенд (8ч)** — Socket.IO + генератор водителя + push (Expo) + dev-кнопки

**2. Socket + FSM (8ч)**
- [ ] getSocket (auth, reconnect backoff) + useRide + FSM
- [ ] **Unit-тесты FSM** (все переходы, идемпотентность) + reconnect-sync

**3. Плавный маркер (4ч)** — интерполяция, **60fps** (worklet)

**4. Push (6ч)**
- [ ] Токeн → бэкенд; handler (foreground-свой баннер); три сценария → Order (deep link)

**5. Фон (4ч)** — фоновый GPS STARTED→COMPLETED, banner, остановка; Android+iOS

**6. Отчёт (2ч)**

**Критерии:**
- [ ] Водитель едет **плавно** (60fps), не телепортируется
- [ ] Reconnect: оборвали WS → плашка → sync → стейт не потерян
- [ ] Push: **три** сценария (foreground/background/terminated) → Order
- [ ] Фоновый трекинг: только на активной поездке, остановка после
- [ ] FSM: все статусы, невалидные события игнорируются (тесты)
- [ ] На обеих платформах

---

## 📦 Итоговая структура после M8

```
src/features/
├── ride/
│   ├── socket.ts           # getSocket, subscribeToOrder
│   ├── state.ts            # FSM (чистая)
│   ├── useRide.ts          # хук: статус + позиция
│   ├── DriverMarker.tsx    # плавный (Reanimated)
│   └── OrderScreen.tsx     # карта + статус + фон
└── notifications/
    ├── handler.ts          # setNotificationHandler
    ├── register.ts         # токен → бэкенд
    ├── InAppBanner.tsx     # foreground-баннер
    └── deepLink.ts         # pushToLink
src/features/geo/
└── backgroundTracking.ts   # STARTED→COMPLETED
```

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Сокeт на экран | Мultiple соединения | Один сокет на приложение |
| Нет reconnect | Потерял сигнал → «застряло» | reconnection + backoff + sync |
| Стейт без FSM | Ретрансляция → «задний ход» | applyEvent (идемпотентно) |
| setState на каждый кадр | Lag | Reanimated worklet |
| Фоновый GPS всегда | Батарея тает | Только STARTED→COMPLETED |
| Push foreground = системный | Двойной баннер | handler shouldShowBanner: false + свой |
| Terminated → нет навигации | Push «просто открыл» | onOpenURL → navigationRef |
| Утечка подписок WS | Память | cleanup unsub |
| iOS background «заснул» | Трек пропадает | Push-fallback + sync |

---

## 💡 Советы на модуль

- **FSM — центр.** Один файл правил, покрыт тестами. Это ваш «мозг» поездки.
- **Плавность = worklet.** Маркер двигается в UI-потоке. 60fps.
- **Push — 3 состояния.** Foreground/background/terminated. Тестируйте **все три** на живом.
- **Батарея — ресурс.** Фоновые вещи — только когда нужны, немедленно останавливаем.
- **Reconnect — норма.** Сеть дрогнёт — должны синхронизироваться, а не «ломаться».

**[→ Модуль 9: Платежи и оценка](../Module_09_Payments/content.md)**
