# Модуль 9: Платежи, чек, оценка

**[← Назад к индексу](../index.md)** | **[Модуль 8: Реалтайм](../Module_08_Realtime_Push/content.md)** | **Далее: [Модуль 10: Производительность](../Module_10_Performance/content.md) →**

---

## 🎯 Цель модуля

- **Деньги в коде**: целые копейки, форматирование, никогда float
- **Stripe Payment Sheet**: intent, **3DS**, decline, **idempotency**
- **Чек** поездки (receipt): офлайн-стойкий
- **Оценка** (звёзды + теги + комментарий) → в синк-очередь (M5)
- **Закрытие цикла**: поездка → оплата → чек → оценка → «Готово»

**Длительность: ~22 часа. После M9 → v0.3.0.**

---

## 📚 Теория

### 9.1. Деньги: правило №1 — целые копейки

```
🚫 const price = 850.45;          // float: 0.1 + 0.2 ≠ 0.3
✅ const priceCents = 85045;       // целое: 850.45 ₽
```

**Почему:** float — бинарный, не может точно представить десятичные доли. `0.1 + 0.2 = 0.30000000000000004`. Для денег это **не баг округления, а баг денег**. Всегда:
- **Храним и передаём** — целые **копейки/центы**
- **Показываем** — форматированием: `85045 → '850,45 ₽'`

```ts
// src/shared/money.ts — чистая, тестируется
export function formatCents(cents: number, currency = 'RUB'): string {
  const abs = Math.abs(cents) / 100;
  const int = Math.floor(abs);
  const frac = Math.round((abs - int) * 100);
  const intStr = int.toLocaleString('ru-RU');          // 12 345
  const sign = cents < 0 ? '−' : '';
  return `${sign}${intStr},${String(frac).padStart(2, '0')} ${currencySymbol(currency)}`;
}
// formatCents(85045) → '850,45 ₽'
// formatCents(1234567) → '12 345,67 ₽'
```

> 💡 Мидл всегда спрашивает «как хранишь деньги?» — ответ: **целые минимальные единицы + форматирование на показ**.

### 9.2. Stripe: модель intent

```
Клиент (RN)                    Бэкенд (ваш)                 Stripe
   │  «оплати поездку 123»        │                            │
   │─────────────────────────────▶│  POST /v1/payment_intents  │
   │                              │───────────────────────────▶│
   │                              │  ← clientSecret            │
   │  ← clientSecret              │                            │
   │                              │                            │
   │  PaymentSheet.present() ─────┼───────────────────────────▶│
   │   (3DS, если нужно)          │                            │
   │  ← confirmed / cancelled     │  ← webhook (async)         │
   │                              │  (succeeded/failed)        │
   ▼                              ▼                            ▼
   UI: успех/ошибка            Статус поездки:             (источник
                               PAYING → PAID               правды)
```

**Ключевые принципы:**
1. **Payment intent создаёт бэкенд** (секретный ключ Stripe — только на сервере). Клиент получает **clientSecret** (публичный).
2. **Клиент** вызывает `PaymentSheet.present({ clientSecret })` — Stripe сам обрабатывает карту, 3DS, CVC.
3. **Подтверждение** приходит **двумя путями**: (а) UI (present resolve/reject) + (б) **webhook** на бэкенд (источник правды). UI — для UX; webhook — для состояния.

```bash
npx expo install @stripe/stripe-react-native
```
(Нативный модуль → dev build. В Expo Go — нет.)

### 9.3. Тестовые карты Stripe

```
4242 4242 4242 4242, любой будущий MM/YY, любой CVC   → успех
4000 0000 0000 9995                                    → decline (insufficient funds)
4000 0000 0000 0002                                    → 3DS challenge (успех после)
```
(Создаёте **test mode** в Stripe Dashboard → test API keys. Никаких настоящих денег.)

### 9.4. Payment: клиент + 3DS + decline

