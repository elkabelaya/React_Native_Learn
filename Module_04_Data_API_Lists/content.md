# 📘 Модуль 4: Данные, API, списки

**[← Назад к индексу](../index.md)** | **Далее: [Модуль 5](../Module_05_State_Management/content.md) →**

**Самый большой модуль** (2 недели). Мы построим «слой данных»: HTTP-клиент, TanStack Query, виртуализированные списки с пагинацией. На базе этого слоя реализуем три главные фичи RideGo: **поиск водителей** (поиск + фильтры + бесконечная прокрутка), **история поездок** (секции + refresh) и **сохранённые адреса** (CRUD с optimistic updates).

**Цели модуля:**
1. Понять разделение **server-state** (данные с API) и **client-state** (состояние UI)
2. Построить API-клиент: axios + интерсепторы + модель ошибок + отмена запросов
3. Освоить TanStack Query: query keys, кэш, invalidation, optimistic updates, бесконечная пагинация
4. Построить быстрые списки: FlatList → FlashList, стабильные keys, `renderItem`-дисциплина
5. Дебаунс поиска, pull-to-refresh, skeleton-загрузка, пустые/ошибочные состояния

**Время выполнения:** ~30 часов.

---

## 1. Теория: server-state ≠ client-state

**Ключевой концепт middle-разработчика:** данные, которые приходят извне (API, WebSocket), — это **server-state**, а не «состояние приложения».

| | Client-state | Server-state |
|---|---|---|
| Пример | открыт ли bottom sheet, выбранная вкладка | список водителей, история поездок |
| Владелец | React (useState, Redux) | **API** (источник правды) |
| Копия | — | кэш (TanStack Query) |
| Действия | setState | fetch, invalidation, mutation |
| Инструмент | React, RTK | **TanStack Query** |

> 💡 **Почему не Redux для API?** Redux — для client-state. Держать server-данные в Redux = реимплементировать кэш (stale-таймауты, рефетчи, отмены, реретраи) руками. TanStack Query — готовый, протестированный. **Средний RN-разработчик держит оба: RTK (client) + Query (server)**.

### TanStack Query: что он даёт

```
useQuery({ queryKey, queryFn })
  → кэширует по queryKey
  → staleTime: «свежесть»
  → рефетч при фокусе/восстановлении сети
  → отмена (AbortController)
  → реретраи
  → loading / error / data состояния — из коробки
  → invalidate → пересчёт зависимых кэшей
```

```bash
npm i @tanstack/react-query
```

Провайдер (в App.tsx):
```tsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,          // 30 с — «свежие»
      gcTime: 5 * 60_000,          // в памяти 5 минут
      refetchOnWindowFocus: false, // mobile: по событию, не по фокусу
      retry: 2,
    },
  },
});

<QueryClientProvider client={queryClient}>...</QueryClientProvider>
```

### Query keys — «структура данных»

```ts
// queryKey — это ключ кэша. Структура = семантика.
['drivers', { q: 'airport', page: 2, class: 'comfort' }]
['rides', { cursor: 'abc' }]
['rides', '123']           // детали одной поездки
['favorites']
['addresses']
```

> 💡 **Правило:** ключ «родителя» (префикс) для invalidation. `invalidateQueries(['drivers'])` — все вариации `['drivers', ...]` станут stale.

---

## 2. Практика: API-клиент (src/api/client.ts)

```bash
npm i axios
```

### Файловая структура

```
src/api/
├── client.ts        # axios instance + interceptors + error model
├── auth.ts          # (модуль 6)
├── drivers.ts       # M4
├── rides.ts         # M4
├── addresses.ts     # M4
└── types.ts         # DTO
```

### client.ts

