# 🚗 RideGo — проект курса

**[← Назад к README](../README.md)**

**RideGo** — сервис заказа поездок (аналог Uber/Max). В этом файле — описание продукта и **архитектурная эволюция**: что добавляется в модуле.

---

## Продукт

### Пользователь
Пользователь RideGo:
1. Открывает приложение → видит карту со своей позицией
2. Указывает **точку посадки** (поиск или перетаскивание пина)
3. Указывает **пункт назначения** → видит маршрут и варианты (классы авто)
4. Выбирает класс → нажимает **«Вызвать»**
5. Отслеживает **прибытие водителя** в реальном времени (маркер едет, ETA)
6. Поездка идёт → **оплата картой** (Stripe)
7. Получает **чек** и ставит **оценку**
8. В **истории** видит все поездки, в **избранном** — любимых водителей

### Сущности

```
User        { id, phone, name, avatar, createdAt }
Address     { id, userId, label, raw, lat, lng, geoHash }
Driver      { id, name, photo, rating, car { model, color, plate }, class, distanceM, etaMin }
RideClass   { ECONOMY | COMFORT | BUSINESS }
Ride        { id, userId, from, to, class, status, priceCents, driverId?, createdAt, completedAt? }
RideStatus  CREATED → ACCEPTED → ARRIVED → STARTED → COMPLETED | CANCELLED
Payment     { id, rideId, status, provider: 'stripe', last4, amountCents }
Rating      { rideId, stars, tags[], comment }
Notification { pushToken, userId, platform }
```

---

## 📦 Архитектурная эволюция (что и когда появляется)

### После Модуля 1 — каркас
```
src/
├── app/            # App.tsx: providers, root-навигация (заглушка)
├── ui/             # (пусто, появится в M2)
├── features/       # (появится в M14 — пока flat)
└── theme/          # токены цвета
```
Splash, иконка, экран-заглушка, запуск на обеих платформах.

### После Модуля 2 — дизайн-система
```
src/ui/
├── Button/         # variants: primary/secondary/danger, loading, disabled
├── Card/
├── Badge/
├── Input/          # плавающий лейбл, ошибка, password
├── Avatar/
├── Spinner/
├── Skeleton/
├── BottomSheet/
└── index.ts        # barrel
src/features/onboarding/   # 3 страницы + свайп
src/theme/tokens.ts        # colors, typography, spacing, radii
```

### После Модуля 3 — навигация
```
src/navigation/
├── RootNavigator.tsx    # Auth stack | App stack (по auth)
├── AuthStack.tsx        # Register, Login, Otp
├── MainTabs.tsx         # Map, Orders, Profile
├── MapStack.tsx         # Map, AddressFrom, AddressTo, ClassChoice, Order
└── types.ts             # RootParamList, typed navigation
src/deep-linking/        # URI scheme ridego:// + universal links
```

### После Модуля 4 — данные
```
src/api/
├── client.ts        # axios, interceptors, error model
├── drivers.ts       # getDrivers({q, page, class})
├── rides.ts         # getRides, getRide
└── types.ts         # DTO + маппинг в доменные типы
src/features/drivers/  # экран поиска: FlatList, пагинация, debounce
src/features/orders/   # история: SectionList, pull-to-refresh
src/hooks/             # useDebouncedValue, usePagination
```

### После Модуля 5 — состояние
```
src/store/
├── index.ts           # RTK store + MMKV persister
├── favorites/         # slice: optimistic toggle
├── rideDraft/         # черновик: from/to/class
├── addresses/         # CRUD сохранённых
└── selectors.ts       # мемоизированные
```

### После Модуля 6 — auth
```
src/features/auth/
├── AuthProvider.tsx   # Keychain tokens, refresh, login/logout
├── refresh.ts         # single-flight 401 → 1 refresh
├── forms/             # RHF + zod: phone, otp, profile
└── biometrics/        # expo-local-authentication
src/api/auth.ts        # /register /login /refresh /me
```

### После Модуля 7 — карта и гео
```
src/features/geo/
├── permissions.tsx    # rationale, denied → settings
├── AddressPicker.tsx  # карта + поиск + перетаскиваемый пин
├── geocoding.ts       # reverse geocode
└── price.ts           # расчёт по дистанции
src/features/map/      # MapScreen: классы, CTA
```

### После Модуля 8 — реалтайм
```
src/features/ride/
├── machine.ts         # state machine: CREATED→…→COMPLETED
├── socket.ts          # WS client: heartbeat, reconnect w/ backoff
├── OrderScreen.tsx    # живой маркер, ETA, отмена
└── notifications.ts   # push + in-app баннеры
src/services/          # background tracking
```

### После Модуля 9 — платежи
```
src/features/payments/
├── stripe.ts          # PaymentSheet, 3DS
├── PayScreen.tsx
├── ReceiptScreen.tsx
└── RatingScreen.tsx   # звёзды + теги
```

### После Модуля 10 — performance
(то же самое, но оптимизированное: FlashList, стабильные keys, lazy-экраны, worklet-анимации, memory cleanup)

