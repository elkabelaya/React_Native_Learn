# 🎓 Завершение курса: что дальше

**[← Назад к README](README.md)**

Поздравляем! Если вы прошли все 14 модулей и релизнули RideGo — вы на уровне **Middle Mobile Developer**. Этот документ — карта следующего роста.

---

## 📦 Ваш итоговый артефакт (проверьте перед собеседованиями)

### 1. GitHub-репозиторий `ridego`
```
✅ README: что это, скриншоты, демо-видео, как запустить
✅ Architecture: диаграмма feature-модулей, 5 ADR
✅ Badges: CI ( Actions), coverage, license
✅ Release: ссылки TestFlight + Play (внутренние треки)
✅ Docs: CONTRIBUTING (конвенции), CODE_OF_CONDUCT
```

### 2. Публичные артефакты для портфолио
- [ ] **Демо-видео 3 мин** (скринкаст: полный цикл поездки)
- [ ] **Performance-отчёт** (M10): до/после, с цифрами
- [ ] **Статья** в Dev.to/Medium/Хабр: «Как я строил RN-приложение: 5 trade-offs» (Expo, RTK vs Zustand, WS vs push, MMKV vs AsyncStorage, FlashList vs FlatList)
- [ ] **Два репозитория**: `ridego` (app) + `ridego-api` (backend) — показывает, что понимаете весь стек

### 3. Резюме
Добавьте блок:
```
Проект: RideGo (React Native, TypeScript) — сервис заказа поездок
• Полный цикл: геолокация → карта → выбор → реалтайм-трекинг (WebSocket)
  → оплата (Stripe, 3DS) → push → чек; iOS + Android
• Performance: TTI −40%, 60 fps, bundle −25% (профилирование, FlashList,
  работа с памятью)
• Quality: unit (Jest) + E2E (Maestro) тесты, CI/CD (GitHub Actions →
  TestFlight/Play), OTA-обновления, Sentry (crash-free 99.5%)
• Architecture: feature-модули, RTK + TanStack Query, MMKV, deep links
Роль: solo-разработка (моб + мок-API) от идеи до релиза
```

---

## 🧠 Что должно «быть в голове» (задайте себе)

| Вопрос | Ждём в ответе |
|--------|---------------|
| Что произойдёт, когда React отрисует элемент? | diff → JSI → Fabric (UI-поток) / old arch (bridge/JSON) |
| Почему Reanimated плавный, а Animated — нет? | worklets в UI-потоке vs JS-поток (глицерин) |
| Как устроен refresh-токен без гонок? | single-flight: промис-шаринг, 401-интерцептор, идемпотентность |
| Что обновится по OTA, а что — нет? | JS-да, натив/permissions/иконка — нет (стор-релиз) |
| Как устроен push на iOS vs Android? | APNs vs FCM, каналы, importance, terminated → deep link |
| Как оптимизировать список 1000 элементов? | FlashList, стабильные keys, мемо renderItem, lazy images |
| Как вы отлаживаете lag? | Profiler (re-renders) → Screenshot Time → CPU flame → memory snapshot |
| Что такое JSI? | Native bindings: JS-объект вызывает нативный объект без сериализации |
| Expo или bare CLI? | Когда что, config plugins, dev builds, прецедент |
| Как протестировать фичу? | Пирамида: unit (логики) → component (RNTL+msw) → E2E (Maestro) |

Если на 8+ вопросов можете ответить уверенно и с примерами из RideGo — вы **средний+.**

---

## 📈 Рост до Senior (6–18 месяцев)

Middle умеет делать фичи. Senior **принимает решения о системе**. Что качать дальше:

### 1. Архитектура (главный разделитель)
- **Monorepo** для мобильного (Turborepo/Nx: shared package между iOS/Android/Flutter/Web)
- **Модульная архитектура** глубже: feature-модули с контрактами, внутренний npm, versioning
- **Cross-platform код-шаринг** (React Native + React Web из одного кодабазы — логики, не UI)
- **Дизайн-система** как продукт: токены, библиотека компонентов, Figma↔код
- **Контроль зависимостей**: как уговаривать команду на новый стек (ADR + proof-of-concept)

