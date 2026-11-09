# 📋 Информация о курсе

**[← Назад к README](README.md)**

---

## Общее

| Параметр | Значение |
|----------|----------|
| Название | React Native Mobile Development |
| Формат | Project-Based Learning: разрабатываем приложение **RideGo** |
| Длительность | 14 модулей, ~18–19 недель (15–20 ч/нед.), ≈ 300 часов практики |
| Стартовый уровень | Уверенный JavaScript/TypeScript, базовый React (желательно), git |
| Итоговый уровень | **Middle Mobile Developer (React Native)** |
| Платформы | iOS + Android (Windows/Linux — Android + iOS по теории) |

---

## 🧠 Что вы будете знать после курса

### Архитектура RN (умение объяснять, а не цитировать)

- Двухпоточная модель: JS thread ↔ UI thread, что происходит при каждом рендере
- Old vs New Architecture: Bridge (JSON) → JSI (native bindings), Hermes (bytecode, precompilation), Fabric (UI-диффы в UI-потоке), TurboModules (ленивые, асинхронные)
- Когда и почему производительность уходит из JS (worklets, Reanimated)

### Стек (осознанный выбор, а не «из туториала»)

| Вопрос | Ваш ответ после курса |
|--------|----------------------|
| Expo или bare RN CLI? | Когда что, dev builds, config plugins, прецеденты миграции |
| Redux Toolkit или Zustand? | Масштаб, экосистема, middleware, персистентность (ADR #1) |
| TanStack Query или Redux для API? | Server-state ≠ client-state, кэш, invalidation |
| FlatList или FlashList? | Виртуализация, критические сценарии (1000+ элементов) |
| WebSocket или long-polling/push? | Каналы, reconnect, что допустимо в фоне |
| AsyncStorage или MMKV? | Скорость, синхронный API, лимиты |
| Animated или Reanimated? | UI thread vs JS thread, worklets |

### Мобильные компетенции (то, чего нет у веб-разработчиков)

- **Permissions** на обеих платформах (rationale, denied → settings)
- **Фон**: App lifecycle, background modes (iOS), foreground services (Android)
- **Push**: APNs/FCM, токен → бэкенд, iOS channels vs Android channels, terminated → deep link
- **Платформенный UX**: safe areas, notch, back button, swipe-back, haptics, keyboard
- **Сторы**: code signing, provisioning, review, staged rollout, OTA-границы
- **Производительность**: метрики (TTI, frame drops, memory), инструменты, бюджеты

### Soft skills мидла

- Оценка большой фичи → спринты (декомпозиция «трекинга в фоне»)
- Code review по чек-листу (что именно чекать в RN-PR)
- ADR-документирование решений
- Коммуникация: бэкенд (контракты), дизайн (токены), PM (метрики)
- Incident-менеджмент по runbook (crash в проде)

---

## 📊 Критерии готовности к уровню Middle (чек-лист)

### Технические

- [ ] Strict TypeScript без `any` в бизнес-коде
- [ ] Объясняете JSI/Hermes/Fabric на уровне «коллеге с собеседования»
- [ ] Реалтайм-фича с reconnect, фоном, push, deep link — от и до
- [ ] Профилирование: React DevTools, Screenshot Time, Android Profiler, flamegraph
- [ ] Оптимизация: re-renders, списки, старт (TTI −40%), bundle (−25%)
- [ ] Тесты: unit (state machine, forms, price calc), component (msw), 3 E2E-флоу
- [ ] Code signing + CI/CD + OTA — пайплайн «в один тап»
- [ ] Знаете границы: что обновится по OTA, а что — только стор-релизом

### Проектные

- [ ] Приложение: 10 обязательных фич (см. README)
- [ ] 60 fps на горячих экранах (средний Android)
- [ ] TTI < 2.5s, crash-free > 99.5%
- [ ] Feature-модульная архитектура + 5 ADR + cross-import lint
- [ ] Релиз: TestFlight + Play internal, staged rollout
- [ ] README + demo-видео 3 мин + performance-отчёт

---

## ⏱️ Длительность модулей

| Модуль | Недель | Часов | Уровень |
|--------|--------|-------|---------|
| 1. Окружение и платформа | 1 | 15 | 🟢 |
| 2. UI, стили, анимации | 1 | 15 | 🟢 |
| 3. Навигация | 1 | 15 | 🟢 |
| 4. Данные, API, списки | 2 | 30 | 🔵 |
| 5. Состояние и хранилище | 1.5 | 22 | 🔵 |
| 6. Auth и безопасность | 1.5 | 22 | 🔵 |
| 7. Гео и карты | 2 | 30 | 🟠 |
| 8. Реалтайм, push, фон | 2 | 30 | 🟠 |
| 9. Платежи | 1.5 | 22 | 🟠 |
| 10. Производительность | 1.5 | 22 | 🟣 |
| 11. Нативный уровень | 1.5 | 22 | 🟣 |
| 12. Тестирование | 1.5 | 22 | 🟣 |
| 13. Релиз, CI/CD | 1.5 | 22 | 🟣 |
| 14. Продакшен-операции | 1 | 15 | 🔴 |
| **Итого** | **~19** | **~282 + проект** | |

---

## 🎓 Требования для старта

### Обязательные
- JavaScript: ES2020+ (promises, async/await, spread, destructuring)
- TypeScript: базовый (interfaces, union, generics-основы)
- Git: ветки, PR, merge, resolution конфликтов
- React: компоненты, props, state, useEffect (хотя бы базово)

### Желательные
- Опыт с любым фреймворком (React, Vue, Angular)
- Базовое понимание HTTP (REST, статусы, headers)

### Не требуется
- ❌ Swift/Kotlin (натив — Optional в модуле 11)
- ❌ Опыт с React Native
- ❌ Опыта с мобильными приложениями

---

## 💰 Инструменты курса — бесплатно

| Инструмент | Бесплатный лимит | Хватит на курс? |
|-----------|------------------|----------------|
| Expo / EAS Build | 30 build-минут/мес | ✅ (≈ 10–15 сборок) |
| Expo EAS Update | Бесконечно | ✅ |
| TestFlight | 100 бета-тестеров, 90 дней | ✅ |
| Google Play internal track | Бесплатно (нужна $25 разовая регистрация) | ✅ |
| Sentry | 5k errors/мес, 10k sessions | ✅ |
| PostHog | 1 млн событий/мес | ✅ |
| Stripe test mode | Бесконечно (тестовые карты) | ✅ |
| Maestro | Открытый исходный код | ✅ |
| GitHub Actions | 2000 минут/мес (public) | ✅ |

Единственная оплата: **$25 разовые** за аккаунт Google Play Developer (для внутреннего трека). Apple Developer Program ($99/год) — опционально: TestFlight работает и на бесплатном аккаунте (7 дней, 3 приложения).

---

## 📞 Где спросить, если застряли

1. **Документация** (первый источник, всегда актуальна)
   - [reactnative.dev](https://reactnative.dev)
   - [docs.expo.dev](https://docs.expo.dev)
   - [reactnavigation.org](https://reactnavigation.org)
   - [tanstack.com/query](https://tanstack.com/query/latest)
2. **Issue-трекеры** библиотек (ищите свой баг — часто с фиксом/обходным)
3. **StackOverflow** (теги `react-native`, `expo`)
4. **Discord Expo / React Native** — живой комьюнити

**Как задавать вопрос** (формула): «Я сделал X, ожидал Y, получил Z. Вот код (минимальный репро) и лог. Версии: RN x, Expo y, OS z. Что уже пробовал: …»

---

**[Далее: Модуль 1 →](Module_01_Environment_Platform/content.md)** | **[Шпаргалка](CHEATSHEET.md)** | **[Что дальше?](COURSE_COMPLETION.md)**
