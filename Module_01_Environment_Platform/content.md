# 📘 Модуль 1: Окружение и устройство платформы

**[← Назад к индексу](../index.md)** | **Далее: [Модуль 2](../Module_02_UI_Styling/content.md) →**

Добро пожаловать на курс! В этом модуле мы заложим фундамент: поймём, **как устроен React Native изнутри** (это главный разделитель junior/middle), настроим окружение и запустим первый **RideGo** на iOS и Android.

**Цели модуля:**
1. Понять архитектуру React Native: Metro, Hermes, Bridge, JSI, Fabric, New Architecture
2. Осознанно выбрать между Expo и bare React Native CLI
3. Настроить окружение: Node, Expo, Xcode, Android Studio
4. Создать проект RideGo, подключить дизайн-токены, запустить на **обоих платформах**

**Время выполнения:** ~15 часов.

---

## 1. Теория: что такое React Native и как он устроен

**React Native** — фреймворк (Meta) для создания нативных мобильных приложений на JavaScript/TypeScript. Ваш JS-код выполняется в **JVM JS-движка (Hermes)**, а UI рендерится в **настоящие нативные контролы** (UIView на iOS, View/TextView на Android) — не в WebView.

### Двухпоточная модель (самое важное!)

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│      JS Thread (Hermes)     │     │       UI Thread (main)      │
│                             │     │                             │
│  React: state, props,       │ ──▶ │  Нативный UI: layout,       │
│  диффы, бизнес-логика       │     │  рендер пикселей            │
│  useEffect, хуки            │     │  гесты, анимации            │
└─────────────────────────────┘     └─────────────────────────────┘
            │    передача сообщений  │
            └────── (bridge) ───────┘
```

- **JS thread** — выполняется ваш React-код. Если JS «завис» (тяжёлый расчёт, `setTimeout` с тяжёлой задачей) — **анимации и жесты тоже зависнут** (в старой архитектуре).
- **UI thread** — рисует пиксели, обрабатывает жесты.
- **Bridge (старая архитектура)** — JSON-сериализация каждой команды между потоками. Быстро, но с накладным налогом и потерей при микротасках.
- **JSI (New Architecture)** — нативные объекты, видимые из JS **без сериализации**. Функция вызывается напрямую, как нативный метод. Скорость ×N.

### Что входит в New Architecture

| Компонент | Что делает | Практический смысл |
|-----------|-----------|--------------------|
| **Hermes** | Быстрый JS-движок (precompil'd bytecode) | Запуск приложения быстрее, меньше памяти |
| **JSI** | Без-сериализационные нативные мосты | Меньше latency в JS↔натив |
| **Fabric** | Новый UI-рендерер (диффы в UI-потоке) | Быстрый, параллельный layout; не блокирует JS |
| **TurboModules** | Ленивая загрузка нативных модулей | Не загружаем 200 модулей при старте |

> 💡 **Почему это важно для middle:** 80% вопросов на RN-собеседованиях — про эту архитектуру. И 80% перформанс-багов — следствие непонимания двухпотоковости.

### Expo vs React Native CLI (bare)

| Критерий | Expo (Managed + dev builds) | Bare RN CLI |
|----------|------------------------------|-------------|
| Создание проекта | `create-expo-app` | `npx react-native init` |
| Нативные модули | **Config Plugins** (app.json) — без ручного код-редактинга | Ручной `ios/` + `android/` |
| OTA-обновления | **EAS Update** из коробки | React Native Community CLI / CodePush (убит) |
| CI/CD | **EAS Build** (облако) | Собственный (Fastlane/Xcode/Gradle) |
| Контроль нативного кода | Ограниченный (prebuild порождает) | Полный |
| Скорость старта | ⚡ Очень быстрая | Медленнее |

**Вывод 2024+:** Expo — де-факто стандарт. Он покрывает 95% сценариев (включая карты, push, платёжные модули) через **config plugins**. Bare CLI — когда нужен полный контроль над `ios/`/`android/` (крупные команды, экзотные модули). **Курс — на Expo**, но в модуле 11 вы заглянете «под капот» нативных папок.

### Сборщик Metro

```
Ваш код (.tsx) → Babel (трансформ в JS) → Metro (бандл, HMR, watch) → Hermes (bytecode) → устройство
```

- **HMR (hot reload)** — при сохранении файла Metro пересылает модуль, React перерисовывает без потери состояния (частичного).
- **Fast Refresh** — сохранение компонента: перерисовка с сохранением state (в отличие от full reload).
- **Кэш** — при «странных» проблемах: `npx expo start -c` (очистить).

### Типы приложений RN

| Тип | Что это | Где используется |
|-----|---------|------------------|
| **Expo Go** | Приложение-обёртка со стора | Прототипы, Expo SDK без нативных модулей |
| **Dev build** | Своя сборка с Expo Dev Client | Курс: модуль 7+ (maps, notifications, keychain) |
| **Production build** | Сборка для сторов | Релиз (модуль 13) |
| **EAS Update (OTA)** | Обновление JS-бандла без стора | Хотфиксы |

> ⚠️ **Expo Go не поддерживает нативные модули** (maps, keychain, notifications — часть). До модуля 7 работаем в Expo Go; в модуле 7 строим dev build.

---

## 2. Теория: структура Expo-проекта (углублённо)

```
ridego/
├── App.tsx                  # Корневой компонент (providers, root)
├── index.ts                 # registerRootComponent(App)
├── app.json                 # Метаданные + config plugins (и icon, scheme, permissions)
├── package.json             # Зависимости, скрипты
├── tsconfig.json            # TS strict, алиасы @/*
├── babel.config.js          # babel-preset-expo + reanimated plugin
├── metro.config.js          # Конфиг сборщика
├── assets/                  # icon.png, splash.png, шрифты
└── src/                     # ВАШ КОД
    ├── theme/
    │   ├── tokens.ts        # цвета, типографика, spacing, радиусы
    │   └── index.ts
    └── (дальше — по мере роста курса)
