# Модуль 10: Производительность

**[← Назад к индексу](../index.md)** | **[Модуль 9: Платежи](../Module_09_Payments/content.md)** | **Далее: [Модуль 11: Натив и платформы](../Module_11_Native_Platform/content.md) →**

---

## 🎯 Цель модуля

- **Измерять** (не угадывать): TTI, frame drops, память, bundle
- **Профилировать**: React DevTools, Android Profiler, Instruments, Screenshot Time
- **Устранять**: ре-рендеры, списки, slow startup, утечки
- **Цели модуля** (из PROJECT): frame drops < 1%, 60fps, bundle −25%, TTI < 2.5с
- **До/после** — отчёт с цифрами

**Длительность: ~22 часа. Самая «инженерная» часть курса.**

> 💡 Главный принцип: **оптимизируй то, что замерил**. Никаких «кажется, медленно». Сначала профиль, потом правка.

---

## 📚 Теория

### 10.0. Бюджеты (из PROJECT/README) и как их держать

| Метрика | Целевое | Как мерить |
|---------|---------|-----------|
| TTI (до интерактива) | < 2.5с (средний Android) | Screenshot Time / manual |
| Frame drops (карта/списки) | < 1% | Android Profiler / Reanimated |
| FPS (drag, scroll) | 60 | Android Profiler |
| Память | ≤ 2МБ рост / 20 поездок | Android Profiler / Xcode |
| JS bundle | −25% от M1-базлайна | Hermes `--dump-...` / metro |
| Crash-free | > 99.5% | Sentry (M14) |

**Базлайн:** замерьте **до** оптимизаций (M1-сборка) — и **после**. Разница — ваш результат.

### 10.1. Инструменты

**React DevTools** (перерендеры):
```bash
npx expo install react-devtools
# Remedy → react-devtools: видите дерево компонентов, props, hooks, «why did you re-render»
```
- Включите **Highlight updates** (клик по компоненту → мигает при ре-рендерe)
- Считайте: сколько раз перерендерился список при dispatch в **другой** слайс (M5)

**Android Profiler** (Android Studio / `adb`):
```bash
# FPS + frame drops:
adb shell dumpsys gfxinfo <package> reset
# ...прокручиваете список/карту...
adb shell dumpsys gfxinfo <package> | grep "Total frames"  # и "50th/90th/95th/99th percentiles"
# Память: Android Studio → Profiler → Memory (heap, allocations)
# CPU: → CPU (where the time goes)
```
- **Frame timeline**: жёлтые = missed frame (< 16.6мс). Ищите **пики**.

**Xcode Instruments** (iOS):
- **Time Profiler** (CPU), **Allocations** (память), **Core Animation FPS** (60fps)
- **Replay** сегмента → где время

**Hermes / bundle:**
```bash
# Размер JS bundle:
npx react-native bundle --platform android --dev false --bundle-output out.bundle --assets-dest out --entry-file index.js
ls -lh out.bundle   # базлайн; потом сравниваем
```

**Screenshot Time** (TTI): таймер от тапа по иконке до «полного» экрана. Методично (5 запусков, медиана).

### 10.2. Ре-рендеры: главный источник лагов

**Причина №1 лагов в RN** — лишний re-render тяжёлых поддеревьев.

**Диагностика:** React DevTools → Highlight updates. Что мигает при dispatch в `favorites`, который **не должен** перерисовывать `drivers`?

**Лекарства:**

1. **`React.memo`** — компоненты, которые не меняются (DriverCard, ClassRow):
```tsx
export const DriverCard = React.memo(function DriverCard({ driver, isFav, onToggle }) {
  // Перерисуется ТОЛЬКО если props (по значению) изменились
  ...
});
```
(⚠️ memo бесполезен, если передаёте **новый объект/функцию** каждый раз — см. 2.)

2. **Стабильные props** (коллбэки/массивы):
```tsx
// ❌ новый callback каждый рендер → memo сломан
<DriverCard onToggle={() => toggle(driver.id)} />
// ✅ useCallback (зависимости!)
const onToggle = useCallback(() => toggle(driver.id), [driver.id, toggle]);
<DriverCard onToggle={onToggle} />
```

3. **Селекторы** (M5) — `useSelector` с `createSelector` → перерендер только при реальном изменении.

4. **Разбить Context** (M2.7): один контекст на всё → мигает всё. Разделить на Read/Write, или на домены.

