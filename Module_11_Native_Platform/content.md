# Модуль 11: Натив и платформы

**[← Назад к индексу](../index.md)** | **[Модуль 10: Performance](../Module_10_Performance/content.md)** | **Далее: [Модуль 12: Тестирование](../Module_12_Testing/content.md) →**

---

## 🎯 Цель модуля

- **Expo config plugins**: настроить натив (без eject)
- **Turbo Modules + JSI**: как пишут нативные модули (посмотреть и написать простой)
- **Платформенные различия** iOS/Android: клавиатура, статус-бар, жесты, safe area
- **Сенсорные/системные API**: биометрия, камера (avatar), share, haptics
- **ADR #2**: config plugins vs prebuild + edit

**Длительность: ~22 часа.**

---

## 📚 Теория

### 11.1. Нативный слой: как устроено

```
JS (React Native)
   │  bridge (Hermes) / JSI (Hermes ↔ C++, синхронно, без JSON-сериализации)
   ▼
Nатив: iOS (ObjC/Swift) + Android (Kotlin/Java)
   │  Turbo Modules: нативные модули, автогенерация, ленивая инициализация
   ▼
Системные API (Camera, Biometrics, Share, Haptics, Location…)
```

**Turbo Modules** — новый механизм (замена старому Bridge): ленивая загрузка, **JSI** для синхронных вызовов, автотипизация. Большинство RN-библиотек (M1–M10) — Turbo Modules под капотом.

### 11.2. Config plugins: настраивать натив без eject

Expo **dev build** + **config plugins** = «натив по-взрослому» без `npx react-native eject`. Плагины **модифицируют** `ios/` и `android/` при `prebuild`:

```bash
# Пример: добавить поддержку Google Maps, permissions, background
npx expo install -- expo-... (библиотека)
# config plugin в app.json:
```

```json
{
  "expo": {
    "plugins": [
      "expo-location",
      "expo-camera",
      "expo-local-authentication",
      [
        "expo-notifications",
        { "icon": "./assets/push.png", "color": "#111" }
      ],
      "./plugins/withCustomPermissions.js"   // свой плагин (M11.3)
    ],
    "ios": {
      "infoPlist": { "NSCameraUsageDescription": "Для фото-аватара" }
    }
  }
}
```

**Как работает:** `npx expo prebuild` → плагины применяются → генерируется `ios/`+`android/` → `npx expo run:ios`. **Никогда не правим натив руками** (потеряется при prebuild) — всё через плагины.

**Свой config plugin** (M11.3): JS-функция, которая мутирует нативные файлы:
```js
// plugins/withMyModule.js
const { withAndroidManifest, withInfoPlist } = require('@expo/config-plugins');
module.exports = (config) => {
  config = withAndroidManifest(config, (c) => {
    c.modResults.manifest.mainActivity?.main?.push? // пример: добавить permission
  });
  return config;
};
```
(Для курса: один простой плагин — добавить permission/ключ.)

### 11.3. Turbo Modules: написать свой (кратко)

**Когда:** библиотека не подходит, нужна **синхронная** нативная функция (например, кастомный hash/хеш, быстрый формат). Для асинхронных (камера, сеть) — существующие модули.

**Скелет** (Android, Kotlin):
```kotlin
class FastFormatModule(spec: ReactModuleSpec) {
  @ReactMethod(isBlockingSynchronousMethod = true)
  fun formatCents(cents: Int, currency: String): String {
    // синхронно (JSI) — без bridge-queue
    return String.format("%s.%s %s", ...);
  }
}
```
**iOS (Objective-C/Swift)** — аналог (`RCT_EXPORT_METHOD` / JSI).

> ⚠️ Для курса: **напишите ОДИН** простой синхронный модуль (например, нативный `formatMoney` или `hash`), подключите config plugin, вызовите из RN. Цель — **увидеть**, как это устроено, а не строить инфраструктуру. (Полноценные модули — это уже senior.)

### 11.4. Платформенные различия: таблица