```ts
// src/api/client.ts
import axios, { AxiosError } from 'axios';
import { Platform } from 'react-native';

export class ApiError extends Error {
  constructor(
    public code: string,          // 'NETWORK' | 'UNAUTHORIZED' | 'VALIDATION' | 'SERVER'
    message: string,
    public status?: number,
    public retryable = false,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export const api = axios.create({
  baseURL: API_URL,               // из env (см. ниже)
  timeout: 15_000,
  headers: { 'Content-Type': 'application/json' },
});

// ── Ответ: маппим на ApiError ──
api.interceptors.response.use(
  (res) => res,
  (err: AxiosError) => {
    let e: ApiError;
    if (!err.response) {
      e = new ApiError('NETWORK', 'Нет соединения с сервером', undefined, true);
    } else if (err.response.status === 401) {
      e = new ApiError('UNAUTHORIZED', 'Сессия истекла', 401, false);
      // refresh — подключим в модуле 6 (single-flight)
    } else if (err.response.status >= 500) {
      e = new ApiError('SERVER', 'Сервер занят. Попробуйте позже.', err.response.status, true);
    } else {
      const msg = (err.response.data as any)?.message ?? 'Ошибка';
      e = new ApiError('VALIDATION', msg, err.response.status, false);
    }
    return Promise.reject(e);
  },
);

// ── Запрос: токены (заглушка; настоящая — модуль 6) ──
api.interceptors.request.use(async (config) => {
  // const token = await getAccessToken();   // M6
  // if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

> 💡 **Почему свой класс `ApiError`?** В UI мы хотим **отличать** «нет сети» (кнопка «повторить»), «401» (автовход), «5xx» (кнопка «повторить»), «400» (показываем текст). Generic `Error` — не даёт. Это **контракт** между API-слоем и UI.

### Env (URL)

```bash
npm i -D dotenv
```
```ts
// src/config.ts
import Constants from 'expo-constants';
export const API_URL =
  (Constants.expoConfig?.extra?.API_URL as string) ?? 'http://localhost:3001/api';
```
```json
// app.json → "extra": { "API_URL": "http://192.168.1.42:3001/api" }  (ваш LAN-IP)
```

> ⚠️ На **реальном устройстве** `localhost` — это само устройство. Нужен **LAN-IP компьютера** (`ipconfig getifaddr en0` / `ip addr`). На эмуляторе Android `10.0.2.2` = localhost хоста.

---

## 3. Практика: DTO и маппинг (src/api/types.ts)

**Принцип:** API отдаёт **DTO** (camelCase, может быть с лишним). Внутри приложения — **доменные типы** (PascalCase или domain-имена, только нужные поля).

```ts
// src/api/types.ts — DTO (что приходит с сервера)
export type DriverDTO = {
  id: string;
  name: string;
  photoUrl: string | null;
  rating: number;        // 0..5
  car: { model: string; color: string; plate: string };
  class: 'economy' | 'comfort' | 'business';
  distanceM: number;     // метры до пользователя
  etaMin: number;        // минуты
  priceCents: number;    // цена в центах (избегает float!)
};

// src/types.ts — доменные типы (внутри приложения)
export type Driver = {
  id: string;
  name: string;
  photo: string | null;
  rating: number;
  car: string;           // "Toyota Corolla · A123BC"
  class: DriverClass;
  distanceM: number;
  etaMin: number;
  priceCents: number;
};

// Мэппинг (в api/drivers.ts)
export const mapDriver = (d: DriverDTO): Driver => ({
  id: d.id,
  name: d.name,
  photo: d.photoUrl,
  rating: d.rating,
  car: `${d.car.model} · ${d.car.plate}`,
  class: d.class,
  distanceM: d.distanceM,
  etaMin: d.etaMin,
  priceCents: d.priceCents,
});
```

> 💡 **Почему маппинг?** DTO — контракт с сервером (может меняться). Доменные типы — наш внутренний контракт. Изменится API → меняется один файл (`mapDriver`), а не 20 экранов. **Middle не размазывает DTO по UI.**

### Форматирование (src/shared/format.ts)

```ts
export const formatPrice = (cents: number): string =>
  (cents / 100).toLocaleString('ru-RU', { style: 'currency', currency: 'RUB', maximumFractionDigits: 0 });

export const formatDistance = (m: number): string =>
  m < 1000 ? `${m} м` : `${(m / 1000).toFixed(1)} км`;

