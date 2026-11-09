# 🚀 Начало работы: настройка окружения

**[← Назад к README](README.md)** | **Далее: [Модуль 1](Module_01_Environment_Platform/content.md) →**

В этом документе — пошаговая настройка окружения. Всё, что здесь установлено, используется на всём курсе.

---

## 1. Системные требования

### 🍎 macOS (рекомендуется — полный курс)

| Компонент | Минимум | Идеально |
|-----------|---------|----------|
| ОС | macOS 13 (Ventura) | macOS 14+ (Sonoma) |
| CPU | M1 | M1 / M2 |
| RAM | 8 ГБ | 16 ГБ |
| Диск | 30 ГБ свободно | 50 ГБ |

### 🪟 Windows / Linux

| Компонент | Требование |
|-----------|-----------|
| ОС | Windows 10/11 (WSL2) / Ubuntu 22.04+ |
| OSGi | Android SDK, adb |
| Ограничение | ❌ Без iOS-симулятора. iOS-части курса — теория + реальный iPhone (необходим) |

> 💡 **Совет:** Для курса критичен **реальный Android** (средний бюджетник: Snapdragon 7-й серии, 6–8 ГБ RAM). Симуляторы врут по производительности — половина модуля 10 невозможна без живого устройства.

---

## 2. Установка инструментов (пошагово)

### Шаг 1: Node.js (18 LTS+)

```bash
# macOS — через Homebrew
brew install nvm
nvm install 18
nvm use 18

# Проверка
node --version   # v18.x
npm --version    # 9.x+
```

Windows: установите [Node 18 LTS](https://nodejs.org/) или fnm.  
Linux: `nvm` (кратко) или официальный установщик.

**Почему Node 18, а не latest?** Expo SDK требует конкретные мажорные версии Node (см. `engines` в package.json проекта). `nvm`/`fnm` позволяют переключать версии без боли.

### Шаг 2: Git

```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Шаг 3: Expo CLI

```bash
npm install -g expo-cli
# или без глобальной установки:
npx expo --help
```

### Шаг 4: Xcode (только macOS)

1. Скачайте **Xcode 15+** из Mac App Store (≈ 15 ГБ)
2. При первом запуске: *Agree to license* → *Open Developer Tool*
3. Установите компоненты: **Settings → Platforms → iOS 17** (скачается ≈ 8 ГБ)
4. Проверьте: **Xcode → Settings → Command Line Tools** — путь к Xcode

```bash
# Проверка
xcodebuild -version
```

### Шаг 5: Android Studio

1. Скачайте **Android Studio** (Hedgehog+)
2. При запуске: install SDK
3. **SDK Manager → SDK**: Android 14 (API 34) — Platform, Platform-Tools, Build-Tools
4. **SDK Manager → Virtual Devices**: создайте эмулятор:
   - Pixel 6 — Android 14
   - Pixel 5 (API 31) — «средний» телефон
   - При возможности загрузите образ с **x86_64 + HW acceleration** (в macOS: Apple Silicon image)
5. **Settings → Appearance & Behavior → System Settings → Android SDK → Paths**: убедитесь, что `ANDROID_HOME`/`ANDROID_SDK_ROOT` заданы

```bash
# Проверка (обработайте путь к Android SDK — для macOS обычно:
# ~/Library/Android/sdk; для Linux: ~/Android/Sdk)
adb --version
```

### Шаг 6: VS Code + расширения

| Расширение | Зачем |
|-----------|-------|
| **Expo Tools** | Автодополнение Expo, dev-server |
| **ESLint + Prettier** | Линт и формат в реальном времени |
| **ES7+ React snippets** | Быстрые шаблоны компонентов (rce, fun) |
| **Native DevTools** (React Native) | DevTools, profiler, inspect element |
| **GitLens** | Бlait-история коммитов |
| **Error Lens** | Ошибки линтера прямо в коде |

`settings.json`:
```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": { "source.fixAll.eslint": "explicit" },
  "emmet.includeLanguages": { "typescriptreact": "javascriptreact" }
}
```

### Шаг 7: (macOS) Homebrew + CocoaPods

```bash
brew install watchman   # ускорит Metro
brew install cocoapods  # нужен для некоторых RN-библиотек
```

---

## 3. Создание первого проекта RideGo

```bash
# 1. Создаём проект (blank-typescript template)
npx create-expo-app@latest ridego --template blank-typescript

# 2. Входим, смотрим структуру
cd ridego
ls -la
# App.tsx, index.ts, package.json, app.json, tsconfig.json, babel.config.js, metro.config.js, .gitignore

