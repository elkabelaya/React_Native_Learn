# 📘 Модуль 2: UI, вёрстка, темизация, анимации

**[← Назад к индексу](../index.md)** | **Далее: [Модуль 3](../Module_03_Navigation/content.md) →**

Добро пожаловать во второй модуль! К этому моменту у вас есть проект и dev build. Теперь мы построим **дизайн-систему** — фундамент всего UI, — и научимся делать **плавные анимации** на 60 fps. Вы получите 10+ переиспользуемых компонентов и экран онбординга.

**Цели модуля:**
1. Управлять базовыми компонентами RN и Flexbox-вёрсткой
2. Построить дизайн-систему: токены, компоненты (Button, Input, Card, Badge, Avatar, Skeleton, BottomSheet)
3. Сделать тёмную тему (системную + переключатель) на токенах
4. Понять Reanimated (worklets, UI thread) и сделать 60 fps-анимации
5. Собрать экран онбординга со свайпом

**Время выполнения:** ~15 часов.

---

## 1. Теория: UI-примитивы React Native

### Основные компоненты

| Компонент | Аналог в вебе | Особенности в RN |
|-----------|---------------|------------------|
| `View` | `div` | Только container; flexbox |
| `Text` | `span`/`p` | **Нельзя вкладывать View в Text** (только Text в Text) |
| `Image` | `img` | `source={require()}` или `{uri}`; `resizeMode` |
| `TextInput` | `input` | `value` + `onChangeText` (контролируемый); `placeholderTextColor` |
| `ScrollView` | `overflow: auto` | Одна вложенная прокрутка; тяжёлый для больших списков |
| `FlatList` | виртуальный список | **Виртуализация** — рендерит видимые элементы |
| `Pressable` | `button` | `onPressIn/Out`, `hitSlop`, `accessibility*` |
| `Switch`, `ActivityIndicator` | — | Нативные контролы |

> ⚠️ **Важные отличия от веба:**
> - Нет DOM, нет CSS — только `StyleSheet.create({})` (объекты)
> - `position: fixed` → `absolute`; `flex-direction` по умолчанию `column`
> - Нет `:hover`, `:focus` — эмулируйте через state (onPressIn)
> - **Зона тапа ≠ видимый элемент** — добавляйте `hitSlop`

### Flexbox: главное отличие

```
CSS по умолчанию:    flex-direction: row     (горизонталь)
RN по умолчанию:     flex-direction: column  (вертикаль)

flex: 1 = «займи весь оставшийся участок» — главный инструмент вёрстки
```

### Безопасные зоны (safe area)

Устройства с вырезом (notch, Dynamic Island) — UI «под вырезом» будет обрезан. Решение: `react-native-safe-area-context` (подключён в модуле 1).

```tsx
import { SafeAreaView, useSafeAreaInsets } from 'react-native-safe-area-context';

<SafeAreaView edges={['top']}>
  {/* верхняя часть — под notch */}
</SafeAreaView>

const { bottom } = useSafeAreaInsets();  // нижняя — для bottom bar
```

### Клавиатура

```tsx
import { KeyboardAvoidingView, Platform, Keyboard } from 'react-native';

<KeyboardAvoidingView
  style={{ flex: 1 }}
  behavior={Platform.OS === 'ios' ? 'padding' : undefined}
  keyboardVerticalOffset={100}   // под вашу шапку
>
  <TextInput autoFocus />
</KeyboardAvoidingView>

Keyboard.dismiss();   // программно
```

> 💡 **Почему iOS/Android по-разному?** iOS — клавиатура «наезжает» на UI (padding решает). Android — перерисовывает (adjustResize) — часто ничего не нужно.

---

## 2. Теория: тёмная тема на токенах

**Принцип:** в компонентах **никогда** не используются хардкод-цвета. Только токены + тема.

```
tokens (статика: spacing, radii, typography)
   +
palette (светлая и тёмная палитры)
   ↓
Theme = { colors: { bg, surface, text, ... }, spacing, radii, typography }
   ↓
ThemeProvider (Context) → useTheme()
   ↓
Компоненты:  const t = useTheme(); <View style={{ backgroundColor: t.colors.surface }} />
```