export const formatEta = (min: number): string =>
  min <= 1 ? 'везде' : `через ${min} мин`;
```

---

## 4. Практика: API-слой (drivers, rides, addresses)

### src/api/drivers.ts

```ts
import { api } from './client';
import { Driver, DriverClass, DriverDTO, mapDriver } from './types';

export type DriversQuery = {
  q?: string;                 // поиск по имени/авто
  class?: DriverClass | 'all';
  page?: number;              // 1-based
  pageSize?: number;          // default 20
};

type DriversResponse = {
  items: DriverDTO[];
  total: number;
  page: number;
  hasMore: boolean;
};

export async function getDrivers(params: DriversQuery, signal?: AbortSignal) {
  const { data } = await api.get<DriversResponse>('/drivers', { params, signal });
  return { ...data, items: data.items.map(mapDriver) };
}
```

### src/api/rides.ts

```ts
import { api } from './client';
import { RideDTO, mapRide, Ride } from './types';

export async function getRides(cursor?: string, signal?: AbortSignal) {
  const { data } = await api.get<{ items: RideDTO[]; nextCursor: string | null }>('/rides', {
    params: { cursor }, signal,
  });
  return { items: data.items.map(mapRide), nextCursor: data.nextCursor };
}

export async function getRide(id: string, signal?: AbortSignal) {
  const { data } = await api.get<RideDTO>(`/rides/${id}`, { signal });
  return mapRide(data);
}

export async function cancelRide(id: string) {
  await api.post(`/rides/${id}/cancel`);
}
```

### src/api/addresses.ts (CRUD)

```ts
export async function getAddresses(signal?: AbortSignal) {
  const { data } = await api.get<AddressDTO[]>('/addresses', { signal });
  return data.map(mapAddress);
}
export async function createAddress(a: Omit<Address, 'id'>) {
  const { data } = await api.post<AddressDTO>('/addresses', a);
  return mapAddress(data);
}
export async function deleteAddress(id: string) {
  await api.delete(`/addresses/${id}`);
}
```

---

## 5. Практика: поиск с debounce (src/hooks/)

**Почему debounce?** Каждый символ → запрос → 10 запросов/сек. Дебаунс — ждём паузу (500 мс).

```ts
// src/hooks/useDebouncedValue.ts
import { useEffect, useState } from 'react';

export function useDebouncedValue<T>(value: T, delay = 500): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);   // ← отмена при новом value
  }, [value, delay]);
  return debounced;
}
```

> ⚠️ **Важно:** TanStack Query сам **отменяет** предыдущий запрос при смене ключа (через AbortController). Debounce — на уровне **ввода**, отмена — на уровне запроса. Два слоя защиты.

---

## 6. Практика: экран поиска водителей

```
src/features/drivers/
├── DriversScreen.tsx        # search bar + классы (chips) + список
├── DriverCard.tsx           # карточка: фото, имя, рейтинг, авто, цена
├── useDrivers.ts            # hook: search + class + infinite query
└── index.ts
```

### useDrivers (TanStack Query infinite)

```ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { getDrivers, DriversQuery } from '@/api/drivers';

export function useDrivers(q: string, class_: DriversQuery['class']) {
  const debouncedQ = useDebouncedValue(q, 500);

  return useInfiniteQuery({
    queryKey: ['drivers', { q: debouncedQ, class: class_ }],
    queryFn: ({ pageParam, signal }) =>
      getDrivers({ q: debouncedQ || undefined, class: class_, page: pageParam, pageSize: 20 }, signal),
    initialPageParam: 1,
    getNextPageParam: (last) => (last.hasMore ? last.page + 1 : undefined),
    // ⚠️ pageParam — номер страницы (1-based). getNextPageParam — следующая
  });
}
```

### DriversScreen

```tsx
import { useState, useMemo, useCallback } from 'react';
import { View, Text, TextInput, FlatList, RefreshControl, StyleSheet, Pressable } from 'react-native';
import { useDrivers } from './useDrivers';
import { DriverCard } from './DriverCard';
import { Skeleton, Badge, Button } from '@/ui';
import { useTheme } from '@/theme/ThemeProvider';
import { DriverClass } from '@/api/types';

