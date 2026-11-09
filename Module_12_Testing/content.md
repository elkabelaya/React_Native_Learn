# Модуль 12: Тестирование

**[← Назад к индексу](../index.md)** | **[Модуль 11: Натив](../Module_11_Native_Platform/content.md)** | **Далее: [Модуль 13: CI/CD и релизы](../Module_13_CICD_Reloading/content.md) →**

---

## 🎯 Цель модуля

- **Pyramid**: unit (React Native Testing Library) → integration → e2e
- **RTL** + Jest: рендер, interaction, queries, mocking (M4 api, M8 socket)
- **Unit** для «мозга»: FSM (M8), calcPrice (M7.7), money (M9.1), geo (M7)
- **Integration**: флоу «от → к → классы → вызов»
- **e2e** (Maestro): ключевые сценарии на sim/emu
- **Цели** (из PROJECT): unit ≥ 80% (critical paths 100%), e2e на CI, **mutation testing**

**Длительность: ~22 часа. «Дом, который вы строите, чтобы не рухнуть».**

---

## 📚 Теория

### 12.1. Pyramida тестов

```
        /\       e2e (Maestro): 3–5 ключевых сценариев. Медленно, дорого, на sim/emu.
       /  \
      /集成\  integration: флоу (несколько модулей вместе). Средне.
     /      \
    / unit  \  unit (RTL + чистые функции): много, быстро, дёшево. 80–90% всех.
   /________\
```

**Правила распределения:**
- **Unit (80–90%):** чистые функции (FSM, цена, деньги, geo) + компоненты (RTL). **Быстро**, по-local.
- **Integration (10%):** «от → к → классы → вызов» (M7+M8 вместе). На sim/emu, реже.
- **e2e (5–10%):** **ключевые** сценарии (M13 на CI): регистрация, поездка-цикл, платёж. **Мало, но критичные.**

**Почему пирамида, а не «всё e2e»:** e2e **медленно** (минуты) и **хрупко** (ложные срабатывания). Unit **быстро** ловит 80% багов. Не перебивайте e2e.

### 12.2. React Native Testing Library (RTL)

```bash
npx expo install --save-dev @testing-library/react-native jest jest-expo
```

**Философия RTL:** тестируйте **то, что видит пользователь** (текст, кнопки, поведение), а не внутренности. **Queries по пользователю:**

```tsx
// src/components/Button.test.tsx
import { render, fireEvent, screen } from '@testing-library/react-native';
import { Button } from './Button';

describe('Button', () => {
  it('вызывает onPress при тапе', () => {
    const on = jest.fn();
    render(<Button title="Go" onPress={on} testID="go" />);
    fireEvent.press(screen.getByText('Go'));
    expect(on).toHaveBeenCalledTimes(1);
  });

  it('не вызывается, если disabled', () => {
    const on = jest.fn();
    render(<Button title="Go" onPress={on} disabled />);
    fireEvent.press(screen.getByText('Go'));
    expect(on).not.toHaveBeenCalled();
  });

  it('показывает loading и блокирует тап', () => {
    const on = jest.fn();
    render(<Button title="Go" onPress={on} loading />);
    // getByTestId — для элементов без текста
    expect(screen.getByTestId('btn-loading')).toBeVisible?();
  });
});
```

**Queries (запомнить):**
| Query | Когда |
|-------|-------|
| `getByTestId` | Элемент без текста (иконка, маркер) |
| `getByText` | Есть текст |
| `getByPlaceholderText` | Поле |
| `getByRole` | Семантика (button, text) |
| `queryBy*` | Ожидание отсутствия (нет элемента) |
| `findBy*` | **Асинхронное** (promise/microtask) — `await findByText(...)` |

**Имена-партнёры:** `get` (один, throw если нет), `query` (один или null), `find` (один, асинхрон), + `All` (много).

### 12.3. Mocking: API (M4), socket (M8), время

**Mock api (axios):**
```ts
// src/test/apiMock.ts
import { api } from '../shared/api/client';

export function mockApi(impl: {
  get?: (url: string) => Promise<unknown>;
  post?: (url: string, data: unknown) => Promise<unknown>;
}) {
  jest.spyOn(api, 'get').mockImplementation(impl.get as never);
  jest.spyOn(api, 'post').mockImplementation(impl.post as never);
}
```