| Аспект | iOS | Android | Как处理的 |
|--------|-----|---------|-----------|
| Статус-бар | `Info.plist` + `StatusBar` | `android:statusBarColor` | `expo-status-bar` (M2) |
| Safe area | Очейный/двухой notch | Разные | `react-native-safe-area-context` (M1) |
| Клавиатура | `KeyboardAvoidingView` (padding) | `adjustResize` | Разные (M6) |
| Жесты назад | Нет (кнопка) | Есть (жест) | Не конфликтовать (M8) |
| Биометрия | Face ID | Fingerprint/StrongBox | `expo-local-authentication` (M6) |
| Push | APNs | FCM | `expo-notifications` (M8) |
| Camera | `NSCameraUsageDescription` | `CAMERA` permission | `expo-camera` (M11) |
| Haptics | `UIImpactFeedbackGenerator` | `Vibrator` | `expo-haptics` (M11) |

**Принцип:** пишите **кросс-платформенно**, но знайте **точки различия**. Проверяйте **оба** на dev build.

### 11.5. Экран/жесты: конфликт «назад»

**Проблема:** Android-жест «назад» (свух справа) + ваш `BackHandler` (M8: «точно отменить?»). Кто победит?

```ts
// src/shared/backHandler.ts
import { BackHandler, Platform } from 'react-native';

export function useBackConfirm(enabled: boolean, onConfirm: () => void) {
  useEffect(() => {
    if (Platform.OS !== 'android' || !enabled) return;
    const sub = BackHandler.addEventListener('hardwareBackPress', () => {
      onConfirm();   // показать «точно?» и вернуть true (поглотили)
      return true;
    });
    return () => sub.remove();
  }, [enabled, onConfirm]);
}
```
(ИOS: жест-назад не «отменяет заказ», только навигацию — проверяем, что `popToTop` не ломает заказ.)

### 11.6. Хэптические/системные отклики (UX-кит)

```bash
npx expo install expo-haptics expo-camera expo-sharing expo-av
```

| API | Где | Пример |
|-----|-----|--------|
| **Haptics** | `expo-haptics` | Тап по звезде (M9): `impactAsync(ImpactFeedbackStyle.Medium)` |
| **Camera** | `expo-camera` | Аватар (M11.7): `requestCameraPermissionsAsync` → снять → upload |
| **Share** | `expo-sharing` | Поделиться чеком (M9.7): PDF → share sheet |
| **AV** | `expo-av` | Звук «водитель прибыл» (M8) — опц. |

### 11.7. Практика: аватар через камеру (M6-профиль → M11)

```tsx
// src/features/profile/camera.ts
import * as Camera from 'expo-camera';

export async function takeAvatar(): Promise<string | null> {
  const { status } = await Camera.getPermissionsAsync();   // M11.4: rationale
  if (status !== 'granted') {
    const req = await Camera.requestPermissionsAsync();
    if (req.status !== 'granted') return null;   // → «в настройках»
  }
  const cam = new Camera.Camera();
  const uri = await cam.takePictureAsync({ quality: 0.7, base64: false });
  // upload (M4/бэкенд) → URL → PUT /me (M6)
  return uri?.uri ?? null;
}
```
(Проверка permissions: `NSCameraUsageDescription` (iOS) + `CAMERA` (Android) — через config plugin.)

### 11.8. ADR #2: config plugins vs prebuild+edit

| Критерий | Config plugins (Expo) | Prebuild + edit (bare) |
|----------|----------------------|------------------------|
| Нативные правки | Через плагины (воспроизводимо) | Вручную в ios/android |
| Обновления Expo | Лёгкие (prebuild перегенерит) | Больные (merge ручной) |
| Сложные модули | Ограничено (нужен плагин) | Свободно |
| Воспроизводимость | Высокая (app.json = источник) | Низкая (ручные правки теряются) |
| Команда/наём | Стандарт (Expo) | Редкость |

**ADR #2 (RideGo):** **config plugins** (app.json — источник правды натива). Причины: воспроизводимость, лёгкие апгрейды Expo, команда привыкает. Bare (eject) — только если появится модуль без плагина (риск: потеря при prebuild). **Задокументировать.**

### 11.9. Кастомные элементы/нативные поверхности

