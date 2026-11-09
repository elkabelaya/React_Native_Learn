# Модуль 7: Гео и карты

**[← Назад к индексу](../index.md)** | **[Модуль 6: Auth](../Module_06_Auth_Security/content.md)** | **Далее: [Модуль 8: Реалтайм и push](../Module_08_Realtime_Push/content.md) →**

---

## 🎯 Цель модуля

- **Dev build** (первый настоящий — Expo Go не умеет maps/location нативно)
- **Permissions** геолокации: rationale, denied → settings, обе платформы
- **react-native-maps**: карта, маркеры, перетаскиваемый пин, reverse-геокодинг
- **Выбор «откуда» и «куда»**: карта + поиск-автокомплит
- **Расчёт цены** по дистанции (чистая функция — тестируется)
- **Полный флоу вызова**: от → к → классы → цена → «Вызвать»

**Длительность: ~30 часа. Сначала dev build, потом карта.**

---

## 📚 Теория

### 7.0. Dev build — переход на «взрослый» режим

**Expo Go не содержит** нативные модули, которых нет в его сборке (maps, location, keychain-с Some flags, notifications-в фоне). С M7 **весь курс** — на dev build.

```bash
# один раз (20–40 минут)
npx expo run:ios          # соберёт в Xcode и поставит на симулятор
npx expo run:android      # соберёт APK/AAB и поставит на эмулятор/устройство
# теперь: npx expo start → приложение-DevClient сканирует QR (или авто-подключение)
```

**DevClient** — ваше приложение с отладкой (fast refresh, console, DevTools) + все нативные модули. Для прода — `eas build` (M13).

> 💡 Держите **оба** dev build (iOS + Android) постоянно на устройствах — переключаетесь по QR. Это «новая нормальная среда».

### 7.1. Permissions: геолокация

```bash
npx expo install expo-location react-native-maps
```

**app.json** (config plugin впишет в Info.plist / AndroidManifest):

```json
{
  "expo": {
    "plugins": [
      [
        "expo-location",
        {
          "locationWhenInUsePermission": "RideGo использует геолокацию, чтобы показать вашу позицию и построить маршрут."
        }
      ]
    ],
    "ios": {
      "infoPlist": {
        "NSLocationWhenInUseUsageDescription": "RideGo использует геолокацию, чтобы показать вашу позицию."
        // M8 добавит: UIBackgroundModes: ["location"] для трекинга
      }
    },
    "android": {
      "permissions": ["ACCESS_COARSE_LOCATION", "ACCESS_FINE_LOCATION"]
    }
  }
}
```

> ⚠️ **После правок app.json** → `npx expo prebuild` (dev build) — иначе нативные файлы не обновятся. **Никогда не правьте ios/ и android/ руками.**

**Запрос разрешения (UX-паттерн):**

```ts
// src/features/geo/permissions.ts
import * as Location from 'expo-location';
import { Linking, Platform } from 'react-native';

export async function ensureLocationPermission(): Promise<'granted' | 'denied' | 'undetermined'> {
  // 1. Уже дано?
  const { status } = await Location.getForegroundPermissionsAsync();
  if (status === 'granted') return 'granted';

  // 2. Пока не решали — показываем СВОЙ rationale, потом системный
  if (status === 'undetermined') {
    // (опционально: свой BottomSheet с объяснением → затем request)
    const req = await Location.requestForegroundPermissionsAsync();
    return req.status as 'granted' | 'denied';
  }

  // 3. Отказали ранее — системный промпт больше не покажем.
  //    Единственный путь — отправить в настройки.
  return 'denied';
}

export async function openLocationSettings() {
  if (Platform.OS === 'ios') {
    await Linking.openURL('app-settings:');   // настройки приложения
  } else {
    await Linking.openURL('intent:#Intent;action=android.settings.APPLICATION_DETAILS_SETTINGS;package=' +
      /* app package */ + ';END');
  }
}
```