**Mock socket (M8):**
```ts
// src/test/socketMock.ts
import * as socket from '../features/ride/socket';
const emit = jest.fn();
jest.spyOn(socket, 'getSocket').mockReturnValue({
  emit, on: jest.fn(), off: jest.fn(),
} as never);
```

**Мок времени** (таймеры: OTP 60с, reconnect):
```ts
jest.useFakeTimers();
// ...
jest.advanceTimersByTime(60_000);   // «прошло 60с»
```

**Мок навигации** (экраны берут `navigation`):
```ts
const navigation = { navigate: jest.fn(), goBack: jest.fn(), setParams: jest.fn() };
render(<MapScreen navigation={navigation} route={{ params: {} }} />);
expect(navigation.navigate).toHaveBeenCalledWith('Order', expect.anything());
```

### 12.4. Unit для «мозга» (чистые функции)

**FSM (M8.3):**
```ts
import { applyEvent, canTransition } from '../features/ride/state';

describe('FSM', () => {
  it('валидные переходы', () => {
    expect(applyEvent('CREATED', 'ACCEPTED')).toBe('ACCEPTED');
    expect(applyEvent('ACCEPTED', 'ARRIVED')).toBe('ARRIVED');
    expect(applyEvent('ARRIVED', 'STARTED')).toBe('STARTED');
    expect(applyEvent('STARTED', 'COMPLETED')).toBe('COMPLETED');
  });
  it('невалидные игнорируются', () => {
    expect(applyEvent('COMPLETED', 'CREATED')).toBe('COMPLETED');   // терминальное
    expect(applyEvent('CREATED', 'COMPLETED')).toBe('CREATED');      // пропуск
  });
  it('идемпотентность (повтор = тот же)', () => {
    expect(applyEvent('ACCEPTED', 'ACCEPTED')).toBe('ACCEPTED');
  });
});
```

**Цена (M7.7):**
```ts
import { calcPrice } from '../features/map/price';
const A = { latitude: 55.75, longitude: 37.61 };
const B = { latitude: 55.76, longitude: 37.63 };   // ~1.4 км

it('короткая поездка — минимальная цена', () => {
  const r = calcPrice(A, B, ECONOMY);
  expect(r.priceCents).toBe(ECONOMY.minPrice * 100);   // минимум
});
it('дёргает за surge', () => {
  const normal = calcPrice(A, B, ECONOMY);
  const surge = calcPrice(A, B, { ...ECONOMY, surge: 2 });
  expect(surge.priceCents).toBeGreaterThanOrEqual(normal.priceCents * 2 - 1);
});
it('целые копейки', () => {
  expect(calcPrice(A, B, ECONOMY).priceCents).toBeInteger?();   // Number.isInteger
});
```

**Деньги (M9.1):**
```ts
it('форматирует копейки', () => {
  expect(formatCents(85045)).toBe('850,45 ₽');
  expect(formatCents(1234567)).toBe('12 345,67 ₽');
  expect(formatCents(-500)).toBe('−5,00 ₽');
});
```

### 12.5. Integration: флоу «от → к → классы → вызов»

```tsx
// src/features/map/MapScreen.integration.test.tsx
// Мок: api (geo, prices), navigation, socket
it('от → к → показывает классы → CTA вызывает', async () => {
  mockApi({
    get: jest.fn(async (url) => url.includes('/geo/search') ? [FAKE_ADDR] : []),
    post: jest.fn(async () => ({ id: 'ride-1' })),
  });
  const nav = mockNav();
  render(<MapScreen navigation={nav} route={{}} />);

  // «Куда» → выбираем адрес (M7)
  fireEvent.press(screen.getByPlaceholderText('Куда'));
  await screen.findByText('Тверская');   // асинхронный поиск
  fireEvent.press(screen.getByText('Тверская'));

  // Классы появились (M7.7)
  expect(await screen.findByText(/Economy/)).toBeVisible?();

  // CTA
  fireEvent.press(screen.getByTestId('call-cta'));
  expect(nav.navigate).toHaveBeenCalledWith('Order', expect.objectContaining({ class: 'ECONOMY' }));
});
```

### 12.6. e2e: Maestro (M13)