const CLASSES: { id: DriverClass | 'all'; label: string }[] = [
  { id: 'all', label: 'Все' },
  { id: 'economy', label: 'Эконом' },
  { id: 'comfort', label: 'Комфорт' },
  { id: 'business', label: 'Бизнес' },
];

export function DriversScreen() {
  const t = useTheme();
  const [q, setQ] = useState('');
  const [class_, setClass_] = useState<DriverClass | 'all'>('all');

  const { data, isLoading, isError, error, isFetchingNextPage, refetch, fetchNextPage, refetchInterval } = useDrivers(q, class_);

  const items = useMemo(() => data?.pages.flatMap(p => p.items) ?? [], [data]);
  const hasMore = !!data?.pages.at(-1)?.hasMore;

  const renderItem = useCallback(({ item }: { item: Driver }) => (
    <DriverCard driver={item} onPress={() => {}} />
  ), []);

  return (
    <View style={[styles.root, { backgroundColor: t.colors.bg }]}>
      <TextInput placeholder="Поиск: имя, авто…" value={q} onChangeText={setQ}
        style={[styles.search, { backgroundColor: t.colors.surface, color: t.colors.textPrimary }]} />

      {/* Chips-фильтры */}
      <FlatList
        data={CLASSES}
        keyExtractor={(c) => c.id}
        horizontal
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={{ gap: 8 }}
        ListHeaderComponent={<Text>Класс</Text>}
        renderItem={({ item }) => (
          <Pressable
            onPress={() => setClass_(item.id)}
            style={[styles.chip, { backgroundColor: class_ === item.id ? t.colors.brand : t.colors.surface }]}
          >
            <Text>{item.label}</Text>
          </Pressable>
        )}
      />

      <FlatList
        data={items}
        keyExtractor={(d) => d.id}                      // ← стабильные keys!
        renderItem={renderItem}
        contentContainerStyle={{ gap: 8, padding: 16 }}
        refreshControl={<RefreshControl refreshing={!!isFetching && !isLoading} onRefresh={refetch} tintColor={t.colors.brand} />}
        onEndReached={() => { if (hasMore && !isFetchingNextPage) fetchNextPage(); }}
        onEndReachedThreshold={0.4}
        ListEmptyComponent={
          isLoading ? <SkeletonList />
          : isError ? <ErrorState error={error} onRetry={refetch} />
          : <EmptyState text="Водители не найдены" />
        }
        ListFooterComponent={isFetchingNextPage ? <Spinner /> : null}
      />
    </View>
  );
}
```

> 💡 **`onEndReached`** — «дотянулись до конца». `onEndReachedThreshold: 0.4` — начинаем подгружать за 40% высоты. Это **бесконечная прокрутка**.

---

## 7. Практика: FlatList vs FlashList

### Когда FlatList «проседает»

- **> 100 видимых/сохранёльных** элементов со сложным `renderItem` (картинки, много текста)
- **Частые вставки/удаления** (список обновляется по WS — модуль 8)
- **Скролл с анимациями** (reordered, swipe-to-delete)

### FlashList (Shopify) — когда брать

```bash
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  keyExtractor={(d) => d.id}
  renderItem={renderItem}
  estimatedItemSize={96}      // ⚠️ обязательно: оценка размера элемента
  drawDistance={500}          // сколько px «за пределами» рендерить
/>
```

> ⚠️ **`estimatedItemSize`** — критичен. Плохая оценка → скачки при скролле. Начните с высоты одного элемента.

**Решение для курса:** DriverCard (сkeleton, фото, текст) — **FlashList**. OrdersList (простой) — **FlatList** (учим оба). В модуле 10 вернёмся: покажем бенчмарк.

---

## 8. Практика: история поездок (OrdersList)

**Секции + cursor-пагинация + pull-to-refresh:**

```ts
// useRides.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { getRides } from '@/api/rides';
import { Ride } from '@/types';