### Реализация

```bash
npx expo install expo-theme-detector   # системная тема
npm i @react-native-clone-without-props  # (опционально) — для наследования
```

```tsx
// src/theme/themes.ts
import { colors } from './tokens';

export const lightTheme = {
  dark: false,
  colors: {
    bg: colors.surface.bg,
    surface: colors.surface.card,
    surfaceAlt: '#F3F4F6',
    border: colors.surface.border,
    textPrimary: colors.text.primary,
    textSecondary: colors.text.secondary,
    brand: colors.brand.primary,
    accent: colors.brand.accent,
    onAccent: '#1F2937',
  },
};

export const darkTheme = {
  dark: true,
  colors: {
    bg: '#0B1120',
    surface: '#111827',
    surfaceAlt: '#1F2937',
    border: '#374151',
    textPrimary: '#F9FAFB',
    textSecondary: '#9CA3AF',
    brand: '#818CF8',
    accent: '#FBBF24',
    onAccent: '#1F2937',
  },
};

export type Theme = typeof lightTheme;
```

```tsx
// src/theme/ThemeProvider.tsx
import { createContext, useContext, useEffect, useState, useCallback, ReactNode } from 'react';
import { useColorScheme } from 'react-native';
import { MMKV } from 'react-native-mmkv';
import { lightTheme, darkTheme, Theme } from './themes';

const storage = new MMKV();

const ThemeCtx = createContext<Theme>(lightTheme);

type ThemeMode = 'system' | 'light' | 'dark';

export function ThemeProvider({ children }: { children: ReactNode }) {
  const systemDark = useColorScheme() === 'dark';
  const [mode, setMode] = useState<ThemeMode>(
    () => (storage.getString('themeMode') as ThemeMode | undefined) ?? 'system'
  );

  const setModePersisted = useCallback((m: ThemeMode) => {
    setMode(m);
    storage.set('themeMode', m);
  }, []);

  const theme = mode === 'system'
    ? (systemDark ? darkTheme : lightTheme)
    : (mode === 'dark' ? darkTheme : lightTheme);

  return <ThemeCtx.Provider value={theme}>{children}</ThemeCtx.Provider>;
}

export const useTheme = () => useContext(ThemeCtx);
```

> ⚠️ **MMKV** устанавливаем уже здесь (быстрое синхронное хранилище): `npx expo install react-native-mmkv`. В модуле 5 пойдёт глубже.

**Подключение в App.tsx:**
```tsx
<ThemeProvider>
  <SafeAreaProvider>
    <StatusBar style={theme.dark ? 'light' : 'dark'} />
    ...
  </SafeAreaProvider>
</ThemeProvider>
```

> 💡 **Почему не `useColorScheme()` напрямую в каждом компоненте?** Один источник правды (провайдер) → переключение темы = один ре-рендер дерева. Плюс — ручной переключатель (system/light/dark) бесплатно.

---

## 3. Теория: Reanimated — анимации на 60 fps

### Почему `Animated` из RN — не всегда

```
Animated (из коробки):  анимации считаются в JS thread
  → JS занят (логика, сеть) = анимация проседает
  → 60 fps нестабильны

Reanimated:  анимации исполняются в UI thread (worklets)
  → JS thread свободна = анимации ВСЕГДА 60 fps
  → даже при тяжёлой логике
```

**Worklet** — специальная функция, которая «переносится» в UI thread. В ней — только «допустимые» операции (арифметика, чтение shared values). Никаких `fetch`, `console`, `setState`.

```tsx
import Animated, { useSharedValue, useAnimatedStyle, withTiming, withSpring } from 'react-native-reanimated';

const progress = useSharedValue(0);

// Триггер (в JS thread):
progress.value = withTiming(100, { duration: 1000 });
// или пружина:
progress.value = withSpring(100, { damping: 15, stiffness: 100 });

// Стиль (в UI thread — worklet):
const style = useAnimatedStyle(() => ({
  width: `${progress.value}%`,
}));

<Animated.View style={style} />
```