```tsx
// src/features/payment/PaymentFlow.tsx
import { PaymentSheet, confirmStripeSetup } from '@stripe/stripe-react-native';

export async function payForRide(rideId: string, amountCents: number, onDone: (r: PayResult) => void) {
  // 1. Intent (бэкенд) — С idempotency key (M9.5)
  const { clientSecret, paymentIntentId } = await api.post('/payments/intent', {
    rideId, amountCents, currency: 'RUB',
    idempotencyKey: rideId,   // повтор = тот же intent
  });

  // 2. Show Stripe UI (PaymentSheet: карта + 3DS + CVC)
  try {
    await PaymentSheet.present({ clientSecret });
    // 3. Успех (UI) → проверяем статус (webhook мог ещё не дойти)
    const status = await pollIntentStatus(paymentIntentId);   // M9.4
    onDone(status === 'succeeded' ? { ok: true } : { ok: false, reason: 'declined' });
  } catch (e) {
    // 4. Отмена/decline (UI reject)
    onDone({ ok: false, reason: e.code === 'PaymentSheetError/cancel' ? 'cancelled' : 'declined' });
  }
}
```

**Опциональный setup intent** (сохранённая карта): `confirmStripeSetup` + `PaymentSheet` — «оплатить сохранённой картой». Для курса: fresh card (4242) достаточно.

**pollIntentStatus** (M9.4): после UI-успеха опрашиваем `GET /payments/intent/:id/status` (1–2 раза) — до того, как webhook обновит бэкенд-статус. Иначе «оплатил, а чек ещё не появился».

### 9.5. Idempotency: повтор платежа = один платёж

**Проблема:** пользователь тапнул «Оплатить», сеть запазнила, тапнул ещё раз → **два платежа**.

**Решение — idempotency key:**

```ts
// Каждая «логическая операция» имеет стабильный ключ
const key = `ride:${rideId}:pay`;   // один ride = один платёж

// Бэкенд: если key уже использован — вернуть ПРЕЖНИЙ результат, не создавать новый
POST /payments/intent  { idempotencyKey: key, ... }
  → при повторе с тем же key + теми же параметрами → тот же clientSecret
```

**Правила idempotency:**
- **Ключ стабilen** для операции (rideId, не `Date.now()`)
- **Тело совпадает** при повторе (иначе — ошибка, а не «тихий новый платёж»)
- **Всё, что создаёт/меняет состояние** (платёж, оценка, избранные) — с ключом
- **M5-очередь** (syncQueue) использует те же ключи → retry не дублирует

### 9.6. Статусы платежа + обработка исходов

```
CREATED → PAYING → SUCCEEDED (→ чек → оценка)
                → FAILED   (decline → «попробовать ещё» / «оплатить потом»)
                → CANCELLED (отмена)
```

```tsx
// PaymentScreen (M9):
{
  case 'PAYING':   → Spinner «Обработка платежа…» (PaymentSheet открыт)
  case 'SUCCEEDED':→ navigate('Receipt', { rideId })
  case 'FAILED':   → «Платёж не прошёл. [Попробовать ещё] [Оплатить позже]»
  case 'CANCELLED':→ вернуться, «Оплатить позже»
}
```

**«Оплатить позже»:** поездка `COMPLETED`, но неоплаченная → в списке «Мои поездки» с пометкой → можно оплатить позже (тот же intent-флоу). **Важно:** оплата — **отделяема** от поездки (не блокируем чек/оценку).

### 9.7. Чек (receipt): офлайн-стойкий

**Чек — это данные, которые уже получены** (поездка + платёж). Не должны «пропасть» при отсутствии сети:

```ts
// src/features/receipt/Receipt.ts
type Receipt = {
  rideId: string;
  from: { label }; to: { label };
  date: number;            // ts
  distanceM: number; durationMin: number;
  class: RideClass; 
  amountCents: number;     // целые (M9.1)
  payment: { status: 'SUCCEEDED' | 'PENDING'; method: string };
};
```

- **Источники:** `useQuery(['ride', id])` + статус платежа.
- **Офлайн:** данные уже в Query-кэше → чек показывается **без сети** (кэш M4). При первом открытии — с сети, дальше — из кэша.
- **Экран:** маршрут, время/дистанция, класс, сумма (формат M9.1), «Оценить» (CTA) + «Поделиться» (опц.).