export function useRides() {
  return useInfiniteQuery({
    queryKey: ['rides'],
    queryFn: ({ pageParam, signal }) => getRides(pageParam ?? undefined, signal),
    initialPageParam: undefined as string | undefined,   // первый запрос без cursor
    getNextPageParam: (last) => last.nextCursor ?? undefined,
  });
}
```

**Группировка по дням** (SectionList):
```ts
// Утилиты: группируем rides по дате (Сегодня, Вчера, «12 марта»)
export function groupByDay(rides: Ride[]): { day: string; items: Ride[] }[] {
  const map = new Map<string, Ride[]>();
  for (const r of rides) {
    const day = dayLabel(new Date(r.createdAt));   // «Сегодня» / «Вчера» / «12 марта»
    map.get(day)?.push(r) ?? map.set(day, [r]);
  }
  return [...map.entries()].map(([day, items]) => ({ day, items }));
}
```

**SectionList:**
```tsx
<SectionList
  sections={sections}
  keyExtractor={(r) => r.id}
  renderItem={({ item }) => <RideRow ride={item} />}
  renderSectionHeader={({ section }) => (
    <Text style={styles.sectionHeader}>{section.title}</Text>
  )}
  stickySectionHeadersEnabled        // липкие заголовки
  refreshControl={<RefreshControl ... />}
  onEndReached={() => fetchNextPage()}
/>
```

> 💡 **SectionList vs FlatList:** SectionList — когда есть **заголовки-разделители**. В остальном — те же props (`onEndReached`, `renderItem`).

---

## 9. Практика: сохранённые адреса (CRUD + optimistic)

```
src/features/addresses/
├── AddressesScreen.tsx
├── AddressRow.tsx
└── useAddresses.ts
```

**Optimistic delete** (удаляем «мгновенно», откатываем при ошибке):

```ts
export function useDeleteAddress() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: deleteAddress,
    onMutate: async (id) => {
      await qc.cancelQueries({ queryKey: ['addresses'] });
      const prev = qc.getQueryData<Address[]>('addresses');
      qc.setQueryData('addresses', (old) => old?.filter(a => a.id !== id));   // оптимистично
      return { prev };
    },
    onError: (_e, _id, ctx) => ctx?.prev && qc.setQueryData('addresses', ctx.prev),   // откат
    onSettled: () => qc.invalidateQueries({ queryKey: ['addresses'] }),               // сверка
  });
}
```

**Свайп-удаление** (Reanimated gesture):
```tsx
import { PanGestureHandler } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withTiming, runOnJS } from 'react-native-reanimated';

const tx = useSharedValue(0);
const gesture = useAnimatedGestureHandler({
  onStart: () => { tx.value = 0; },
  onActive: (e) => { tx.value = e.translationX; },
  onEnd: (e) => {
    const shouldDelete = e.translationX < -80;
    tx.value = withTiming(shouldDelete ? -200 : 0);
    if (shouldDelete) runOnJS(onDelete)();
  },
});

<PanGestureHandler onHandlerStateChange={...}>
  <Animated.View style={{ transform: [{ translateX: tx }] }}>
    {/* Под карточкой — красная зона удаления */}
  </Animated.View>
</PanGestureHandler>
```

> 💡 **`runOnJS`** — из worklet (UI thread) в JS (setState, mutation). Мост между мирами.

---

## 10. Практика: состояния UI (стандарт)

**Каждый данные-экран = 5 состояний.** Запомните шаблон:

```
┌ isLoading      → Skeleton (пульсирующие placeholders)
├ isError        → ErrorState: иконка + текст + [Повторить] (refetch)
├ empty          → EmptyState: иконка + «Нет данных» + CTA
├ isFetching     (bg) → маленький Spinner в шапке / dim
└ data           → контент
```

**Компоненты** (добавьте в `src/ui/`):
```tsx
// Skeleton.tsx — пульс (Reanimated)
export function Skeleton({ width, height = 16, radius }: {...}) {
  const opacity = useSharedValue(1);
  useEffect(() => {
    'worklet';   // ← пульс в UI thread
    opacity.value = withSequence(
      withTiming(0.4, { duration: 600 }),
      withTiming(1, { duration: 600 }),
    );
    // loop:
    const loop = () => { /* с задержкой */ };
  }, []);
  return <Animated.View style={{ width, height, borderRadius: radius, opacity, backgroundColor: 'gray' }} />;
}