### Пресеты (90% задач)

```tsx
import { FadeIn, FadeOut, SlideInUp, SlideInDown, BounceIn } from 'react-native-reanimated';

<Animated.View entering={FadeIn.duration(250)}>...</Animated.View>
<Animated.View exiting={FadeOut.duration(150)}>...</Animated.View>
<Animated.View entering={SlideInUp.springify(damping: 20)}>...</Animated.View>
```

> ⚠️ Для `exiting` нужен родитель с **condition** (условным рендером) — иначе не сработает.

### Конфигурация (модуль 1: babel)

```js
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: ['react-native-reanimated/plugin'],   // ОБЯЗАТЕЛЬНО: последний в списке
  };
};
```

```bash
npx expo install react-native-reanimated
# После: npx expo prebuild --clean (нативный модуль)
```

---

## 4. Практика: дизайн-система RideGo (src/ui/)

Структура (создайте все):

```
src/ui/
├── Button/
│   ├── Button.tsx
│   └── index.ts
├── Card/
├── Badge/
├── Input/
├── Avatar/
├── Spinner/
├── Skeleton/
├── BottomSheet/
└── index.ts      # barrel: export * from './Button' и т.д.
```

### 4.1 Button (флагман системы)

```tsx
// src/ui/Button/Button.tsx
import { ReactNode, useCallback } from 'react';
import { Pressable, Text, StyleSheet, ActivityIndicator, StyleProp, ViewStyle } from 'react-native';
import { useTheme } from '@/theme/ThemeProvider';
import { spacing, radii, typography } from '@/theme/tokens';

type Variant = 'primary' | 'secondary' | 'accent' | 'danger' | 'ghost';
type Size = 'sm' | 'md' | 'lg';

export type ButtonProps = {
  title: string;
  onPress: () => void;
  variant?: Variant;
  size?: Size;
  disabled?: boolean;
  loading?: boolean;
  icon?: ReactNode;
  style?: StyleProp<ViewStyle>;
  accessibilityLabel?: string;
};

const sizeMap: Record<Size, { h: number; px: number; fs: number }> = {
  sm: { h: 36, px: 16, fs: 13 },
  md: { h: 48, px: 24, fs: 15 },
  lg: { h: 56, px: 32, fs: 17 },
};

export function Button({
  title, onPress, variant = 'primary', size = 'md',
  disabled, loading, icon, style, accessibilityLabel,
}: ButtonProps) {
  const t = useTheme();
  const s = sizeMap[size];

  const bg = {
    primary: t.colors.brand,
    secondary: t.colors.surfaceAlt,
    accent: t.colors.accent,
    danger: '#EF4444',
    ghost: 'transparent',
  }[variant];
  const fg = variant === 'accent' ? t.colors.onAccent
    : variant === 'ghost' ? t.colors.brand
    : variant === 'secondary' ? t.colors.textPrimary
    : '#FFFFFF';

  const isDisabled = disabled || loading;

  const Press = (
    <Pressable
      onPress={onPress}
      disabled={isDisabled}
      accessibilityLabel={accessibilityLabel ?? title}
      accessibilityRole="button"
      accessibilityState={{ disabled: !!isDisabled }}
      style={({ pressed }) => [
        styles.base,
        { height: s.h, paddingHorizontal: s.px, backgroundColor: bg, opacity: isDisabled ? 0.5 : pressed ? 0.85 : 1 },
        style,
      ]}
    >
      {loading ? <ActivityIndicator color={fg} /> : (
        <>{icon && <Text style={{ marginRight: 8 }}>{icon}</Text>}<Text style={{ color: fg, fontSize: s.fs, fontWeight: '600' }}>{title}</Text></>
      )}
    </Pressable>
  );

  // Плавное нажатие (60 fps, UI thread)
  return <Animated.View style={[styles.hit, pressed && { opacity: 0.9 }]}>{Press}</Animated.View>;
}

const styles = StyleSheet.create({
  base: { borderRadius: radii.pill, alignItems: 'center', justifyContent: 'center', flexDirection: 'row' },
  hit: { /* hitSlop встроен в Pressable */ },
});
```