```

### app.json — что мы уже настраиваем

```json
{
  "expo": {
    "name": "RideGo",
    "slug": "ridego",
    "version": "0.1.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "scheme": "ridego",                        // ← URI scheme (deep links, M3)
    "userInterfaceStyle": "automatic",         // ← светлая/тёмная (M2)
    "assetBundlePatterns": ["**/*"],
    "ios": {
      "bundleIdentifier": "com.yourname.ridego",
      "supportsTablet": false
    },
    "android": {
      "package": "com.yourname.ridego",
      "versionCode": 1
    }
  }
}
```

> ⚠️ **bundleIdentifier/package** — фиксируйте с первого дня. Изменить потом = новый аккаунт приложения (потеря рейтинга, push-токенов). Для курса используйте `com.ВашеИмя.ridego`.

---

## 3. Практика: первый запуск RideGo

### Шаг 1: Создание проекта

```bash
npx create-expo-app@latest ridego --template blank-typescript
cd ridego
git init && git add . && git commit -m "chore: init ridego"
gh repo create ridego --public --source=. --push
```

### Шаг 2: Папка src

```bash
mkdir -p src/theme src/app src/ui src/features
```

### Шаг 3: Токены дизайна (src/theme/tokens.ts)

```ts
// src/theme/tokens.ts
export const colors = {
  // Бренд
  brand: {
    primary: '#4F46E5',   // индиго — кнопки
    onPrimary: '#FFFFFF',
    accent: '#F59E0B',    // янтарный — CTA «вызвать»
  },
  // Поверхности (светлая)
  surface: {
    bg: '#F9FAFB',
    card: '#FFFFFF',
    border: '#E5E7EB',
  },
  // Текст
  text: {
    primary: '#111827',
    secondary: '#6B7280',
    inverse: '#FFFFFF',
  },
  // Состояния
  state: {
    error: '#EF4444',
    success: '#10B981',
    warning: '#F59E0B',
  },
} as const;

export const spacing = {
  xs: 4, sm: 8, md: 16, lg: 24, xl: 32, xxl: 48,
} as const;

export const radii = {
  sm: 6, md: 12, lg: 20, pill: 999,
} as const;

export const typography = {
  title: { fontSize: 28, fontWeight: '700' as const, letterSpacing: -0.5 },
  subtitle: { fontSize: 17, fontWeight: '600' as const },
  body: { fontSize: 15, fontWeight: '400' as const, lineHeight: 21 },
  caption: { fontSize: 12, fontWeight: '500' as const },
};
```

> 💡 `as const` — TypeScript: литеральные типы. `fontWeight: '700' as const` — иначе TS потеряет строковый литерал.

### Шаг 4: Экран-заглушка (src/app/HomeScreen.tsx)

```tsx
import { View, Text, StyleSheet, Pressable } from 'react-native';
import { colors, spacing, radii, typography } from '../theme/tokens';