// ErrorState.tsx
export function ErrorState({ error, onRetry }: { error?: Error; onRetry: () => void }) {
  return (
    <View>
      <Text>{(error as ApiError)?.code === 'NETWORK' ? 'Нет сети' : error?.message}</Text>
      <Button title="Повторить" onPress={onRetry} variant="secondary" />
    </View>
  );
}
```

---

## 11. Практика: mock-бэкенд (подробнее)

Набросок (полный — в проекте):

```js
// ridego-api/src/app.js (Express)
const express = require('express');
const app = express();
app.use(express.json());

// В-памяти "БД" (или SQLite)
let drivers = seedDrivers(200);   // генерация
let rides = seedRides(50);

app.get('/api/drivers', (req, res) => {
  const { q, class: cls, page = 1, pageSize = 20 } = req.query;
  let items = drivers;
  if (q) items = items.filter(d => d.name.toLowerCase().includes(q.toLowerCase()));
  if (cls !== 'all') items = items.filter(d => d.class === cls);
  const start = (page - 1) * pageSize;
  const slice = items.slice(start, start + pageSize);
  res.json({ items: slice, total: items.length, page: +page, hasMore: start + pageSize < items.length });
});

app.get('/api/rides', (req, res) => {
  const { cursor } = req.query;   // base64(lastIndex)
  const idx = cursor ? Buffer.from(cursor, 'base64').toString() : 0;
  const slice = rides.slice(+idx, +idx + 20);
  res.json({ items: slice, nextCursor: +idx + 20 < rides.length ? Buffer.from(String(+idx + 20)).toString('base64') : null });
});

// CRUD /addresses, POST /rides, /rides/:id/cancel ...
```

> 💡 **Cursor = base64(index)** — честная imитация cursor-пагинации (как в реальных API: Stripe, Linear).

---

## 📝 Домашнее задание (Модуль 4)

### Обязательные задачи

1. **API-клиент**
   - axios + interceptors (ApiError, timeout)
   - `src/api/` — drivers, rides, addresses + types + маппинг
   - Env URL (real device LAN IP)
   - **Проверка:** без сети → `ApiError('NETWORK')`; 401 → `UNAUTHORIZED`; 500 → `SERVER`

2. **Поиск водителей (полный)**
   - Search bar с debounce (500 мс) + иконка «очистить»
   - Chips-фильтры (класс)
   - FlashList: поиск, пагинация, pull-to-refresh
   - DriverCard: фото, имя, рейтинг (звёзды), авто, цена, «быстрее всех» (бейдж)
   - Состояния: skeleton, empty, error (нет сети / 500)
   - **60 fps** на реальном Android при скролле 200 карточек

3. **История поездок (полный)**
   - SectionList по дням (Сегодня, Вчера, «12 марта»)
   - Cursor-пагинация + pull-to-refresh
   - RideRow: маршрут, дата, цена, статус (Badge)
   - Пустое состояние: иконка + «Поездок пока нет» + кнопка «Вызвать»
   - Тап → RideDetail (заглушка, с деталями)

4. **Сохранённые адреса (CRUD)**
   - Список + add (форма: label, search → координаты)
   - Свайп-удаление (optimistic)
   - Optimistic add
   - **Откат** при ошибке (симулируйте: заглушите сеть)

5. **Тесты API-слоя (unit)**
   - `api/client.test.ts`: маппинг ошибок (network, 401, 500, 400)
   - `api/drivers.test.ts`: маппинг DTO → Driver, пагинация (hasMore)
   - `useDrivers.test.tsx` (renderHook + QueryClientProvider): debounce, invalidation
   - **MSW** — mock-сервер (на уровне fetch/axios adapter)

### Дополнительные задачи

6. **Бенчмарк FlatList vs FlashList** на DriverCard: 1000 элементов, замерите TTFD (time to first draw) + frame drops
7. **Исключение «двойного fetch»** при навигации туда-назад (staleTime)
8. **Отмена запроса при смене запроса** — проверьте в Network, что отменяются (AbortController)

### Критерии сдачи

- [ ] 3 фичи (drivers, rides, addresses) — все состояния, пагинация, refresh
- [ ] Debounce: 10 запросов «аэропорт» = 1 запрос (с паузой)
- [ ] Infinite scroll: 200+ элементов без фриза (60 fps)
- [ ] Optimistic: удаление мгновенно, откат при ошибке
- [ ] Typecheck + Lint зелёные
- [ ] Unit-тесты: client + drivers + useDrivers (msw)
- [ ] PR: `feat: api layer + drivers + orders + addresses`

### Как проверить

1. **Drivers:** введите «аэропорт» → skeleton → список. Быстро печатайте → один запрос (Network).
2. **Скролл:** 200 карточек, включите FPS — ≥ 55 fps
3. **Orders:** скролл вниз → подгрузка. Пульс refresh.
4. **Addresses:** добавьте, удалите (свайп) → исчез мгновенно. Заглушите сеть → удалите → вернётся (откат).
5. **Тесты:** `npm test` — зелёные.

---

## 💡 Полезные команды (модуль 4)

```bash
# Установка
npm i axios @tanstack/react-query
npx expo install @shopify/flash-list
npm i -D msw @types/axios