> 💡 **`Animated` здесь опционален** — в «чистом» варианте достаточно Pressable. Reanimated нужен там, где **длительные** анимации (slide, spring), а не «подсветка».

### 4.2 Input (плавающий лейбл + ошибка)

```tsx
// src/ui/Input/Input.tsx
import { useState, useCallback } from 'react';
import { View, Text, TextInput as RNTextInput, StyleSheet, TextInputProps, ViewStyle, StyleProp } from 'react-native';
import Animated, { useAnimatedStyle, useSharedValue, withTiming } from 'react-native-reanimated';
import { useTheme } from '@/theme/ThemeProvider';
import { spacing, radii, typography } from '@/theme/tokens';

type InputProps = Omit<RNTextInputProps, 'style'> & {
  label: string;
  error?: string;
  containerStyle?: StyleProp<ViewStyle>;
};

export function Input({ label, error, style, containerStyle, ...rest }: InputProps) {
  const t = useTheme();
  const [focused, setFocused] = useState(false);
  const [value, setValue] = useState('');

  // Плавающий лейбл: 60 fps
  const labelY = useSharedValue(0);
  const labelScale = useSharedValue(1);
  const focusedRef = useSharedValue(false);

  const handleFocus = useCallback(() => {
    setFocused(true);
    focusedRef.value = true;
    labelY.value = withTiming(-18, { duration: 150 });
    labelScale.value = withTiming(0.75, { duration: 150 });
  }, []);

  const handleBlur = useCallback(() => {
    setFocused(false);
    if (!value) {
      focusedRef.value = false;
      labelY.value = withTiming(0, { duration: 150 });
      labelScale.value = withTiming(1, { duration: 150 });
    }
  }, [value]);

  const floatLabel = useAnimatedStyle(() => ({
    transform: [{ translateY: labelY.value }, { scale: labelScale.value }],
  }));
  const floatBorder = useAnimatedStyle(() => ({
    borderColor: focusedRef.value ? t.colors.brand : error ? '#EF4444' : t.colors.border,
    borderWidth: focusedRef.value ? 2 : 1,
  }));

  return (
    <View style={[styles.container, containerStyle]}>
      <Animated.View style={[styles.field, floatBorder, { backgroundColor: t.colors.surface }]}>
        <RNTextInput
          style={[styles.input, { color: t.colors.textPrimary, fontSize: typography.body.fontSize }, style]}
          placeholder={label}
          placeholderTextColor={t.colors.textSecondary}
          value={value}
          onChangeText={setValue}
          onFocus={handleFocus}
          onBlur={handleBlur}
          cursorColor={t.colors.brand}
          {...rest}
        />
      </Animated.View>
      <Animated.Text style={[styles.label, floatLabel, { color: focused ? t.colors.brand : t.colors.textSecondary, top: 8, left: 14 }]}>
        {label}
      </Animated.Text>
      {error && <Text style={{ color: '#EF4444', fontSize: 12, marginTop: 4 }}>{error}</Text>}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { gap: 4 },
  field: { borderRadius: radii.md, height: 54, justifyContent: 'center', overflow: 'hidden' },
  input: { paddingHorizontal: 14, flex: 1 },
  label: { position: 'absolute', backgroundColor: 'transparent' },
});
```

> 💡 Это самый «живой» компонент модуля: плавающий лейбл через Reanimated. **Проверьте на реальном Android** — именно там видны просадки.

### 4.3 Остальные компоненты (кратко — реализуем в домашке)

| Компонент | Суть | Ключевые props |
|-----------|------|----------------|
| **Card** | Контейнер с тенью и radius | `elevation`, `padding`, `shadow` (iOS/Android по-разному!) |
| **Badge** | Маленькая плашка | `variant: success/warning/error/brand`, `size` |
| **Avatar** | Круглая иконка | `size`, `source`, `fallback` (инициалы) |
| **Spinner** | Обёртка ActivityIndicator | `size`, `color` |
| **Skeleton** | Пульсирующий placeholder | `width`, `height`, `radius` (Reanimated pulse) |
| **BottomSheet** | Всплывающая панель снизу | `visible`, `onClose`, `snapPoints` (Reanimated gesture) |