**Принципы permissions (модуль и дальше):**
- **Lazy**: просим **в момент необходимости** (открыли карту), не при запуске
- **Rationale**: объясняем **зачем** (свой экран/шيت → потом системный)
- **Denied → settings**: если отказали — даём кнопку «Открыть настройки» (единственный путь)
- **Не блокируем приложение** без разрешения: показываем экран «Включите геолокацию» с CTA

### 7.2. expo-location: получить позицию

```ts
// src/features/geo/location.ts
import * as Location from 'expo-location';
import { useCallback, useEffect, useState } from 'react';

export function useMyLocation(
  active: boolean,
  opts: { accuracy?: Location.Accuracy; distanceInterval?: number } = {},
) {
  const [position, setPosition] = useState<Location.LocationObject | null>(null);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    if (!active) return;
    let sub: Location.LocationSubscription | null = null;
    let cancelled = false;

    (async () => {
      const { status } = await Location.getForegroundPermissionsAsync();
      if (status !== 'granted') { setError(new Error('DENIED')); return; }

      // Мгновенный снимок
      const cur = await Location.getCurrentPositionAsync({
        accuracy: opts.accuracy ?? Location.Accuracy.Balanced,
      });
      if (!cancelled) setPosition(cur);

      // Подписка (живое обновление)
      sub = await Location.watchPositionAsync(
        {
          accuracy: opts.accuracy ?? Location.Accuracy.High,
          distanceInterval: opts.distanceInterval ?? 5, // обновлять, если сдвинулись > 5 м
        },
        (loc) => { if (!cancelled) setPosition(loc); },
      );
    })().catch((e) => !cancelled && setError(e as Error));

    return () => {
      cancelled = true;
      sub?.remove();   // ОБЯЗАТЕЛЬНО: иначе утечка + фоновый GPS
    };
  }, [active]);

  return { position, error };
}
```

**Accuracy:** `Low` (~2км, экономия), `Balanced` (~100м), `High` (~3–5м). Для карты «где я» — `Balanced`; для «указать точку на пороге» — `High`.

### 7.3. react-native-maps: базовый набор

```tsx
import MapView, { Marker, Polyline, Region, PROVIDER } from 'react-native-maps';

const MOSCOW: Region = { latitude: 55.7558, longitude: 37.6173, latitudeDelta: 0.05, longitudeDelta: 0.05 };

<MapView
  style={flex1}
  initialRegion={MOSCOW}
  provider={Platform.OS === 'android' ? PROVIDER_GOOGLE : undefined} // Android: Google Maps (нужен API key)
  showsUserLocation
  userLocationTrackingMode={UserLocationTrackingMode.follow}   // камера следует за пользователем
  onRegionChangeComplete={(r) => { /* reverse geocode центра — M7.4 */ }}
>
  <Marker coordinate={myPos} title="Вы здесь" />
  <Marker coordinate={dest} title="Куда">
    <Image source={carIcon} style={{ width: 40, height: 40 }} />
  </Marker>
  <Polyline coordinates={routeCoords} strokeColor="#111" strokeWidth={4} />
</MapView>
```

**Android: Google Maps API key** — создать в Google Cloud Console → enable Maps SDK for Android → **ограничить key** (SHA-1 вашего dev/app) → в `app.json`:
```json
"android": { "googleMapsApiKey": { "debug": "...", "release": "..." } }
```
**iOS:** MapKit — ключ не нужен. (С этим разберётесь в dev build; без ключа Android-карта «серая».)

### 7.4. AddressPicker: карта + поиск + перетаскиваемый пин

**Два способа задать точку:**
1. **Поиск** (автокомплит адресов) → тап → пин на карте
2. **Перетаскивание** пина вручную → reverse geocode → адрес