# 3. Поднимаем dev-сервер
npx expo start
```

В терминале появится **QR-код**. Отсканируйте его **Expo Go** (App Store / Play) — и приложение запустится на вашем телефоне по Wi-Fi.

**Два способа запуска:**

| Способ | Когда использовать | Плюсы | Минусы |
|--------|-------------------|-------|--------|
| **Expo Go** (скан QR) | Первые модули (1–6) | Быстро, без установки | Нет нативных модулей (maps, notifications, keychain — часть!) |
| **Dev build** (`eas build --profile development`) | Модуль 7+ | Полный контроль, все модули | Медленнее (сборка) |

> ⚠️ **Ключевой момент курса:** React Native Maps, expo-notifications, react-native-keychain — это **нативные модули**. В Expo Go их нет. Поэтому к модулю 7 мы построим **dev build** с нужными нативными зависимостями. До этого Expo Go — отличный инструмент.

---

## 4. Структура Expo-проекта (разбираем)

```
ridego/
├── App.tsx                  # Точка входа (root-компонент)
├── index.ts                 # registerRootComponent(App)
├── app.json                 # Метаданные: имя, иконка, splash, permissions
├── package.json             # Зависимости и скрипты
├── tsconfig.json            # TS-конфиг (strict включён в шаблоне)
├── babel.config.js          # Пресеты babel (reanimated и др.)
├── metro.config.js          # Конфиг сборщика
└── assets/                  # Иконки, шрифты, изображения
```

### Что важно в `app.json`:

```json
{
  "expo": {
    "name": "RideGo",
    "slug": "ridego",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "scheme": "ridego",                    // ← URI scheme для deep links
    "assetBundlePatterns": ["**/*"],
    "ios": {
      "supportsTablet": false,
      "bundleIdentifier": "com.yourname.ridego"
    },
    "android": {
      "package": "com.yourname.ridego",
      "versionCode": 1
    },
    "plugins": []                          // сюда будут попасть config plugins (maps, notifications)
  }
}
```

---

## 5. Учимся читать логи (важно!)

**50% багов RN живут в логах, а не в коде.** Откройте три окна:

1. **Терминал** (`npx expo start`) — ошибки JS,Metro
2. **React Native DevTools** (расширение VS Code) — React-профилирование
3. **Лог платформы**:
   - **iOS**: Xcode → *Window → Devices and Simulators* → выберите device → *Open Console*
   - **Android**: `adb logcat | grep -iE "ReactNativeJS|System.out|AndroidRuntime"`

**Быстрая диагностика:**
- Белая/чёрная зависшая страница → смотрите терминал (Metro) и Console (нативные логи)
- Ошибка на одной платформе → **сначала** сравните Console iOS vs `adb logcat`
- Crash при запуске → обычно natивный модуль (см. log про JSI)

---

## 6. Типовые проблемы при настройке

| Проблема | Решение |
|----------|---------|
| `node-gyp` ошибки при `npm install` | Убедитесь: Node 18 LTS (не 21), `brew install node` или nvm |
| Expo Go не видит QR по Wi-Fi | Телефон и компьютер — в одной сети; попробуйте USB (`npx expo start --tunnel`) |
| Metro: `Port 8081 is in use` | `npx expo start --port 8082` или убейте процесс: `lsof -i :8081` |
| `cocoapods` не установлен (macOS) | `brew install cocoapods` |
| Android Studio: `SDK location not found` | Задайте `ANDROID_HOME` в `local.properties`: `sdk.dir=/Users/you/Library/Android/sdk` |
| Эмулятор очень медленный | macOS: используйте Apple Silicon image; включите HW acceleration; RAM эмулятора — 4 ГБ |
| `Hermes` / New Arch ошибка | Проект Expo 50+ уже на New Architecture; убедитесь, что библиотека поддерживает RN 0.74+ |

---

## 7. Инициализация репозитория

```bash
cd ridego

# 1. Git-репозиторий (шаблон уже даёт .gitignore)
git init
git add .
git commit -m "chore: init ridego project (expo blank-typescript)"

# 2. GitHub
gh repo create ridego --public --source=. --push
# (или вручную: gh auth login → new repo → push)

# 3. Первостепенные скрипты в package.json:
#    "lint": "eslint . --ext .ts,.tsx"
#    "typecheck": "tsc --noEmit"
#    "start": "expo start"
```

**Conventional Commits** (будем использовать весь курс):

| Тип | Пример |
|-----|--------|
| `feat` | `feat: add driver search screen with pagination` |
| `fix` | `fix: refresh race on 401 — single-flight` |
| `chore` | `chore: add eslint + prettier` |
| `refactor` | `refactor: extract DriverCard into ui/` |
| `test` | `test: add state machine unit tests` |
| `perf` | `perf: FlashList for driver list` |

Каждый модуль = feature-ветка + 1 PR: `git checkout -b feat/module-1-onboarding` → работа → PR.

---

## ✅ Чек-лист готовности к модулю 1

- [ ] Node 18 LTS (`node --version`)
- [ ] Expo CLI (`npx expo --help`)
- [ ] Xcode + iOS 17 SDK (macOS)
- [ ] Android Studio + SDK 34 + эмулятор Pixel 6
- [ ] VS Code + расширения (Expo Tools, ESLint, Prettier, Native DevTools)
- [ ] Реальный Android рядом (в идеале)
- [ ] GitHub-репозиторий `ridego` создан
- [ ] Понимаете, где посмотреть логи (терминал + Console)

**[Далее: Модуль 1 →](Module_01_Environment_Platform/content.md)**