> 💡 **Правило:** memo + стабильные props + узкие селекторы = 80% выигрыша от re-renders.

### 10.3. Списки: FlatList → FlashList

**FlatList** хорош, но для **тяжёлых/длинных** списков (карта + списки) → **FlashList** (Shopify): пул рециклинга + предзагрузка + виртуализация.

```bash
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

// renderItem должен быть memo (M10.2) — иначе выигрыш мал
<FlashList
  data={drivers}
  dataKey="id"
  estimatedItemSize={120}          // важно: точная оценка высоты
  renderItem={({ item }) => <DriverCard ... />}   // memo
  drawDistance={5}                  // предзагрузка
  // numColumns для сеток
/>
```

**Когда FlashList:** длинные списки (100+), тяжёлые item (картинки, сложные стили), сетки. **Для 20 item** — FlatList достаточно (не усложняйте).

**Обязательные настройки FlatList/FlashList:**
- `data` — **стабильный** массив (не новый каждый рендер → `useMemo`)
- `keyExtractor` — стабильные ключи
- `getItemLayout` — если высоты одинаковые (ускоряет)
- renderItem — **memo**

### 10.4. Медленный старт (slow startup)

**Причины и решения:**

| Причина | Решение |
|---------|---------|
| Все фичи импортны сразу | **Lazy** экраны (M3.9) + `require` лениво |
| Тяжёлый JS в startup | Не импортировать тяжёлые либы в App.tsx |
| Синхронная работа в main | Вынести в useEffect / лениво |
| Большой bundle | Tree-shaking, удалить лишнее, code-splitting |

```tsx
// Ленивые экраны (M3.9) — уже есть. Проверьте, что РАБОТАЕТ:
const MapScreen = React.lazy(() => import('../features/map/MapScreen'));
// → грузится при первом показе, не при старте
```

**Bundle −25% (цель):**
1. **Baseline** (M1): замерьте размер
2. **Топ тяжёлых** импортов: `npx react-native bundle ...` + анализ (что тянет)
3. **Убрать/замениить**: большие иконки → SVG-подмножество; лишние либы → не импортировать
4. **Lazy** фич (M3.9): тяжёлые экраны не в критическом пути
5. **Итог**: повторный замер → −25%

> 💡 **Hermes** уже собран (M1). Не трогайте V8. Оптимизируем **JS**, а не движок.

### 10.5. Память: утечки

**Типичные утечки в RN:**

| Утечка | Признак | Решение |
|--------|---------|---------|
| Подписки (WS, location, netinfo) не сняты | Рост памяти при переходе экранов | **cleanup** в useEffect (M7/M8) |
| Таймеры не вычищены | Рост | clearTimeout в cleanup |
| Слушатели API/Query не отписаны | Рост | отписка / `enabled: false` |
| Карты/анимации не размонтированы | Рост | unmount → остановка (M8.7) |
| Замыкания держат большие объекты | Рост | не хренить в ref «всё» |

**Проверка:** Android Profiler → Memory → **Allocation** (heap-граф). Пройдите 20 поездок (M8-флоу) → **≤ 2МБ рост** (цель). Если растёт линейно — утечка.

**Instruments (iOS):** Allocations → **Leak** (объекты, которые не должны жить).

### 10.6. Анимации: Reanimated = 60fps (M2.8)

- Все плавные анимации — **Reanimated worklet** (UI-поток), не JS
- **Interpolate** (M2.8) — GPU, дёшево
- **Не** `Animated` (старый, JS-поток) для сложных
- **Прозрачность/масштаб** — дёшево; **layout (width/height)** — дорого (reflow)

### 10.7. Изображения (M2.1)

- **`resizeMode` + `resizeMethod: 'resize'`** (M2.1) — не держать 4К-фото в памяти под 40×40
- **Lazy-load** в списках (`Image` с placeholder, грузим при показе)
- **Кэш** (`expo-image` — кэширует, лучший выбор):
```bash
npx expo install expo-image
```
```tsx
import { Image } from 'expo-image';
<Image source={{ uri: avatarUrl }} style={{ width: 40, height: 40, borderRadius: 20 }} transition={200} />
```
(Аватары водителей/авто — `expo-image`, не `Image`.)

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 0. Базлайн (2ч)
- [ ] Измерить **до** оптимизаций: TTI (5 запусков, медиана), bundle-размер, frame drops (прокрутка «Мои поездки»), память (10 поездок)
- [ ] Записать в `docs/progress/M10-baseline.md`