```tsx
// src/features/geo/AddressPicker.tsx
export function AddressPicker({
  initial, onPick,
}: { initial?: GeoPoint; onPick: (p: GeoPoint & { label: string }) => void }) {
  const [region, setRegion] = useState<Region>(MOSCOW);
  const [pin, setPin] = useState<LatLng>(initial ?? regionCenter);
  const [q, setQ] = useState('');
  const [suggestions, setSuggestions] = useState<Address[]>([]);
  const dragging = useRef(false);

  // Поиск с debounce + geocoding (M7.5)
  useEffect(() => {
    if (q.length < 3) { setSuggestions([]); return; }
    const t = setTimeout(() => {
      geocode(q).then(setSuggestions).catch(() => setSuggestions([]));
    }, 300);
    return () => clearTimeout(t);
  }, [q]);

  return (
    <View style={flex1}>
      {/* Поиск поверх карты */}
      <SafeAreaView edges={['top']}>
        <Input value={q} onChangeText={setQ} placeholder="Найти адрес" testID="addr-search" />
        {suggestions.length > 0 && (
          <FlatList data={suggestions} keyExtractor={(a) => a.id}
            renderItem={({ item }) => (
              <Pressable onPress={() => { setPin({ latitude: item.lat, longitude: item.lng }); setRegion(zoomTo(item)); onPick({ ...item, label: item.name }); }}>
                <Text>{item.name}</Text>
              </Pressable>
            )}
          />
        )}
      </SafeAreaView>

      <MapView
        style={flex1}
        region={region}
        onRegionChangeComplete={setRegion}
        onMapLongPress={(e) => setPin({ latitude: e.nativeEvent.coordinate.latitude, longitude: e.nativeEvent.coordinate.longitude })}
        onMarkerDragStart={() => (dragging.current = true)}
        onMarkerDrag={(e) => setPin(e.nativeEvent.coordinate)}
        onMarkerDragEnd={async (e) => {
          dragging.current = false;
          const label = await reverseGeocode(e.nativeEvent.coordinate);   // M7.5
          onPick({ ...e.nativeEvent.coordinate, label });
        }}
      >
        <Marker
          coordinate={pin}
          draggable
          pinColor="orange"
          title="Перетащите на точку"
        />
      </MapView>

      <Button title="Подтвердить" disabled={!pin} onPress={() => onPick({ ...pin, label: pin.label ?? 'Выбранная точка' })} testID="addr-confirm" />
    </View>
  );
}
```

**Запомните паттерн:** `onMapLongPress` → поставить пин; `onMarkerDrag*` → двигать; `onRegionChangeComplete` → (reverse geocode / подгрузить тайлы). Это 90% работы с картой.

### 7.5. Геокодинг и поиск

**Reverse geocode** (координаты → адрес):
```bash
npx expo install expo-geocoding
```
```ts
import * as Geocoding from 'expo-geocoding';
export async function reverseGeocode(c: LatLng): Promise<string> {
  try {
    const res = await Geocoding.reverseGeocodeAsync(c);
    return res[0]?.placemark?.name ?? `${c.latitude.toFixed(5)}, ${c.longitude.toFixed(5)}`;
  } catch {
    return `${c.latitude.toFixed(5)}, ${c.longitude.toFixed(5)}`;   // фолбэк: координаты
  }
}
```

**Forward geocode / search** (адрес → координаты): `expo-geocoding` умеет `geocodeAsync`, но для **автокомплита** лучше **бэкенд** (молниеносный поиск по своей базе + кеширование):
```
GET /geo/search?q=тверская&limit=5 → [{ id, name, lat, lng, geoHash }]
```
(В мок-бэкенде M7 — прокси к OpenStreetMap Nominatim **или** фикс-база адресов вашего города. Nominatim имеет rate-limit — для курса достаточно фикс-базы 20–50 адресов.)

**geoHash** (сохранённые адреса M5): короткий строковый ключ геоточки (~100м) — для быстрых «рядом»-запросов. Опционально.

