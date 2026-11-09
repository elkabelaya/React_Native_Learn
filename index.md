# 📱 React Native Mobile Development — навигация по курсу

**[← Назад к README](README.md)**

---

## 📚 Карта курса

```
┌─────────────────────────────────────────────────────────────────┐
│                    REACT NATIVE: JUNIOR → MIDDLE               │
└─────────────────────────────────────────────────────────────────┘

  УРОВЕНЬ 1: ОСНОВЫ                    УРОВЕНЬ 2: ДАННЫЕ
  ┌───────────────────┐                ┌───────────────────┐
  │ M1  Environment   │                │ M4  Data/API      │
  │     Platform      ├──────────────▶│     + Lists       │
  └───────────────────┘                └───────────────────┘
           │                                  │
           ▼                                  ▼
  ┌───────────────────┐                ┌───────────────────┐
  │ M2  UI/Styling    │                │ M5  State/        │
  │     + Animations  ├──────────────▶│     Storage       │
  └───────────────────┘                └───────────────────┘
           │                                  │
           ▼                                  ▼
  ┌───────────────────┐                ┌───────────────────┐
  │ M3  Navigation    │                │ M6  Auth/Security │
  │     + Deep Links  │                │     + Biometrics  │
  └───────────────────┘                └───────────────────┘
                                          │
                                          ▼
  УРОВЕНЬ 5: ПРОД          ┌───────────────────┐          УРОВЕНЬ 4: MIDDLE
  ┌───────────────────┐    │ M12 Testing       │◀─────────┌───────────────────┐
  │ M14 Production    │◀───│     (Jest/msw/    │──────────│ M11 Native/       │◀── M10 Performance
  │     Operations    │    │     Maestro)      │          │     Platform      │
  └───────────────────┘    └───────────────────┘          └───────────────────┘
  Sentry, аналитика,             │                                ▲
  A/B, архитектура               │                                │
                                 ▼                                │
  ┌───────────────────┐    ┌───────────────────┐          ┌───────────────────┐
  │ M13 Release/CI/CD │◀───│ M9  Payments      │──────────│ M8  Realtime/     │
  │     (EAS/Stores)  │    │     (Stripe)      │          │     Push/Background│
  └───────────────────┘    └───────────────────┘          └───────────────────┘
                                                                         │
                                                                         │   УРОВЕНЬ 3: ПЛАТФОРМА
                                                                         ▼
                                                                  ┌───────────────────┐
                                                                  │ M7  Geo/Maps      │
                                                                  └───────────────────┘
```

---

## 📖 Файлы курса

| Файл | Описание |
|------|----------|
| [README.md](README.md) | Главная страница курса: программа, стек, график |
| [GETTING_STARTED.md](GETTING_STARTED.md) | Настройка окружения: Node, Expo, Xcode, Android Studio |
| [COURSE_INFO.md](COURSE_INFO.md) | Информация: длительность, требования, критерии Middle |
| [CHEATSHEET.md](CHEATSHEET.md) | Шпаргалка: команды Expo, паттерны RN, частые конфиги |
| [COURSE_COMPLETION.md](COURSE_COMPLETION.md) | Что после курса: рост до Senior, карьера |
| [PROJECT/README.md](PROJECT/README.md) | RideGo: описание проекта и его архитектурная эволюция |

---

## 🗂️ Модули (по порядку прохождения)

### 🟢 Уровень 1: Основы

- [ ] [Модуль 1: Окружение и устройство платформы](Module_01_Environment_Platform/content.md) — Expo, Metro, Hermes, JSI, Fabric; первый запуск на двух платформах
- [ ] [Модуль 2: UI, вёрстка, темизация, анимации](Module_02_UI_Styling/content.md) — базовые компоненты, Flexbox, токены, тёмная тема, Reanimated
- [ ] [Модуль 3: Навигация](Module_03_Navigation/content.md) — React Navigation, вкладки, deep links, auth-guard

### 🔵 Уровень 2: Данные и состояние

- [ ] [Модуль 4: Данные: API, списки, кэширование](Module_04_Data_API_Lists/content.md) — axios, TanStack Query, FlatList/FlashList, мок-бэкенд
- [ ] [Модуль 5: Состояние и локальное хранилище](Module_05_State_Management/content.md) — Redux Toolkit, MMKV, optimistic updates, ADR #1
- [ ] [Модуль 6: Авторизация и безопасность](Module_06_Auth_Security/content.md) — OTP, Keychain, refresh-flow, biometrics, формы (RHF + zod)

### 🟠 Уровень 3: Платформенная сложность

- [ ] [Модуль 7: География и карты](Module_07_Geo_Maps/content.md) — permissions, expo-location, react-native-maps, расчёт цены
- [ ] [Модуль 8: Реалтайм: WebSocket, push, фон](Module_08_Realtime_Push/content.md) — машина состояний, reconnect, push, background service
- [ ] [Модуль 9: Платежи, чеки, оценки](Module_09_Payments/content.md) — Stripe, 3DS, decline, idempotency