### Шаг 1. Ре-рендеры (6ч)
- [ ] React DevTools → Highlight updates: найти «всё мигает»
- [ ] `React.memo` на DriverCard, ClassRow, ReceiptRow
- [ ] `useCallback` на все коллбэки-props
- [ ] Селекторы (M5) — проверить, что `favorites`-dispatch не рендерит `drivers`
- [ ] **До/после**: число перерендеров (скриншоты DevTools)

### Шаг 2. Списки (4ч)
- [ ] «Мои поездки»/«Водители» → **FlashList** (estimatedItemSize, renderItem memo)
- [ ] `data` — useMemo; keyExtractor стабильный
- [ ] **До/после**: frame drops при прокрутке (dumpsys gfxinfo)

### Шаг 3. Старт (4ч)
- [ ] Проверить **lazy** экраны (M3.9) реально работают
- [ ] Убрать тяжёлые импорты из критического пути
- [ ] **Bundle**: базлайн → −25% (указать что убрали/заменили)

### Шаг 4. Память (4ч)
- [ ] Прогнать 20 поездок (M8) → память (цель: ≤ 2МБ рост)
- [ ] Найти утечки (WS/location/таймеры) → cleanup
- [ ] **До/после**: график heap

### Шаг 5. Анимации + изображения (2ч)
- [ ] Все анимации → Reanimated worklet (60fps, M8-маркер, M2-переходы)
- [ ] Аватары → expo-image (resize, lazy, кэш)

### Шаг 6. Отчёт (2ч) — `docs/progress/M10.md`
- **Таблица до/после** по всем 6 метрикам
- Скриншоты: DevTools (ре-рендеры), gfxinfo (frame drops), heap
- Что сделали → какая метрика улучшилась
- Ошибки: 3+

---

## ✅ Практическое задание модуля

### Задача: «До/после по всем метрикам + bundle −25%»

**1. Базлайн (2ч)** — все 6 метрик «до»

**2. Ре-рендеры (6ч)**
- [ ] memo + useCallback + селекторы
- [ ] **До/после** перерендеров (DevTools)

**3. Списки (4ч)** — FlashList; **до/после** frame drops

**4. Старт (4ч)** — lazy + bundle; **bundle −25%** (доказано)

**5. Память (4ч)** — 20 поездок; **≤ 2МБ рост** (доказано)

**6. Анимации+изображения (2ч)** — Reanimated worklet; expo-image

**7. Отчёт (2ч)**

**Критерии (из PROJECT):**
- [ ] TTI < 2.5с (средний Android) — замерено
- [ ] Frame drops < 1% (карта + списки) — замерено
- [ ] 60fps (drag, scroll) — замерено
- [ ] Память ≤ 2МБ / 20 поездок — замерено
- [ ] **Bundle −25%** от M1 — замерено
- [ ] Каждое улучшение — **до/после** с цифрами (не «стало быстрее»)

---

## 📦 Итог

Код не сильно меняется (memo, FlashList, lazy, cleanup, expo-image). **Меняется методология:** вы теперь **измеряете, профилируете, оптимизируете по цифрам**. Это навык, который отличает мидла от джуна.

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Оптимизация «на глаз» | Тратите время на то, что не болит | Профиль → цифры → правка |
| memo с новыми props | Бесполезно | Стабильные props (useCallback/useMemo) |
| Новый массив в data | FlatList переотрисовывает | useMemo / стабильный источник |
| Утечка подписок | Рост памяти, лаги | cleanup в useEffect |
| 4К-фото под 40px | Память, lag | resize / expo-image |
| Анимация в JS-потоке | Drop при сложной | Reanimated worklet |
| FlashList для 5 item | Излишне | FlatList (не усложняйте) |
| Right-то что? | — | Всегда до/после, всегда цифры |

---

## 💡 Советы на модуль

- **Цифры > мнения.** «Стало быстрее» — нет. «Frame drops 8% → 0.5%» — да.
- **Базлайн — обязательно.** Без «до» нет «после».
- **80/20:** memo + стабильные props + FlashList + lazy + cleanup закрывают 80%.
- **Не усложняйте:** 20 item — FlatList; длинный/тяжёлый — FlashList.
- **Память — проверка на дистанции.** 20 поездок, а не 2.

**[→ Модуль 11: Натив и платформы](../Module_11_Native_Platform/content.md)**