### 7.6. Маршрут и расстояние

**Маршрут (полилиния)** — из моста:
- **Google Directions / OSRM /HERE** (реальные) — но для курса: **мост генерит полилинию** (плавная кривая/ломаная между точками) + расстояние. Этого достаточно для UI.
- Расстояние: **геодезическое** (haversine) — чистая функция:

```ts
// src/features/geo/geo.ts — ЧИСТАЯ, тестируется
const R = 6371000; // м
export function haversine(a: LatLng, b: LatLng): number {
  const toRad = (d: number) => (d * Math.PI) / 180;
  const dLat = toRad(b.latitude - a.latitude);
  const dLng = toRad(b.longitude - a.longitude);
  const s =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(toRad(a.latitude)) * Math.cos(toRad(b.latitude)) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(s));
}

// «Дорожное» расстояние ≈ хaversine × коэффициент извилистости
export function roadDistance(a: LatLng, b: LatLng): number {
  return Math.round(haversine(a, b) * 1.3);
}
```

### 7.7. Расчёт цены (чистая функция — сердце модуля)

```ts
// src/features/map/price.ts — ЧИСТАЯ, без副作用, тестируется
export type RideClass = {
  id: 'ECONOMY' | 'COMFORT' | 'BUSINESS';
  base: number;          // базовая ставка, ₽
  perKm: number;         // ₽/км
  perMin: number;        // ₽/мин
  minPrice: number;      // минимум
  surge: number;         // коэффициент спроса (1.0 = нет)
};

export function calcPrice(from: LatLng, to: LatLng, cls: RideClass): {
  distanceM: number; durationMin: number; priceCents: number;
} {
  const distanceM = roadDistance(from, to);
  const durationMin = Math.max(1, Math.round(distanceM / (50_000 / 60))); // ~50 км/ч

  const raw =
    cls.base + (distanceM / 1000) * cls.perKm + durationMin * cls.perMin;
  // ЦЕНЫ В ЦЕНТАХ/КОПЕЙКАХ — ЦЕЛЫЕ ЧИСЛА (правило M9)
  const priceCents = Math.max(cls.minPrice * 100, Math.round(raw * cls.surge * 100));
  return { distanceM, durationMin, priceCents };
}
```

**Зачем чистая функция:**
- **Тестируется** без карты/моста/сети (M12: unit-тесты)
- **Одинаковая** на клиенте и сервере (може синить с бэкендом)
- **Предсказуемая** (нет случайности) → «цена 850 ₽» — детерминированно

### 7.8. Экран карты (MapScreen)

Собираем всё:

```tsx
export function MapScreen({ navigation }) {
  const { position } = useMyLocation(true);
  const [from, setFrom] = useState<GeoPoint | null>(null);
  const [to, setTo] = useState<GeoPoint | null>(null);
  const [classes, setClasses] = useState<ClassQuote[]>([]);

  // Когда есть from и to → запрашиваем классы+цены (бэкенд или локально M7.7)
  useEffect(() => {
    if (!from || !to) { setClasses([]); return; }
    const t = setTimeout(async () => {
      const quotes = CLASSES.map((c) => ({ ...c, ...calcPrice(from, to, c) }));
      setClasses(quotes);
    }, 200);
    return () => clearTimeout(t);
  }, [from, to]);

  return (
    <SafeAreaView style={flex1}>
      <MapView ...> {/* пользователь, from-пин, to-пин, полилиния */}
        <Marker coordinate={from} pinColor="green" title="Откуда" />
        <Marker coordinate={to} pinColor="red" title="Куда" draggable />
        {route && <Polyline coordinates={route} ... />}
      </MapView>

      {/* Поисковые поля (M3.6-паттерн) */}
      <View style={searchFields}>
        <Input value={from?.label} placeholder="Откуда" onFocus={() => navigation.navigate('AddressFrom')} />
        <Input value={to?.label} placeholder="Куда" onFocus={() => navigation.navigate('AddressTo')} />
      </View>

      {/* Нижняя карточка: классы (появляется, когда есть маршрут) */}
      {classes.length > 0 && (
        <BottomSheet open onClose={() => {}}>
          <FlatList data={classes} keyExtractor={(c) => c.id}
            renderItem={({ item }) => <ClassRow item={item} onPick={...} />}
            ListFooterComponent={
              <Button title={`Вызвать · ${formatPrice(selected.priceCents)}`} testID="call-cta"
                onPress={() => navigation.navigate('Order', { from, to, class: selected.id })} />
            }
          />
        </BottomSheet>
      )}
    </SafeAreaView>
  );
}
```