### После Модуля 11 — натив
```
src/native/
├── haptics.ts         # паттерные вибрации
├── BackHandler.tsx    # Android back
└── BatteryModule/     # свой TurboModule (advanced)
```

### После Модуля 12 — тесты
```
src/**/__tests__/      # unit: machine, price, refresh, formatters
src/features/**/__tests__/*.test.tsx   # component (msw)
.e2e/                  # Maestro-флоу
```

### После Модуля 13 — релиз
```
app.json               # иконки, adaptive icon, i18n
i18n/ru.json, i18n/en.json
.github/workflows/     # CI: PR → tests, main → beta, tag → prod+OTA
eas.json               # profiles: development/preview/production
```

### После Модуля 14 — продакшен (итоговая структура)
```
src/
├── app/
├── theme/
├── shared/                 # api client, hooks, utils, i18n, telemetry
├── features/
│   ├── onboarding/
│   ├── auth/
│   ├── map/        (geo, address pickers, price)
│   ├── drivers/
│   ├── ride/       (machine, socket, order, bg)
│   ├── payments/
│   ├── orders/     (history, receipt)
│   └── profile/
└── navigation/
```

**Правило зависимостей** (контролируется ESLint `import/no-restricted-paths`):
- `features/*` → может импортировать `shared/*`, `theme/*`, `ui/*`
- `features/*` → **не может** импортировать другие `features/*`
- `shared/*` → не знает о `features/*`
- `navigation/*` → единственный, кто знает все фичи (собирает граф)

---

## 🎯 Обязательные фичи (definition of done)

1. ✅ Карта + выбор «откуда/куда» + классы с ценами
2. ✅ Поиск водителей (пагинация, поиск, фильтры, рейтинг)
3. ✅ Auth: OTP, biometrics, Keychain, refresh
4. ✅ Полный цикл поездки: вызов → трекинг → оплата → чек → оценка
5. ✅ Push + in-app + deep link из push + фоновый трекинг
6. ✅ Stripe: success / 3DS / decline, idempotency
7. ✅ История + избранные + сохранённые адреса
8. ✅ Тёмная тема + i18n (RU/EN) + офлайн-каталог
9. ✅ Haptics + платформенный UX
10. ✅ (Advanced) собственный TurboModule

## 📏 Нон-функциональные требования

| Метрика | Бюджет |
|---------|--------|
| TTI (first interactive) | < 2.5s на реальном среднем Android |
| Frame drops | < 1% на горячих экранах (список, карта) |
| 60 fps | на перетаскивании пина, трекинге, bottom sheet |
| Crash-free users | > 99.5% (Sentry) |
| Memory | ≤ 2MB growth за 20 поездок в сессии |
| Bundle (JS) | −25% от baseline (конец M10) |
| Тесты | ≥ 70% бизнес-логики, 3 E2E зелёные |
| Lint / Type | 0 ошибок (CI gate) |

---

## 🧪 Мок-бэкенд (в параллель с приложением)

Отдельный репозиторий `ridego-api` (Node + Express + SQLite + Socket.IO):

| Метод | Путь | Назначение |
|-------|------|-----------|
| POST | `/auth/register` | телефон → код |
| POST | `/auth/otp` | код → JWT pair |
| POST | `/auth/login` | пароль → JWT pair |
| POST | `/auth/refresh` | refresh → новая пара |
| GET | `/me` | профиль |
| PUT | `/me` | обновление профиля |
| GET | `/drivers?q=&page=&class=` | поиск (пагинация) |
| POST | `/rides` | создать поездку |
| GET | `/rides?cursor=` | история (пагинация) |
| GET | `/rides/:id` | детали |
| POST | `/rides/:id/cancel` | отмена |
| POST | `/rides/:id/favorite` | избранное (toggle) |
| POST | `/addresses` / DELETE `/addresses/:id` | CRUD адресов |
| POST | `/payments/intent` | stripe intent |
| POST | `/rides/:id/rating` | оценка |
| WS | `/socket?token=` → `order:{id}` | позиции водителя, статусы |

**Генератор GPS:** WebSocket-сервер каждые 2 сек шлёт новую позицию водителя, движущегося по полилинии (от старта к точке посадки, затем по маршруту) — имитация реального трекинга.

> Это **ваш** бэкенд для курса — пишется в модулях 4–8. Не обязательно красивый: главное — честные контракты + генератор GPS.

---

## 🛣️ Дорожная карта релизов

| Релиз | После модуля | Состав |
|-------|--------------|--------|
| v0.1.0 | 3 | UI + навигация (демо, без данных) |
| v0.2.0 | 6 | + данные, состояние, auth |
| v0.3.0 | 9 | + карта, трекинг, push, платежи (пред-прода) |
| v0.9.0 | 13 | perf + натив + тесты + CI + i18n |
| **v1.0.0** | 14 | Sentry, аналитика, A/B, **релиз в сторах** |

---

**[К модулю 1 →](../Module_01_Environment_Platform/content.md)**