### 9.8. Оценка: звёзды + теги + комментарий → очередь

```tsx
// src/features/rating/RateScreen.tsx
export function RateScreen({ rideId, driver }) {
  const [stars, setStars] = useState(0);
  const [tags, setTags] = useState<string[]>([]);
  const [comment, setComment] = useState('');
  const { data } = useQuery({ queryKey: ['driver', driver.id], queryFn: () => api.get(`/drivers/${driver.id}`), enabled: !!driver });

  const TAGS_BY_RATING = {
    1: ['Худшая поездка', 'Грязный автомобиль'],
    2: ['Было неприятно'],
    3: ['Нормально'],
    4: ['Хорошая поездка', 'Чисто'],
    5: ['Отлично!', 'Внимательный', 'Комфортно'],
  };

  const submit = async () => {
    if (stars === 0) return;
    // В синк-очередь (M5.7) — офлайн-стойко, idempotent (M9.5)
    dispatch(enqueue({ id: `rate:${rideId}`, type: 'rating', rideId, stars, tags, comment, ts: Date.now() }));
    // Если есть сеть — сразу сливаем (M5.7); иначе «отправится позже»
    await drainQueueIfOnline();
    navigation.popToTop();   // → главный экран
  };

  return (
    <SafeAreaView>
      <Avatar driver /> <Text>{driver.name}</Text>
      <Stars value={stars} onChange={setStars} />        {/* M2/модуль: 5 звёзд, тап = оценка */}
      {stars > 0 && (
        <Wrap>
          {TAGS_BY_RATING[stars].map((t) => <TagChip key={t} label={t} selected={tags.includes(t)} onToggle={...} />)}
        </Wrap>
      )}
      <TextInput value={comment} onChangeText={setComment} placeholder="Комментарий (необязательно)" multiline />
      <Button title={stars ? 'Отправить' : 'Пропустить'} onPress={stars ? submit : () => navigation.popToTop()} testID="rate-submit" />
    </SafeAreaView>
  );
}
```

**Почему в очередь (M5.7):** оценка часто — в машине, **плохая сеть**. В очередь → отправится при сети → **idempotent** (M9.5) → не дублируется.

### 9.9. Закрытие цикла

```
Order (M8: COMPLETED) → Payment (M9.4) → SUCCEEDED → Receipt (M9.7) → Rate (M9.8) → главный экран
                                    └→ FAILED → «попробовать/позже» → (Receipt без оплаты)
```

После `Rate` → `popToTop()` → **поездка завершена**. Черновик (M5) чистится (M7).

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Мок-бэкенд: платежи (6ч)
- [ ] `POST /payments/intent { rideId, amountCents, idempotencyKey }` → создаёт Stripe test intent → `clientSecret`
- [ ] **Idempotency** (M9.5): таблица idempotency_keys; повтор с тем же key+телом → тот же intent
- [ ] `GET /payments/intent/:id/status` → succeeded/failed/pending
- [ ] **Webhook** `/stripe/webhook` (payment_intent.succeeded/failed) → обновляет статус поездки (M9.6)
- [ ] (Stripe CLI: `stripe listen --forward-to localhost:4000/stripe/webhook` — локально)

### Шаг 2. PaymentSheet + исходы (6ч)
- [ ] `payForRide` (M9.4): intent → present → poll → исход
- [ ] **Три исхода** (тестовые карты): успех (4242), decline (9995), 3DS (0002)
- [ ] PaymentScreen: PAYING/FAILED/CANCELLED/SUCCEEDED + «попробовать ещё»/«позже»
- [ ] **Idempotency**: двойной тап → один платёж (проверить в логах бэкенда)

### Шаг 3. Чек (4ч)
- [ ] Receipt (M9.7): офлайн-стойкий (кэш M4), формат денег (M9.1)
- [ ] «Оплатить позже» → из «Мои поездки» (M4) → тот же флоу