**UX-детали:**
- Поля «Откуда/Куда» — **по фокусу** открывают модалку `AddressPicker` (M7.4)
- Пока нет `to` — карта «за вами», поле «Куда» подсвечено
- Маршрут и классы — **после** обеих точек (дебаунс 200мс на перетаскивание)
- **Перетаскивание пина = 60fps** (M10: анимация через Reanimated, не setState на каждый кадр)

### 7.9. Слоты-заглушки для M8

```ts
// src/features/ride/stubs (появятся настоящими в M8)
export function useRideSocket(rideId: string) {
  // M8: WebSocket: позиция водителя, статусы
  return { driverPos: null, status: 'CREATED' };
}
```
В `MapScreen`/`OrderScreen` уже **имеются** слоты под живые данные M8 — чтобы в M8 просто «включить» реалтайм.

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 0. Dev build (4ч — разовые)
- [ ] `npx expo run:ios` → DevClient на симуляторе
- [ ] `npx expo run:android` → DevClient на эмуляторе (+ Google Maps key)
- [ ] `npx expo prebuild` после правок app.json
- [ ] Проверить: maps/location/keychain работают в dev build (а не в Expo Go)

### Шаг 1. Мок-бэкенд: гео
- [ ] `GET /geo/search?q=&limit=` (фикс-база 20–50 адресов вашего города)
- [ ] `POST /rides` (M8, но контракт сейчас): `{ from, to, class }` → `{ id, status: 'CREATED', priceCents }`

### Шаг 2. Permissions + моя позиция (4ч)
- [ ] `ensureLocationPermission` + экран-разрешение (rationale → системный → denied→settings)
- [ ] `useMyLocation` (M7.2) + маркер «Вы» (follow)
- [ ] Unit-тест `useMyLocation`: cleanup подписки (unmount → sub.remove())

### Шаг 3. AddressPicker (6ч)
- [ ] Карта + поиск (debounce) + suggestions + перетаскиваемый пин + reverse geocode
- [ ] `onMapLongPress` / `onMarkerDrag*` / `onRegionChangeComplete`
- [ ] Отдельные маршруты `AddressFrom` / `AddressTo` (M3) → `onPick` →回到 MapScreen
- [ ] **60fps** при перетаскивании (никаких тяжёлых setState на кадр — Reanimated)

### Шаг 4. Маршрут + цена + классы (6ч)
- [ ] Полилиния (бэкенд/кривая) + haversine + roadDistance
- [ ] `calcPrice` (M7.7) + **unit-тесты**: короткие/длинные/нулевые расстояния, surge, minPrice, целые копейки
- [ ] BottomSheet классы (Economy/Comfort/Business) + выбор + CTA «Вызвать»

### Шаг 5. Полный флоу (4ч)
- [ ] От → К → классы → цена → «Вызвать» → (M8) `Order`
- [ ] Черновик (M5): от/к/класс сохраняются (пережил перезапуск)
- [ ] На Both platforms: iOS MapKit + Android Google Maps — одинаково

### Шаг 6. Отчёт (2ч) — `docs/progress/M7.md`
- Скриншоты: permissions-фолды, выбор точки (поиск + drag), маршрут, классы
- Unit-тесты цены (результаты)
- Performance: 60fps при drag (Android Profiler — M10 подтвердит)
- Ошибки: 3+ (включая «серая карта Android» → API key)