export function HomeScreen() {
  return (
    <View style={styles.container}>
      <View style={styles.logo}>
        <Text style={styles.logoText}>🚗</Text>
      </View>
      <Text style={styles.title}>RideGo</Text>
      <Text style={styles.subtitle}>Ваша поездка начинается здесь</Text>

      <Pressable style={styles.button} onPress={() => console.log('pressed')}>
        <Text style={styles.buttonText}>Начать</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: colors.surface.bg,
    alignItems: 'center',
    justifyContent: 'center',
    padding: spacing.lg,
  },
  logo: {
    width: 96,
    height: 96,
    borderRadius: radii.lg,
    backgroundColor: colors.brand.primary,
    alignItems: 'center',
    justifyContent: 'center',
    marginBottom: spacing.lg,
  },
  logoText: { fontSize: 48 },
  title: { ...typography.title, color: colors.text.primary, marginBottom: spacing.xs },
  subtitle: { ...typography.body, color: colors.text.secondary, marginBottom: spacing.xxl },
  button: {
    backgroundColor: colors.brand.accent,
    borderRadius: radii.pill,
    paddingVertical: spacing.md,
    paddingHorizontal: spacing.xxl,
  },
  buttonText: { color: colors.brand.onPrimary, fontWeight: '600', fontSize: 16 },
});
```

### Шаг 5: App.tsx — точка входа

```tsx
import { StatusBar } from 'expo-status-bar';
import { HomeScreen } from './src/app/HomeScreen';

export default function App() {
  return (
    <>
      <StatusBar style="auto" />
      <HomeScreen />
    </>
  );
}
```

### Шаг 6: Запуск на обеих платформах

```bash
# Dev-сервер
npx expo start
```

- **iOS (симулятор):** клавиша `i` в терминале Expo
- **iOS (устройство):** отсканируйте QR Expo Go
- **Android (эмулятор):** клавиша `a`
- **Android (устройство):** отсканируйте QR Expo Go

**Проверка:** на обоих устройствах — экран с логотипом, заголовком, кнопкой. Тап по кнопке → `console.log('pressed')` в терминале.

### Шаг 7: Конфигурируем TypeScript строго (tsconfig.json)

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

Теперь `import { colors } from '@/theme/tokens'` — работает. Добавьте скрипты в `package.json`:

```json
"scripts": {
  "start": "expo start",
  "android": "expo run:android",
  "ios": "expo run:ios",
  "lint": "eslint . --ext .ts,.tsx",
  "typecheck": "tsc --noEmit"
}
```

---

## 4. Практика: первая «платформенная» забота

### Платформенный выбор цвета иконок

```tsx
import { Platform } from 'react-native';

// src/app/PlatformBadge.tsx
export function PlatformBadge() {
  const label = Platform.OS === 'ios' ? 'iOS' : 'Android';
  return <Text>Вы на {label}</Text>;
}
```

Добавьте на HomeScreen. Убедитесь: на iOS-симуляторе «Вы на iOS», на Android-эмуляторе — «Вы на Android». Это ваш первый урок: **один код, две платформы — проверяйте обе**.

### Safe Area (notch)

```bash
npx expo install react-native-safe-area-context
```

```tsx
// App.tsx
import { SafeAreaProvider } from 'react-native-safe-area-context';

