# Модуль 5: Состояние и локальное хранилище

**[← Назад к индексу](../index.md)** | **[Модуль 4: Данные, API](../Module_04_Data_API_Lists/content.md)** | **Далее: [Модуль 6: Авторизация и безопасность](../Module_06_Auth_Security/content.md) →**

---

## 🎯 Цель модуля

- **Разделение** client-state (RTK) / server-state (Query, M4) — **не дублировать**
- **Redux Toolkit** + **Slice Colocation** (M5.1)
- **Persistence** (MMKV) — асинхронно, без блокировки UI
- **Optimistic updates** (M4.9) — мгновенно + откат + **sync-очередь** (M5.7)
- **Offline-first** (M5.8): очередь, reconnect (M8), drain, идемпотентность
- **ADR #1** (состояние)

**Длительность: ~22 часа. «Где живёт состояние — это вопрос, не факт».**

---

## 📚 Теория

### 5.1. Три типа состояния (не смешивать)

| Тип | Пример | Владелец | Где |
|-----|--------|----------|-----|
| **Server-state** | список водителей, поездка | **API** (источник правды) | **TanStack Query** (M4) |
| **Client-state** | избранное, черновик заказа, адрес | **React** | **Redux Toolkit** |
| **UI-state** (эфемерное) | открыт ли modal, активный экран | **Компонент** | `useState` |

> 💡 **Главное правило middle:** **не держите server-state в Redux** (M4). Redux — только то, что **ваше** (избранное, черновик). Server-данные — Query. Смешение = «кэш руками».

```
┌─────────────────────────────────────────────┐
│  Redux Store (RTK)                          │
│  ├─ favorites (per driver)                  │  ← client-state
│  ├─ rideDraft (от/к/класс)                  │  ← черновик
│  └─ addresses (локальный кэш)               │  ← локальная копия
│  (persist → MMKV, M5.3)                     │
├─────────────────────────────────────────────┤
│  TanStack Query Cache (M4)                  │  ← server-state
│  ├─ ['drivers', ...]                        │
│  └─ ['rides', ...]                          │
├─────────────────────────────────────────────┤
│  Component useState                         │  ← UI-state (эфемерное)
│  └─ isModalOpen, activeTab                  │
└─────────────────────────────────────────────┘
```

### 5.2. Redux Toolkit

```bash
npx expo install @reduxjs/toolkit react-redux
```

**Slice Colocation** (M5.1): **slice живёт рядом с фичей**, а не в «одном god-store»:

```
src/
├─ features/
│  ├─ favorites/slice.ts       # фича «избранное»
│  ├─ rideDraft/slice.ts       # черновик заказа
│  └─ addresses/slice.ts       # адреса (локальная копия)
├─ app/store.ts               # комбинирует slices
└─ store.ts                   # re-export
```

```ts
// src/features/favorites/slice.ts
export const favoritesSlice = createSlice({
  name: 'favorites',
  initialState: [] as string[],
  reducers: {
    toggle: (state, a: PayloadAction<string>) =>
      state.includes(a.payload) ? state.filter(i => i !== a.payload) : [...state, a.payload],
    clear: () => [],
  },
});
export const { toggleFavorite, clearFavorites } = favoritesSlice.actions;
export const selectFavoriteIds = (s: RootState) => s.favorites;
```

```ts
// src/app/store.ts
import { configureStore } from '@reduxjs/toolkit';
export const makeStore = () => configureStore({
  reducer: {
    favorites: favoritesSlice.reducer,
    rideDraft: rideDraftSlice.reducer,
    addresses: addressesSlice.reducer,
  },
  middleware: (gDM) => gDM({ serializableCheck: false }),  // MMKV (M5.3)
});
export type RootState = ReturnType<ReturnType<typeof makeStore>['getState']>;
```

**Why RTK** (middle): `createSlice` (immutable без boilerplate), `createAsyncThunk` (M5.6), typed hooks. **Не** raw Redux.

### 5.3. Persistence (MMKV)

```bash
npx expo install mmkv @react-native-async-storage/async-storage
```