**Maestro** — декларативные flow на **YAML**, работают на **real device/emulator/simulator**:

```yaml
# e2e/registration.yaml
---
- launchApp:
    packageName: com.yourname.ridego
- tapOn: "Телефон"
- inputText: "+79991234567"
- tapOn: "Продолжить"
- assertVisible: "Введите код"
- inputText: "123456"          # фикс OTP (M6)
- assertVisible: "RideGo"       # главный экран
```

```yaml
# e2e/ride-cycle.yaml
---
- launchApp: ...
# От → К → классы → Вызвать → (M8) → COMPLETED → Payment (4242) → Receipt → Rate
- tapOn: "Куда"
- inputText: "Тверская"
- tapOn: "Тверская"
- tapOn: "Вызвать"
# ... (ожидание статусов — M8)
- tapOn: "Оплатить"
- tapOn: "4242 4242 4242 4242"   # тестовая карта (M9)
- assertVisible: "Чек"
```

**Принципы e2e:**
- **3–5** ключевых сценариев (не все!)
- **Стабильные** селекторы: `testID` / текст (не координаты)
- **Только критичные** (M13): регистрация, поездка, платёж
- На **CI** (M13) — на эмуляторе/симуляторе

### 12.7. Coverage + mutation testing (цели M12/PROJECT)

**Coverage** (Jest):
```bash
npx jest --coverage --collectCoverageFrom='src/**/*.{ts,tsx}' --coverageThresholds?
# → % statements/branches/functions/lines
```
**Цель:** ≥ 80% (critical paths — 100%). **Critical paths = FSM, calcPrice, money, geo, idempotency** (M8/M7/M9).

**Mutation testing** (Stryker):
```bash
npx @stryker-mutator/core
# Stryker «ломает» код (меняет условия) и проверяет, что тесты ЛОВЯТ
# Score: % мутантов, убитых тестами. Цель: ≥ 80% (на critical paths)
```
**Зачем:** высокий % coverage **не значит** качество (тесты могут «пропускать»). Mutation — «а ловят ли тесты реальные баги?».

```
# .stryker.conf.json — только critical paths (быстро)
{ "testCommand": "jest", "mutate": "src/features/ride/state.ts src/features/map/price.ts src/shared/money.ts" }
```

### 12.8. Тестовые утилиты (M12-кит)

```
src/test/
├── setup.ts            # jest-expo preset, cleanup, fake timers off
├── factories.ts        # createUser, createRide, createDriver (фикс-данные)
├── apiMock.ts          # M12.3
├── socketMock.ts       # M12.3
├── navMock.ts          # M12.3
└── render.tsx          # render с providers (M5 store, M4 Query, M6 auth)
```

```tsx
// src/test/render.tsx — «всё в одном»
export function renderWithProviders(ui: ReactElement, overrides?: Partial<Providers>) {
  return render(
    <QueryClientProvider client={testQueryClient}>
      <ReduxProvider store={testStore}>   # пустой store
        <NavigationContainer>{ui}</NavigationContainer>
      </ReduxProvider>
    </QueryClientProvider>
  );
}
// testQueryClient: retry: false, gcTime: 0 (тесты быстрые, без кэша)
```

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Инфраструктура (4ч)
- [ ] `jest-expo` + RTL; `package.json`: `test`, `test:watch`, `test:coverage`
- [ ] `src/test/*` (setup, factories, mocks, render)
- [ ] Coverage: `--collectCoverageFrom`, threshold (config)

### Шаг 2. Unit: чистые функции (6ч)
- [ ] **FSM** (M8.3): все переходы + невалидные + идемпотентность
- [ ] **calcPrice** (M7.7): ≥ 8 кейсов (короткое/длинное/zero/surge/min/целые)
- [ ] **money** (M9.1): формат (≥ 5: норм/big/neg/0/целые)
- [ ] **geo** (M7): haversine (известные пары)
- [ ] **idempotency** (M9.5): ключ + повтор
- [ ] **Тесты синк-очереди** (M5.7): enqueue/dequeue/drain (success/error/empty)