**Тень — классическая ловушка платформ:**
```tsx
const cardShadow = {
  // iOS:
  shadowColor: '#000',
  shadowOpacity: 0.08,
  shadowRadius: 12,
  shadowOffset: { width: 0, height: 4 },
  // Android:
  elevation: 4,
};
```

**BottomSheet (скелет, Reanimated gesture):**
```tsx
import Animated, {
  useSharedValue, useAnimatedStyle, withTiming, runOnUI, useAnimatedGestureHandler,
} from 'react-native-reanimated';
import { PanGestureHandler } from 'react-native-gesture-handler';

const translateY = useSharedValue(0);  // 0 = открыт, 1 = закрыт (в долях)

const gestureHandler = useAnimatedGestureHandler({
  onActive: (e) => {
    translateY.value = e.translationY / 300;   // нормализуем
  },
  onEnd: (e) => {
    const shouldClose = e.velocityY > 300 || e.translationY > 150;
    translateY.value = withTiming(shouldClose ? 1 : 0, { duration: 200 });
    if (shouldClose) runOnJS(onClose)();      // JS-колбэк
  },
});
```

> ⚠️ `react-native-gesture-handler` — нативный модуль (требует dev build). `npx expo install react-native-gesture-handler`. В корне: `<GestureHandlerRootView>`.

### 4.4 Barrel + barrel-экспорт

```ts
// src/ui/index.ts
export * from './Button';
export * from './Card';
export * from './Badge';
export * from './Input';
export * from './Avatar';
export * from './Spinner';
export * from './Skeleton';
export * from './BottomSheet';
```

> 💡 **Почему barrel?** `import { Button, Card } from '@/ui'` — чисто. Но **не перебарщивайте**: barrel = весь модуль в бандл (tree-shaking в Metro — частичный). Для design system — ок.

---

## 5. Практика: экран онбординга (src/features/onboarding/)

**Три экрана со свайпом** (Reanimated + gesture-handler):

```
src/features/onboarding/
├── Onboarding.tsx           # контейнер: 3 pages, dots, CTA
├── pages/
│   ├── Page1.tsx            # «Поездки в один тап»
│   ├── Page2.tsx            # «Живой трекинг»
│   └── Page3.tsx            # «Оплата картой»
├── OnboardingPage.tsx       # generic: иконка, заголовок, текст
└── index.ts
```

### Логика (скелет)

```tsx
import { useState, useRef, useEffect } from 'react';
import { View, Text, StyleSheet, Pressable, useWindowDimensions } from 'react-native';
import { Swipeable } from 'react-native-gesture-handler';
import Animated, { FadeIn, Extrapolate, interpolate, useAnimatedStyle, useSharedValue, withSpring } from 'react-native-reanimated';
import { Button, Card } from '@/ui';
import { useTheme } from '@/theme/ThemeProvider';
import { spacing, radii, typography } from '@/theme/tokens';

const PAGES = [
  { emoji: '🗺️', title: 'Куда угодно', text: 'Укажите точку — построим маршрут и покажем цену заранее.' },
  { emoji: '📡', title: 'Живой трекинг', text: 'Следите за водителем в реальном времени прямо на карте.' },
  { emoji: '💳', title: 'Оплата без боли', text: 'Карта в приложении — чек сразу после поездки.' },
];

export function Onboarding({ onDone }: { onDone: () => void }) {
  const t = useTheme();
  const { width } = useWindowDimensions();
  const [page, setPage] = useState(0);
  const translate = useSharedValue(0);
  const [lastSwipe, setLastSwipe] = useState(0);

  const onSwipe = (direction: -1 | 1) => {
    const next = Math.min(Math.max(page + direction, 0), PAGES.length - 1);
    setLastSwipe(direction);
    translate.value = withSpring(-next * width, { damping: 20 });
    setPage(next);
  };

  const rowStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translate.value }],
  }));

  return (
    <View style={[styles.root, { backgroundColor: t.colors.bg }]}>
      <Animated.View style={[styles.row, rowStyle]}>
        {PAGES.map((p, i) => (
          <OnboardingPage key={i} {...p} width={width} last={i === PAGES.length - 1}
            onSkip={onDone} onCta={() => (i < PAGES.length - 1 ? onSwipe(1) : onDone())}
            onBack={i > 0 ? () => onSwipe(-1) : undefined}
          />
        ))}
      </Animated.View>

      {/* Точки-индикаторы */}
      <View style={styles.dots}>
        {PAGES.map((_, i) => (
          <View key={i} style={[styles.dot, i === page && styles.dotActive, { backgroundColor: t.colors.brand }]} />
        ))}
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  root: { flex: 1 },
  row: { flexDirection: 'row', flex: 1 },
  dots: { position: 'absolute', bottom: 120, alignSelf: 'center', flexDirection: 'row', gap: 8 },
  dot: { width: 8, height: 8, borderRadius: 4, opacity: 0.3 },
  dotActive: { opacity: 1, width: 20, borderRadius: 4 },
});
```