### 2. Масштаб
- **Мультиплатформенная команда**: как строить RN-команду 5+ человек (разделение, ownership)
- **Release train**: как выпускать 2–4 релиза/нед (feature flags, feature freeze, hotfix-процессы)
- **Качество как система**: error budgets, SLO, on-call, incident response
- **Аналитика как движатель фич**: метрики → гипотезы → A/B → решение

### 3. Глубина
- **Натив**: писать свои модули, отлаживать нативные проблемы (memory, JNI/Obj-C)
- **Производительность как практика**: performance budget, CI-профилирование, регрессии
- **Безопасность**: secure storage, код-обфускация, Threat model, GDPR
- **DevTools**: написать свой инструмент (профилировщик, отладчик)

### 4. Soft skills Senior
- **Менторство** джуниоров (code review, pairing)
- **Тех. roadmap** отдела
- **Коммуникация с не-техниками** (бизнес-кейсы, ROI фич)
- **Представление на конференциях** (JSConf, React Native Summit, локальные митапы)

---

## 🗺️ Дорожная карта навыков (по годам)

| Период | Фокус | Метрика |
|--------|-------|---------|
| 0–3 мес | Закрепление: поддержка RideGo, баги, edge-cases | Уверенность в отладке |
| 3–6 мес | Одна глубокая тема (например, performance или натив) | Способен провести оптимизацию с нуля |
| 6–12 мес | Архитектура + масштаб: monorepo, дизайн-система | Лидируешь в PR-ревью команды |
| 12–24 мес | Системное мышление + влияние | Принимает решения, за которыми идут |

---

## 🚪 Куда идти работать

Позиции, которые вам открыты после курса:

| Позиция | Вероятность | Зарплата (ориентир, РФ/Remote, $) |
|---------|-------------|-----------------------------------|
| Middle RN Developer | ✅ высокая | 2.5–4.5k (Rusia) / 3–5k (Remote) |
| Frontend → RN (трансфер) | ✅ | как frontend + премия |
| Mobile Dev (Flutter + RN) | ⚠️ если выучите Dart | шире выбор |
| Senior RN (через 2 года) | 🎯 | 4–6.5k (Rusia) / 5–8k (Remote) |

**Что ищут работодатели (и как закрываете):**
- RN + TypeScript — ✅ (M1–14)
- Реалтайм / WebSocket — ✅ (M8)
- Платежи — ✅ (M9)
- Мат. уровень: отлаживание производительности — ✅ (M10)
- Тесты (incl. E2E) — ✅ (M12)
- CI/CD + сторы — ✅ (M13)
- Мониторинг + аналитика — ✅ (M14)
- Натив (Swift/Kotlin) — ⚠️ Optional (M11 advanced)

**Бонусы, которые сильно повышают ценность:**
- Flutter (второй фреймворк — широкий выбор вакансий)
- Нативный опыт (хотя бы чтение Swift/Kotlin)
- Go/Node бэкенд (полный stack)
- Опыт с реальными продуктами (выпуск, мониторинг, incident)

---

## 📚 Ресурсы для продолжения

### Книги
- «The Road to Less React» — мысли о состоянии
- «Designing Data-Intensive Applications» (Kleppmann) — для понимания бэкенда/реалтайма
- «React Native: Up and Running» (3rd) — углубление
- «Expo, React Native and TypeScript» (Bauerfeld)

### Документация (читать регулярно — она меняется)
- [reactnative.dev/blog](https://reactnative.dev/blog) — новые возможности
- [expo.dev/blog](https://expo.dev/blog)
- [reactnative.dev/architecture](https://reactnative.dev/architecture) — New Architecture
- Changelog: RN, Expo SDK, Reanimated, TanStack Query

### Конференции и митапы
- React Native Summit
- JSConf / Frontend Conf
- Локальные митапы React Native
- Expo + React Native (YouTube, подкаст)

### Подкасты / YouTube
- React Native Podcast
- Expo (официальный канал)
- Fireship (быстрые разборы)
- AC (Apparition) — RN deep dive

---

## 🏁 Итог

Вы прошли путь:
```
JavaScript → React Native → Mobile Dev → Middle
(набор фич)  (платформа)  (границы)   (система)
```

И главное: вы умеете **строить** продукт от идеи до сторов. Это то, что отличает мидла от джуна и что ищут в продуктовых командах.

**Дальше — практика, масштаб и влияние. Удачи!** 🚀

---

**[Наверх ↑](#-завершение-курса-что-дальше)**