export default function App() {
  return (
    <SafeAreaProvider>
      <StatusBar style="auto" />
      <HomeScreen />
    </SafeAreaProvider>
  );
}
```

> `SafeAreaProvider` — будет использоваться во всех модулях (навигация, bottom sheet).

---

## 5. Разбираем нативный уровень (для понимания)

Выполните:

```bash
npx expo prebuild    # порождает ios/ + android/ из app.json
```

Загляните (не правьте!):

- `android/app/src/main/AndroidManifest.xml` — permissions, activity, meta-data
- `ios/Info.plist` — CFBundleIdentifier, URL types (scheme), background modes

**Вывод:** `app.json` + **config plugins** = эти файлы генерируются. Вы не правите их руками (кроме `expo prebuild --clean` при смене плагинов).

> ⚠️ **Никогда не коммитьте** `ios/` и `android/` в Expo-проект (они в `.gitignore`). Это не bare-проект. Если увидите чужой Expo-репо с закоммиченными нативками — это красный флаг.

---

## 📝 Домашнее задание (Модуль 1)

### Обязательные задачи

1. **Иконка + сплэш**
   - Сгенерируйте иконку приложения (512×512) и splash (2048×2048)
   - Сгенерируйте **adaptive icon** (Android: foreground + background)
   - Подключите в `app.json` (поля `icon`, `splash`, `android.adaptiveIcon`)
   - После изменения: `npx expo prebuild --clean` → `npx expo run:ios` / `run:android`

2. **Dev build (критично!)**
   ```bash
   npm i -g eas-cli
   eas login
   eas build:configure    # создаст eas.json с profile development
   eas build -p android --profile development --local
   eas build -p ios --profile development --local
   ```
   - Установите dev build на **реальный Android**
   - Запустите: `npx expo start --dev-client`
   - ✅ Приложение открывает **в вашей иконке** (не Expo Go)

3. **Экран-заглушка — расширьте**
   - Добавьте `PlatformBadge` (как в уроке)
   - Добавьте версию приложения из `expo-constants`:
     ```bash
     npx expo install expo-constants
     ```
     ```tsx
     import Constants from 'expo-constants';
     <Text>Version {Constants.expoConfig?.version} ({Constants.expoConfig?.runtimeVersion})</Text>
     ```

4. **TypeScript strict + алиасы**
   - Убедитесь: `strict: true`, `@/*` алиас работает
   - Переберите код: все импорты через `@/`
   - `npm run typecheck` — 0 ошибок

5. **Линтер**
   ```bash
   npm i -D eslint eslint-config-expo eslint-plugin-react-hooks @trivago/prettier-plugin-sort-imports
   ```
   Создайте `.eslintrc.js`:
   ```js
   module.exports = {
     extends: ['expo', 'prettier'],
     plugins: ['react-hooks'],
     rules: {
       'react-hooks/rules-of-hooks': 'error',
       'react-hooks/exhaustive-deps': 'warn',
       '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
     },
   };
   ```
   `npm run lint` — 0 ошибок.

### Дополнительные задачи (по желанию)

6. **Настройте Prettier** (`.prettierrc`):
   ```json
   { "printWidth": 100, "singleQuote": true, "trailingComma": "all" }
   ```
7. **Изучите `eas.json`** — что такое profiles (development/preview/production), когда каждый используется.
8. **Нативное чтение:** найдите в `android/app/src/main/...` объявление MainActivity и `MainActivity.kt` (после prebuild). Что делает `super.onCreate()`?

### Критерии сдачи

- [ ] RideGo открывается на iOS-симуляторе и Android-эмуляторе **в вашей иконке** (dev build)
- [ ] Реальный Android: dev build установлен, QR → приложение
- [ ] `npm run typecheck` — 0 ошибок, `npm run lint` — 0 ошибок
- [ ] Импорт через `@/` работает
- [ ] `git log` — аккуратные conventional commits
- [ ] GitHub-репо синхронизирован

### Как проверить

1. `npx expo start --dev-client` → отсканируйте на реальном Android и iOS-симуляторе
2. Проверьте иконку на экране (не Expo Go-ионку)
3. `npm run typecheck && npm run lint` — зелёные
4. Откройте Xcode → Devices → Console: нет красных ошибок

---

## 💡 Полезные команды (модуль 1)

```bash
# Expo
npx expo start                    # dev server
npx expo start --dev-client       # для dev build
npx expo start -c                 # очистить кэш Metro
npx expo doctor                   # диагностика

# Нативные сборки
npx expo prebuild                 # ios/ + android/ из app.json
npx expo prebuild --clean         # пересоздать (после config plugins)
npx expo run:ios                  # собрать + запустить в Xcode
npx expo run:android              # собрать + запустить

# EAS
eas login
eas build:configure               # eas.json
eas build -p android --profile development
eas build -p ios --profile development
eas build:configure               # уже есть
eas update                        # OTA (модуль 13)

# Диагностика
adb logcat | grep -iE "ReactNativeJS|System.out"   # Android-логи
# iOS: Xcode → Window → Devices and Simulators → Console
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| Expo Go: `This native module cannot be found` | Пытаетесь использовать нативный модуль (maps/keychain) в Expo Go | Постройте dev build (`eas build --local`) |
| `Port 8081 is in use` | Другой Metro запущен | `lsof -i :8081` → убить, или `--port 8082` |
| Metro: `unable to resolve module` | Опечатка в импорте | Проверьте путь, алиасы tsconfig + metro (Expo сам читает tsconfig paths) |
| `Hermes: could not allocate` (эмулятор) | Мало памяти у эмулятора | RAM эмулятора → 4 ГБ; закройте лишнее |
| `xcodebuild: no Xcode selected` | Не выбрана CommandLine Tools | `sudo xcode-select --switch /Applications/Xcode.app` |
| `adb: device offline` | Эмулятор ещё стартует | Подождите boot; `adb devices` — должен быть `device` |
| Приложение белое, в консоли `JSI ...` | Несовместимость версий RN и модуля | `npx expo install` (не npm i) для всех RN-пакетов |
| Fast refresh «ломает» состояние | Нормальное поведение | Не пугайтесь; это не баг |
| `Cannot find module 'react-native/Libraries/...'` | Несоответствие verсий RN и библиотеки | `npx expo install react-native` (Expo сам сверит) |

---

## 💡 Советы на модуль

- **Не правьте `ios/` и `android/` вручную** — только через `app.json` + config plugins. Это фундаментальный принцип Expo.
- **Два окна логов** — в постоянной работе: терминал Metro + Console (Xcode или adb logcat). Привыкайте.
- **Dev build — ваш друг.** С него (модуля 7) и дальше весь курс. Настройте `eas build --local` — это 20 минут разового.
- **Git с первого дня.** Каждый шаг — коммит. Конвенция: `chore:`, `feat:`, `fix:`, `test:`, `refactor:`, `perf:`.

**[→ Модуль 2: UI, вёрстка, темизация, анимации](../Module_02_UI_Styling/content.md)**