---

## ✅ Практическое задание модуля

### Задача: «Карта: от → к → классы → цена → Вызвать»

**1. Dev build (4ч)** — оба устройства, prebuild-цикл, Google Maps key (Android)

**2. Гео (8ч)**
- [ ] Permissions (lazy, rationale, denied→settings) — обе платформы
- [ ] `useMyLocation` (follow, cleanup) + моя позиция
- [ ] **Unit-тесты**: cleanup подписки; haversine (известные пары точек)

**3. Выбор адреса (8ч)**
- [ ] AddressPicker: поиск (debounce) + suggestions + drag-пин + reverse geocode
- [ ] AddressFrom / AddressTo модалки → MapScreen
- [ ] **60fps** drag (Reanimated, без setState-per-frame)

**4. Маршрут и цена (6ч)**
- [ ] Полилиния + расстояние + `calcPrice`
- [ ] **Unit-тесты `calcPrice`**: ≥ 8 кейсов (короткое/длинное/zero/surge/min/целые)
- [ ] Классы в BottomSheet + выбор + CTA

**5. Флоу + черновик (4ч)**
- [ ] Полный цикл до «Вызвать» (на обеих платформах)
- [ ] Черновик (M5) сохраняется (от/к/класс)

**6. Отчёт (2ч)**

**Критерии:**
- [ ] Permissions: 3 состояния (undetermined/granted/denied) → корректный UX
- [ ] Точка задаётся и поиском, и перетаскиванием (60fps)
- [ ] Цена: целые копейки, детерминированно, покрыта тестами
- [ ] Маршрут: полилиния + классы + CTA
- [ ] iOS + Android — одинаково

---

## 📦 Итоговая структура после M7

```
src/features/
├── geo/
│   ├── permissions.ts      # ensure + settings
│   ├── location.ts         # useMyLocation
│   ├── AddressPicker.tsx   # карта+поиск+drag
│   ├── geocoding.ts        # reverse + search
│   └── geo.ts              # haversine, roadDistance (чистые)
├── map/
│   ├── MapScreen.tsx       # от→к→классы→CTA
│   ├── price.ts            # calcPrice (чистая)
│   └── ClassRow.tsx
└── ride/
    └── stubs.ts            # задел M8
src/store/
└── rideDraft.ts            # (M5 — теперь пишется из MapScreen)
```

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Expo Go для maps/location | `native module not found` | Dev build (M7.0) |
| Серая карта Android | Нет/неверный Google API key | Ключ + SHA-1 ограничение |
| Permissions при запуске | Мгновенный отказ (пользователь не понимает зачем) | Lazy + rationale |
| Denied → некуда | Пользователь застрял | Кнопка «Открыть настройки» |
| setState на каждый кадр drag | Lag, < 60fps | Reanimated / throttle |
| Утечка watchPosition | GPS в фоне, батарея | cleanup sub.remove() |
| Цена в float (850.5) | Ошибка округления | Целые копейки (M7.7) |
| Правка ios/android руками | Потеряется при prebuild | Только app.json + plugins |
| onRegionChange в цикле | Бесконечные reverse geocode | onRegionChangeComplete (дебаунс) |

---

## 💡 Советы на модуль

- **Dev build — новая нормальная среда** с M7. Не возвращайтесь в Expo Go.
- **Цена — чистая функция.** Это самый тестируемый код модуля. Целые копейки.
- **Permissions = UX.** Rationale + denied→settings — не «надоедливость», а уважение.
- **60fps при drag** — держите. M10 подтвердит цифрами.
- **Две платформы, две карты** (MapKit/Google). Тестируйте оба.

**[→ Модуль 8: Реалтайм, push, фон](../Module_08_Realtime_Push/content.md)**