- **`RCTView` / `View`** — RN. Для **нативного виджета** (например, нативный счётчик, видео) — **Turbo Module** возвращает нативный view (M11.3).
- **WebView** (`react-native-webview`) — для вёрстки, которую не хочется писать на RN (опц., для «помощи»).

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Config plugins (6ч)
- [ ] `app.json`: все плагины (location, camera, biometrics, notifications, maps) в одном месте
- [ ] **Свой плагин**: `plugins/withRideGo.js` (добавляет 1 permission / 1 info.plist key)
- [ ] `npx expo prebuild` → проверить, что нативные файлы сгенерированы **с** вашими настройками
- [ ] **ADR #2** (M11.8) → `docs/adr/0002-native-approach.md`

### Шаг 2. Turbo Module (6ч)
- [ ] Написать **один** простой синхронный модуль (например, нативный `formatMoney` или `hash`) — Android (Kotlin) + iOS
- [ ] Config plugin для подключения
- [ ] Вызвать из RN + unit-тест
- [ ] (Не строить инфраструктуру — только «увидеть»)

### Шаг 3. Платформенные различия (4ч)
- [ ] Проверить **обе** платформы: статус-бар, safe area, клавиатура, жест-назад
- [ ] `useBackConfirm` (M11.5) — Android не «отменяет» заказ
- [ ] Тест: iOS (face-id) + Android (fingerprint) биометрия (M6)

### Шаг 4. Системные API (4ч)
- [ ] **Haptics**: звёзды (M9), тап по пину (M7)
- [ ] **Camera**: аватар (M11.7) → upload → PUT /me (M6)
- [ ] **Share**: чек (M9.7) → PDF → share sheet
- [ ] (Опц.) **AV**: звук «водитель прибыл»

### Шаг 5. Отчёт (2ч) — `docs/progress/M11.md`
- ADR #2 (config plugins)
- Скриншоты: камера-аватар, share, биометрия (обе платформы)
- Turbo module: код + вызов + тест
- Ошибки: 3+ (включая «натив потерялся при prebuild»)

---

## ✅ Практическое задание модуля

### Задача: «Натив по-взрослому + платформенные различия»

**1. Config plugins (6ч)** — app.json (все плагины) + свой плагин + ADR #2

**2. Turbo Module (6ч)** — один синхронный модуль (Kotlin + iOS) + подключение + тест

**3. Платформы (4ч)** — обе платформы: статус/safe/клавиатура/жест-назад; биометрия

**4. Системные API (4ч)** — haptics, camera-аватар, share-чек, (AV)

**5. Отчёт (2ч)**

**Критерии:**
- [ ] Натив настраивается **только** через app.json + плагины (не руками)
- [ ] Свой config plugin работает
- [ ] **ADR #2** задокументирован
- [ ] Один Turbo Module (Kotlin+iOS) вызывается из RN + протестирован
- [ ] Платформенные различия (5+) обработаны, проверены на **обеих**
- [ ] Camera-аватар, share-чек, haptics, биометрия — работают

---

## 📦 Итог

Вы теперь понимаете **весь стек**: JS → JSI/Turbo → натив → системные API. Знаете, когда использовать config plugins, а когда — писать модуль. И не боитесь платформенных различий.

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Правка ios/android руками | Потеряется при prebuild | Только плагины (M11.2) |
| Нет rationale на permission | Отказ | Свой экран + системный (M7.1) |
| Жест-назад отменяет заказ | Потеря UX | BackHandler + confirm (M11.5) |
| Клавиатура: iOS/Android по-разному | Обрезка | Разные (padding/adjustResize) |
| Синхронный модуль через bridge | Лag | JSI (M11.3) |
| Асинхронное через JSI | Не так | Bridge (обычный метод) |
| Expo Go для нативных модулей | Не работает | Dev build (M7) |

---

## 💡 Советы на модуль

- **app.json = источник правды** натива. Плагины — инструмент.
- **Turbo Module** — чтобы «увидеть», не строить. Один простой — достаточно.
- **Платформы — 90% одинаково, 10% иначе.** Знайте 10% (таблица M11.4).
- **Проверяйте обе** — на dev build.

**[→ Модуль 12: Тестирование](../Module_12_Testing/content.md)**