### 🟣 Уровень 4: Средний уровень

- [ ] [Модуль 10: Производительность](Module_10_Performance/content.md) — профилирование, re-renders, списки, старт, память
- [ ] [Модуль 11: Нативный уровень и платформы](Module_11_Native_Platform/content.md) — TurboModules, permissions, haptics, свой модуль
- [ ] [Модуль 12: Тестирование](Module_12_Testing/content.md) — Jest, RNTL, msw, Maestro
- [ ] [Модуль 13: Релиз, CI/CD, сторы](Module_13_Release_CI_CD/content.md) — code signing, EAS, TestFlight/Play, OTA, i18n

### 🔴 Уровень 5: Продакшен

- [ ] [Модуль 14: Мониторинг, аналитика, архитектура](Module_14_Production_Operations/content.md) — Sentry, PostHog, A/B, feature-модули, финальный релиз

---

## ✅ Чек-лист прогресса (общий)

### После модулей 1–3 (Basic)
- [ ] RideGo запускается на iOS и Android (dev build)
- [ ] Можете объяснить своими словами: JSI, Hermes, Fabric, Metro
- [ ] Дизайн-система: 10+ компонентов, тёмная тема, 60 fps анимации
- [ ] Навигация: вкладки, stack, deep links, auth-guard (stub)

### После модулей 4–6 (Data & State)
- [ ] Мок-бэкенд: drivers, trips — с пагинацией, поиском, фильтром
- [ ] Списки: 500+ элементов без джанков, все UI-состояния
- [ ] Redux Toolkit: favorites, rideDraft, addresses — персистентно (MMKV)
- [ ] Полный auth-цикл: OTP → Keychain → refresh → биометрия → logout
- [ ] Гонка 401: 5 запросов → 1 refresh (покрыто unit-тестами)

### После модулей 7–9 (Platform)
- [ ] Флоу: откуда → куда → класс → цена → вызов
- [ ] Живой трекинг: WS, плавный маркер, ETA, reconnect
- [ ] Push на обеих платформах, deep link из push (terminated app)
- [ ] Фоновый трекинг (Android foreground service / iOS background)
- [ ] Оплата: success / 3DS / decline, чек, оценка

### После модулей 10–13 (Middle)
- [ ] Performance-отчёт: до/после, с цифрами и скринами
- [ ] 60 fps на «горячих» экранах, TTI < 2.5s, bundle −25%
- [ ] Платформенные фичи: haptics, back button, battery, TurboModule
- [ ] Тесты: ≥ 70% бизнес-логики, 10+ component, 3 E2E-флоу
- [ ] CI: PR → тесты → beta (TestFlight/Play) → tag → production + OTA
- [ ] i18n: RU/EN, валюты, даты

### После модуля 14 (Production)
- [ ] Sentry: crash-free > 99.5%, runbook первого инцидента
- [ ] Воронка заказа в PostHog, A/B «новый чек»
- [ ] Архитектура: feature-модули, 5 ADR, ESLint-правила импортов
- [ ] Финальный релиз v1.0.0, demo-видео, README
- [ ] Презентация: 30 минут без шпаргалки

---

## 💡 Как пользоваться курсом

1. **Всегда по порядку.** Модуль 8 использует навигацию (3), состояние (5), auth (6) и карту (7). Пропуск создаст «дыры», которые больно заделывать потом.
2. **Каждый модуль = 1 PR.** Заведите репозиторий GitHub в модуле 1. Финальный PR каждого модуля — это строка в вашем резюме.
3. **Два устройства рядом.** При каждом «странном» поведении сразу смотрите: как ведёт себя iOS vs Android. Разница платформ — 50% багов RN.
4. **Performance-ноутбук.** С первого «медленного» экрана записывайте: что делали, что изменили, метрика до/после. В модуле 10 этот ноутбук станет отчётом.
5. **Заметки.** 1 заметка на модуль (что решил, почему, что было сложно) → в конце курса это ваша статья «Как я строил RN-приложение: 5 trade-offs».

## ⚠️ Чего делать НЕ стоит

- ❌ Переиспользовать чужой код из интернета без понимания — вы придёте на собеседование и не сможете объяснить свой код
- ❌ Писать на JS «потому что быстрее» — курс на TypeScript strict; привычка any = дорога обратно на Junior
- ❌ Тестировать производительность на симуляторе — он лжёт. Нужен реальный (в идеале бюджетный) Android
- ❌ Делать «всё сразу» — модуль за модулем, каждый закрывается зелёным PR

## 🚨 Если застряли

- **> 2 часа на одной ошибке** → StackOverflow, issue-трекер библиотеки, discord Expo
- **Не понимаете концепцию** → перечитайте доки, затем сформулируйте: «я ожидал X, получил Y, вот репро»
- **Баг на одной платформе** → сравните логи Console/Xcode ДО изменения кода

**[Наверх ↑](#-react-native-mobile-development-—-навигация-по-курсу)**