# Запуск mock-бэкенда
cd ../ridego-api && npm i express cors && npm run dev

# Тесты
npm i -D jest jest-expo @testing-library/react-native msw
# jest.config.js: preset: 'jest-expo'
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `ECONNREFUSED` на реальном устройстве | `localhost` = само устройство | LAN-IP (`192.168.x.x`) в app.json extra |
| Двойной запрос при монтировании | React StrictMode + staleTime | Убедитесь: `staleTime` > 0; это норм. Для прода — `refetchOnMount: false` |
| Список «прыгает» при подгрузке | Нет `estimatedItemSize` (FlashList) / нестабильные keys | Стабильные `id`; оцените размер |
| `keyExtractor` warning (duplicate keys) | Один id встречается дважды | Убедитесь: уникальные id (или `id + index` как fallback) |
| Pull-to-refresh не срабатывает | Нет `refreshControl` у FlatList | Добавьте (или `refreshControl` у SectionList) |
| Debounce «не работает» | `setTimeout` без cleanup | `useEffect` + `clearTimeout` (см. урок) |
| MSW: запросы уходят в реальность | Axios не ходит через fetch | Конфигурируйте msw на `XMLHttpRequest` (axios использует XHR в RN) |
| Оптимистичное обновление «ломает» порядок | setQueryData без учёта порядка | После `onSettled` — `invalidateQueries` (сверка с сервером) |
| FlatList «белый» (нет элементов) | `data` undefined | `data={items ?? []}` + `ListEmptyComponent` |

---

## 💡 Советы на модуль

- **Server-state — это Query, не Redux.** Не тащите API-данные в RTK.
- **Query key = документация.** Прочитав `['drivers', { q, class, page }]` — понятно, что кэшируется.
- **Одна кнопка «Повторить» на каждый error.** Не площите «Попробуйте позже» — давайте действие.
- **Скелетоны — это UX.** Пустой экран 1.5 с — «зависло». Skeleton — «работаем».
- **Стабильные keys.** Если `keyExtractor` использует `index` — вы заложили баг (порядок меняется → перерисовка всего).
- **Проверяйте реальный Android.** FlashList vs FlatList — разница видна только на «слабом» устройстве.

**[→ Модуль 5: Состояние и локальное хранилище](../Module_05_State_Management/content.md)**