> ⚠️ **`useWindowDimensions()`** — ширина экрана. **`interpolate()`** — для плавных дот-индикаторов (расширяется с page).

**Онбординг показывается один раз** (флаг в MMKV):
```tsx
const storage = new MMKV();
const seen = storage.getBoolean('onboarding.seen', false);
// в onDone: storage.set('onboarding.seen', true)
```

### Подключение

```tsx
// App.tsx
{seen ? <HomeScreen /> : <Onboarding onDone={...} />}
```

---

## 📝 Домашнее задание (Модуль 2)

### Обязательные задачи

1. **Доработать design system (все 8 компонентов)**
   - Button: все 5 variant × 3 size + loading + disabled + icon + `accessibility*` + `hitSlop`
   - Input: плавающий лейбл (Reanimated) + ошибка + password (show/hide) + suffix (иконка)
   - Card: тень (iOS shadow + Android elevation) + `elevation` prop
   - Badge: 4 variant, с точкой/иконкой
   - Avatar: `size` (sm/md/lg), fallback на инициалы, кольцо (online)
   - Skeleton: pulse через Reanimated (opacity loop)
   - Spinner: обёртка (size, color из темы)
   - BottomSheet: snap-points (0.5, 1), свайп-закрытие, dim-фон, `KeyboardAvoiding`

2. **Тёмная тема (полный цикл)**
   - Toggle в HomeScreen: три состояния (system/light/dark) с пиктограммами
   - При смене: **плавная** анимация переходов цветов (Reanimated `withTiming` на shared color) — бонус, если сложно — мгновенно
   - Проверить: переключение **системной** темы (Settings телефона) → приложение реагирует
   - Все 8 компонентов должны корректно работать в обеих темах

3. **Онбординг (полностью)**
   - 3 страницы со свайпом (в обе стороны)
   - Доты: активный — расширен (interpolate)
   - Кнопки: «Пропустить» (первая), «Назад»/«Далее» (средние), «Начать» (последняя)
   - Отобрается один раз (флаг MMKV)
   - **60 fps**: перетащите на реальном Android — без джанков

4. **Доступность (обязательно для middle)**
   - Во всех интерактивных компонентах: `accessibilityLabel`, `accessibilityRole`, `accessibilityState`
   - `hitSlop` на иконках-кнопках (минимум 44×44pt)
   - Контраст: проверьте текст на background (WCAG AA: ≥ 4.5:1)

5. **TypeScript + Lint**
   - Все props типизированы (`type`, не `any`)
   - `npm run typecheck` — 0 ошибок, `npm run lint` — 0 ошибок
   - Компоненты принимают `style?: StyleProp<ViewStyle>` (композиция)

### Дополнительные задачи (по желанию)