### Шаг 4. Оценка (4ч)
- [ ] Rate (M9.8): звёзды + теги + комментарий + «пропустить»
- [ ] **В очередь** (M5.7) + idempotency (M9.5) + drain при сети
- [ ] **Офлайн-тест**: сломал сеть → оценил → «отправится позже» → включил сеть → ушло (не дубли)

### Шаг 5. Цикл + v0.3.0 (2ч)
- [ ] Полный: Order(COMPLETED) → Payment → Receipt → Rate → главный
- [ ] **Релиз v0.3.0** (EAS → TestFlight + Play internal)
- [ ] Smoke: все три исхода платежа + офлайн-оценка

### Шаг 6. Отчёт (2ч) — `docs/progress/M9.md`
- Деньги: правило целых + форматирование
- Три исхода платежа (скрины: success/decline/3DS)
- Idempotency: доказательство «двойной тап = 1 платёж»
- Офлайн-оценка (скрин «отправится позже» → ушло)
- Ошибки: 3+

---

## ✅ Практическое задание модуля

### Задача: «Платёж + чек + оценка + v0.3.0»

**1. Мок-bэкенд (6ч)** — Stripe test intent + idempotency + webhook + status

**2. Платёж (6ч)**
- [ ] PaymentSheet: intent → present → poll → три исхода (success/decline/3DS)
- [ ] **Idempotency**: двойной тап → один платёж (тест)
- [ ] «Оплатить позже» (отдельно от поездки)

**3. Чек (4ч)** — офлайн-стойкий (кэш), формат денег, «поделиться»(опц.)

**4. Оценка (4ч)** — звёзды+теги+комментарий → очередь → idempotency → офлайн-стойко

**5. Цикл + v0.3.0 (2ч)** — полный; релиз; smoke (3 исхода + офлайн-оценка)

**6. Отчёт (2ч)**

**Критерии:**
- [ ] Деньги: **целые копейки** везде, форматирование (unit-тесты formatCents)
- [ ] Платёж: success / decline / 3DS — все три обработаны
- [ ] Idempotency: повтор = один платёж (доказано)
- [ ] Чек: офлайн-стойкий
- [ ] Оценка: офлайн-стойкая, idempotent, не дублируется
- [ ] Полный цикл: Order → Payment → Receipt → Rate → главный
- [ ] **v0.3.0** на TestFlight + Play internal

---

## 📦 Итоговая структура после M9

```
src/features/
├── payment/
│   ├── pay.ts              # payForRide (intent+present+poll)
│   ├── PaymentScreen.tsx   # исходы
│   └── idempotency.ts      # ключи
├── receipt/
│   ├── ReceiptScreen.tsx   # офлайн-стойкий
│   └── format.ts           # (→ shared/money)
├── rating/
│   ├── RateScreen.tsx      # звёзды+теги+коммент
│   └── stars.tsx           # Stars
└── orders/                 # (M4) + «оплатить позже»
src/shared/
└── money.ts                # formatCents (чистая)
```

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Деньги в float | 850.45 → 850.4500000001 | Целые копейки (M9.1) |
| Secret key в RN | Кража | Только бэкенд; клиент — clientSecret |
| Нет idempotency | Двойной платёж | Ключ + повтор = тот же intent |
| UI-подтверждение как истина | Webhook ещё не пришёл | Poll status + webhook |
| Чек требует сеть | «Пропал» офлайн | Кэш M4 (офлайн-стойкий) |
| Оценка требует сеть | Потеряна | syncQueue (M5) + idempotency |
| 3DS «завис» | Не понимаете флоу | Stripe сам; не блокируем UI |
| Платёж блокирует чек | Плохой UX | Отделяем: чек/оценка — без оплаты |

---

## 💡 Советы на модуль

- **Целые копейки** — первый и главный принцип. Запомните навсегда.
- **Idempotency — суперсила.** Платёж, оценка, избранное — всё с ключом. Retry не дублирует.
- **Два пути подтверждения** (UI + webhook). UI — для UX, webhook — для правды.
- **Офлайн-стойкость** — через кэш (M4) + очередь (M5). Не теряем.
- **v0.3.0** — полный цикл поездки. Отметьте.

**[→ Модуль 10: Производительность](../Module_10_Performance/content.md)**