**MMKV** = **быстрый** (нативный, ~10× AsyncStorage), **sync** API, key-value. **Для persist** (не для частых запись):

```ts
// src/app/persist.ts
import { MMKV } from 'react-native-mmkv';
import { reduxSlicePersistence } from 'redux-persist';   # или само
export const storage = new MMKV();
// persistSlice / persistReducer (M5.3)
```

**Persist только нужное** (M5.3):
- ✅ Persist: `favorites`, `addresses`, `rideDraft` (черновик), `consent` (M6.6)
- ❌ **Never** persist: **токены** (M6.2 → Keychain, не MMKV!), server-данные (M4 Query)

**Асинхронность** (M5.3): MMKV sync → **быстро**; но **гидрация** при старте (M10: TTI). Не блокируйте UI — hydrate в фоне.

### 5.4. Optimistic updates (client-side, RTK)

**RTK optimistic** (через `createAsyncThunk` + `onFulfilled/onRejected`):
```ts
const toggleFavorite = createAsyncThunk(
  'favorites/toggle',
  async (driverId, { getState }) => {
    // мгновенно: toggle локально
    // API: PATCH /favorites
    await api.post(`/drivers/${driverId}/favorite`);
  },
  {
    rejectWithValue: (e) => (e as ApiError).message,
  },
);
// reducer: pending → оптимистично; rejected → откат (M4.9)
```

**Правила** (M4.9 + M5.7):
1. **Мгновенно** (UX)
2. **Откат** при ошибке
3. **Сверка** с сервером (invalidate)
4. **Идемпотентность** (M5.7/M9.5): повтор = тот же
5. **Очередь offline** (M5.7)

### 5.5. Sync-очередь: offline-first (M5.7)

**Проблема:** пользователь офлайн → действия «теряются»? **Решение: очередь** → при reconnect → drain.

```ts
// src/offline/syncQueue.ts
type QueuedAction = { id: string; type: string; payload: unknown; at: number };
export const syncQueue = {
  enqueue(a: QueuedAction) { /* MMKV: push */ },
  async drain() {           // M8: при reconnect
    for (const a of this.items()) {
      try { await this.execute(a); this.remove(a.id); }
      catch { this.requeue(a); break; }   // стоп — retry позже (M8)
    }
  },
};
```

**Правила** (M5.7):
- **Атомарные** действия (не «частичное»)
- **Идемпотентность** (M9.5): повтор = тот же
- **Retry** (M8): при reconnect → drain (M8)
- **Таймаут** (M8): старое — expire/отметить
- **Порядок** (M5.7): FIFO (если зависит)

### 5.6. ADR #1 (состояние)

```
# ADR #1 — Подход к состоянию

## Context
Приложение: server-state (M4) + client-state (RTK) + UI (useState). Offline-first.

## Decision
- Server-state → TanStack Query (M4)
- Client-state → Redux Toolkit (slice colocation, M5.1) + MMKV persist (M5.3)
- UI-state → useState
- Offline → sync-очередь (M5.7)

## Consequences
+ Чёткое разделение (middle)
+ Slice colocation = масштабируется (M14)
− Два инструмента (RTK + Query) — знать, когда какой
```

**Rationale:** **Разделение** — главный middle-навык. **Slice colocation** — масштабируемость (M14). **ADR** — «почему» (не «что»).

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Slice colocation + store (4ч)
- [ ] RTK: `favorites` (M5.2), `rideDraft` (черновик), `addresses`
- [ ] **Slice colocation** (M5.1): `src/features/*/slice.ts` + `app/store.ts`
- [ ] **Providers** (M1.2) в App.tsx
- [ ] **Типизация**: `RootState`, `AppDispatch` (typed hooks)
- [ ] **Правило** (M5.2): server → Query (M4), client → RTK

### Шаг 2. Persistence (4ч)
- [ ] **MMKV** (M5.3): persist `favorites`, `addresses`, `rideDraft`
- [ ] **Не persist** (M5.3): токены (M6.2), server-данные
- [ ] **Гидрация** (M10): фоновая, без блокировки UI
- [ ] Проверить: restart → избранное/черновик сохранены