6. **Иконки:** подключите `@expo/vector-icons` (Ionicons), замените эмодзи онбординга на иконки
7. **Шрифты:** `expo-font` + Google Font (Inter) → токены typography
8. **`useAnimatedProps`** для плавного изменения цвета кнопок
9. **Storybook-экран**: отдельный `DevMenu` — все компоненты в сетке (как «витрина»)
10. **`Animated` vs `Reanimated` бенчмарк**: анимируйте `translateX` тяжёлой задачей в фоне (setTimeout loop) — покажите, что Reanimated не проседает

### Критерии сдачи

- [ ] 8 компонентов, каждый: все states, доступность, тема
- [ ] Button: `loading` блокирует тапы, `disabled` — серый, `hitSlop` работает
- [ ] Input: лейбл плавно всплывает (на двух платформах)
- [ ] BottomSheet: 2 snap-points, свайп, клавиатура не перекрывает
- [ ] Тёмная тема: системная + ручная, без «миганий»
- [ ] Онбординг: свайп 60 fps (реальный Android), shown один раз
- [ ] Typecheck + Lint — зелёные
- [ ] PR: `feat: design system + onboarding + dark theme`

### Как проверить

1. **Dev Menu** (Storybook): откройте все компоненты в обеих темах
2. **Онбординг:** 3× свайп, затем «Начать» → HomeScreen. Перезапустите приложение — онбординга нет
3. **60 fps:** включите **FPS-счётчик** (Android: `adb shell service call SurfaceFlinger 1008 i32 2` или React DevTools profiler) — при свайпе ≥ 55 fps
4. **Доступность:** включите **VoiceOver (iOS)** / **TalkBack (Android)** — прочитайте экран вслух: лейблы осмысленны

---

## 💡 Полезные команды (модуль 2)

```bash
# Установка
npx expo install react-native-reanimated react-native-gesture-handler react-native-mmkv expo-font @expo/vector-icons
# После нативных: npx expo prebuild --clean

# Отладка анимаций
# React DevTools → Profiler → найдите компонент → смотрите re-renders
# Android FPS: adb shell dumpsys gfxinfo com.yourname.ridego
# iOS FPS: Xcode → Debug → View Debugging → Rendering
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| Reanimated: `Cannot read property ... of undefined` | Работа с JS-объектом в worklet | В worklet — только shared values, арифметика. fetch/setState — через `runOnJS` |
| Animation не стартует | `entering/exiting` без condition | Убедитесь: компонент действительно монтируется/размонтируется (условный рендер) |
| Бabel-ошибка: `You must add the reanimated plugin` | Нет/не тот плагин | `babel.config.js`: `plugins: ['react-native-reanimated/plugin']` — **последним** |
| Тень «ломает» Android | `shadow*` — только iOS | Для Android — `elevation` (и наоборот) |
| `SafeAreaView` от RN «съедает» top на Android | Устаревший компонент | Всегда `react-native-safe-area-context` |
| Плавающий лейбл «прыгает» | `withTiming` на каждый render | Управляйте через `useSharedValue` + один триггер (focus/blur) |
| BottomSheet под клавиатурой | Нет KeyboardAvoiding | Оберните в `KeyboardAvoidingView` (iOS `behavior="padding"`) |
| Две платформы выглядят по-разному | Разные default (шрифты, тени) | Норм! Проектируйте токены под обе, тестируйте обе |

---

## 💡 Советы на модуль

- **Токены — закон.** Если видите хардкод-цвет в компоненте — это баг. Middle не хардкодит.
- **`StyleSheet.create`** мемоизирует стили — не создавайте объекты стилей внутри рендера (кроме динамических `style={[...]}`).
- **Два устройства рядом.** Тени, шрифты, отступы клавиатуры — по-разному. Тестируйте обе.
- **Анимация = физика.** `withSpring` (дamping, stiffness) — «реалистично», `withTiming` — «механично». Для UI — пружины.
- **Worklet-граница.** Не понимаете, можно ли конструкцию в worklet — смотрите [docs: Worklets](https://docs.swmansion.com/react-native-reanimated/docs/guides/limitations-in-worklets).

**[→ Модуль 3: Навигация](../Module_03_Navigation/content.md)**