### Шаг 3. Unit: компоненты (RTL) (6ч)
- [ ] **Button/TextInput/Avatar** (M2): onPress/disabled/loading; validation
- [ ] **Stars** (M9.8): тап = оценка; теги
- [ ] **DriverCard** (M4): отображение + тап; isFav
- [ ] **PhoneForm** (M6): валидация (корректный/некорректный номер)
- [ ] **OtpInput** (M6): авто-переход, paste, 6-полей

### Шаг 4. Integration (4ч)
- [ ] **MapScreen** (M12.5): от → к → классы → CTA
- [ ] **PaymentFlow** (M9.4): success/decline/3DS (mock)
- [ ] **Ride** (M8): статусы (mock socket) → UI

### Шаг 5. e2e: Maestro (2ч)
- [ ] `e2e/registration.yaml` + `e2e/ride-cycle.yaml` (M12.6)
- [ ] Прогнать на sim/emulator: `npx maestro test e2e/...`

### Шаг 6. Coverage + mutation (2ч)
- [ ] `jest --coverage` → **≥ 80%** (critical paths 100%)
- [ ] **Stryker** на critical paths → **≥ 80%** мутантов убито
- [ ] Закрыть «дыры» (добавить тесты на выживших мутантов)

### Шаг 7. Отчёт (2ч) — `docs/progress/M12.md`
- Pyramida: сколько unit/integration/e2e
- Coverage: % по типам; critical paths = 100%
- Mutation: score (critical paths)
- Что ловят тесты (примеры выживших мутантов → добавили тест)
- Ошибки: 3+

---

## ✅ Практическое задание модуля

### Задача: «Pyramida + coverage 80% + mutation 80% (critical) + e2e»

**1. Инфраструктура (4ч)** — jest-expo + RTL + test-kit (mocks, render, factories)

**2. Unit: чистые (6ч)** — FSM, calcPrice, money, geo, idempotency, syncQueue (все — 100% critical)

**3. Unit: компоненты (6ч)** — Button, Stars, DriverCard, PhoneForm, OtpInput (RTL)

**4. Integration (4ч)** — MapScreen, PaymentFlow, Ride (mock)

**5. e2e (2ч)** — Maestro: registration + ride-cycle (на sim/emulator)

**6. Coverage + mutation (2ч)** — **≥ 80%** coverage; **≥ 80%** mutation (critical)

**7. Отчёт (2ч)**

**Критерии (из PROJECT):**
- [ ] **Unit coverage ≥ 80%**, critical paths (FSM/цена/деньги/geo/idempotency) = **100%**
- [ ] **Mutation ≥ 80%** (critical paths, Stryker)
- [ ] **Integration**: 3+ флоу (Map, Payment, Ride)
- [ ] **e2e (Maestro)**: 3–5 сценариев, зелёные на sim/emulator
- [ ] `npm test` — зелёный локально (M13 — на CI)
- [ ] Тесты **ловят** баги (mutation-доказательство)

---

## 📦 Итог

У вас **дом** из тестов: unit (много, быстро) → integration (флоу) → e2e (критичные). Coverage 80% + mutation 80% (critical) = **уверенность** в рефакторинге и релизах. **M13** — этот дом на CI.

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Всё e2e | Медленно, хрупко | Пирамида (M12.1) |
| Тест internal state | Хрупко при рефакторингe | Тестируйте поведение (RTL) |
| getByTestId для всего | Хрупко | Текст/role → testID как последnеe |
|coverage 100% ≠ качество | Тесты «пропускают» | Mutation (M12.7) |
| Мок навигации/сокет/время забыли | Тесты «вешаются» | Мок (M12.3) |
| Тесты зависят от порядка | Хрупко | Изоляция (новый store/кэш) |
| sleep в e2e | Хрупко | `assertVisible` (await) |

---

## 💡 Советы на модуль

- **Пирамида — не догма, а баланс.** Unit — 80%, e2e — мало.
- **Тестируйте поведение** (что видит пользователь), не внутренности.
- **Critical paths = 100%.** FSM/цена/деньги/geo/idempotency — **здесь баг = потеря денег/пользователя**.
- **Mutation — проверка качества** тестов. Не «на всякий», а на critical.
- **e2e = ключевые.** 3–5 сценариев, не «весь app».

**[→ Модуль 13: CI/CD и релизы](../Module_13_CICD_Reloading/content.md)**