### Шаг 3. Optimistic + откат (4ч)
- [ ] **favorites** (M5.4): toggle мгновенно + API + откат при ошибке
- [ ] **addresses** (M4.9): optimistic delete + откат
- [ ] **Сверка** (M5.4): invalidate после
- [ ] **Идемпотентность** (M9.5): повтор = тот же

### Шаг 4. Sync-очередь: offline-first (4ч)
- [ ] **Очередь** (M5.7): enqueue/dequeue/drain
- [ ] **Атомарность** + **идемпотентность** (M5.7/M9.5)
- [ ] **Reconnect** (M8): drain при reconnect (M8)
- [ ] **Таймаут** (M8) + **порядок** (M5.7)
- [ ] **Симуляция** (M4): офлайн → действия → онлайн → drain

### Шаг 5. ADR #1 (2ч)
- [ ] `docs/adr/0001-state-approach.md` (M5.6)

### Шаг 6. Отчёт (2ч) — `docs/progress/M5.md`
- Разделение: server/client/UI (примеры)
- Slice colocation: структура
- Persist: что persist, что нет
- Optimistic: мгновенно + откат
- Sync-очередь: offline → drain
- ADR #1
- Ошибки: 3+

---

## ✅ Практическое задание модуля

### Задача: «Разделение + RTK + MMKV + optimistic + sync-очередь + ADR #1»

**1. Slice colocation + store (4ч)** — favorites/rideDraft/addresses; typed; **server→Query**

**2. Persistence (4ч)** — MMKV (favorites/addresses/draft); **не** токены; гидрация

**3. Optimistic (4ч)** — toggle + откат + сверка + идемпотентность

**4. Sync-очередь (4ч)** — enqueue/dequeue/drain; reconnect (M8); offline

**5. ADR #1 (2ч)** — `docs/adr/0001-state-approach.md`

**6. Отчёт (2ч)**

**Критерии (из PROJECT):**
- [ ] **favorites** persist (MMKV), **откат** при ошибке (оптимистично)
- [ ] **Разделение** (M5.1): server→Query, client→RTK, UI→useState
- [ ] **Slice colocation** (M5.1): slices по фичам
- [ ] **Черновик** (M5.2): переживает restart
- [ ] **Sync-очередь** (M5.7): offline → reconnect → drain (идемпотентно)
- [ ] **ADR #1** задокументирован
- [ ] **Гидрация** (M10): не блокирует UI

---

## 📦 Итог

Вы понимаете **три типа состояния** и **не смешиваете** их (главный middle-навык). RTK (client) + Query (server) + useState (UI). **Slice colocation** масштабируется. **Offline-first** через sync-очередь. **ADR #1** — «почему».

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Server-state в Redux | «кэш руками», баги | Query (M4) (M5.1) |
| Один god-slice | Хаос, конфликты | Slice colocation (M5.1) |
| Persist токенов в MMKV | Кража (крадёт MMKV) | **Keychain** (M6.2) |
| Persist всего | Медленный старт, бloat | Только нужное (M5.3) |
| Optimistic без отката | Ложное состояние | Откат (M5.4) |
| Очередь без идемпотентности | Дубли (M9) | Idempotency (M9.5) |
| Drain при каждом reconnect | Давление | Debounce/ограничить (M8) |
| Гидрация блокирует UI | Медленный старт (M10) | Фоновая (M5.3) |

---

## 💡 Советы на модуль

- **Где живёт состояние?** — server (Query), client (RTK), UI (useState). Не смешивайте.
- **Slice colocation** — масштабируемость (M14). Не god-store.
- **Persist** только своё. **Токены → Keychain** (M6.2).
- **Optimistic = мгновенно + откат + сверка + идемпотентность.**
- **Offline = очередь** (M5.7). Drain при reconnect (M8).

**[→ Модуль 6: Авторизация и безопасность](../Module_06_Auth_Security/content.md)**
