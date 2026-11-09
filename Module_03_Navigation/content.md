# 📘 Модуль 3: Навигация

**[← Назад к индексу](../index.md)** | **Далее: [Модуль 4](../Module_04_Data_API_Lists/content.md) →**

В третьем модуле мы построим **каркас приложения**: навигационную модель RideGo с вкладками, стеками, типизацией маршрутов, deep links и авторизационным гвардом. Это скелет, на который в следующих модулях «наденут» данные, состояние и auth.

**Цели модуля:**
1. Понять архитектуру React Navigation (v7): native-stack, bottom-tabs, как они устроены
2. Построить **типизированную** навигацию (param lists, typed hooks)
3. Реализовать весь граф экранов RideGo: вкладки → стеки → модалки
4. Сделать **deep links** (URI scheme) — из push, из текста, из QR
5. Настроить **auth guard** (stub — полный цикл в модуле 6)

**Время выполнения:** ~15 часов.

---

## 1. Теория: React Navigation v7

### Экосистема

```
@react-navigation/native        # ядро: NavigationContainer, context, deep links
@react-navigation/native-stack  # нативные стеки (UI на нативном уровне — быстро)
@react-navigation/bottom-tabs   # нижние вкладки (нативный tab bar)
react-native-screens            # нативные экраны (обязательная зависимость)
react-native-safe-area-context  # уже подключён (модуль 2)
```

```bash
npx expo install @react-navigation/native @react-navigation/native-stack @react-navigation/bottom-tabs react-native-screens react-native-safe-area-context
```

### Почему native-stack, а не JS-стеки?

| | JS stack (`@react-navigation/stack`) | **Native stack** |
|---|---|---|
| Переходы | React-рендер | Нативные (Android/iOS) |
| Swipe-back (iOS) | Эмуляция | Настоящий |
| Анимации | Reanimated | Нативные |
| Память | React-компоненты в памяти | Нативные view |
| Скорость | Медленнее | ⚡ Быстрее |

> **Вывод:** для 95% приложений — `native-stack`. JS-stack — когда нужны очень кастомные анимации переходов.

### Типизация маршрутов (критично для middle!)

**Проблема без типов:** `navigation.navigate('Order', { id: 123 })` — опечатка `id` → `undefined` в другом экране, TS не поймает.

**Решение — param list + глобальный `declare`:**

```ts
// src/navigation/types.ts
export type RootStackParamList = {
  Root: NavigatorScreenParams<MainTabsParamList> | undefined;  # ← "обёртка"
  Auth: undefined;
};

export type AuthStackParamList = {
  Register: undefined;
  Login: undefined;
  Otp: { phone: string };
};

export type MainTabsParamList = {
  Map: NavigatorScreenParams<MapStackParamList> | undefined;
  Orders: undefined;
  Profile: undefined;
};

export type MapStackParamList = {
  Map: undefined;
  AddressFrom: undefined;
  AddressTo: { from: Address };
  ClassChoice: { from: Address; to: Address };
  Order: { rideId: string; from: Address; to: Address };
};

// Глобальная типизация → useNavigation() «знает» все экраны
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

> 💡 `NavigatorScreenParams<X>` — тип для «вложенного» навигатора. Без него `navigate('Map', { screen: 'AddressTo' })` не типизировалось бы.

**Использование:**
```tsx
const navigation = useNavigation<NativeStackNavigationProp<MapStackParamList>>();
navigation.navigate('ClassChoice', { from, to });   // ← типовое, с проверкой
```

---

## 2. Теория: вложенность и граф

```
RootStack
├── Auth (если не authed)  ── AuthStack
│     Register → Login → Otp
└── Main (если authed)      ── BottomTabs
      ├── Map (tab)         ── MapStack
      │     Map → AddressFrom → AddressTo → ClassChoice → Order (modal)
      ├── Orders (tab)      ── OrdersStack
      │     OrdersList → RideDetail
      └── Profile (tab)
            Profile → Settings → (biometrics, theme, about)
```

**Два уровня вложенности** (stack в tab в stack) — типичная «сервисная» навигация. Разберём по частям.

> 💡 **Почему Order — модалка?** Пользователь видит карту, вызывает → выезжает bottom sheet/модалка с деталями. При отмене — карта остаётся. Это UX, а не «техническая» модалка.

---

## 3. Практика: RootStack + Auth Guard (stub)

```tsx
// src/navigation/RootNavigator.tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { ActivityIndicator, View, StyleSheet } from 'react-native';
import { MainTabs } from './MainTabs';
import { AuthStack } from './AuthStack';
import { useAuth } from '@/features/auth/useAuth';   // stub в модуле 3
import { colors } from '@/theme/tokens';

const Stack = createNativeStackNavigator<RootStackParamList>();

export function RootNavigator() {
  const { user, isInitialized, isAuthed } = useAuth();

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isInitialized ? (
        isAuthed ? (
          <Stack.Screen name="Root" component={MainTabs} />
        ) : (
          <Stack.Screen name="Auth" component={AuthStack} />
        )
      ) : (
        <Stack.Screen
          name="Root"
          // Перезагрузка: показываем спиннер, пока auth «решается»
          component={() => (
            <View style={styles.loading}>
              <ActivityIndicator size="large" color={colors.brand.primary} />
            </View>
          )}
        />
      )}
    </Stack.Navigator>
  );
}
```

> ⚠️ **`isInitialized`** — ключевой паттерн: auth проверяет сохранённые токены (Keychain, MMKV). Пока не решил — показываем загрузку, **не мигание** Auth→Main→Auth.

### Auth guard (stub для модуля 3, полный — в модуле 6)

```ts
// src/features/auth/useAuth.ts (stub)
import { useState } from 'react';

export function useAuth() {
  // STUB: в модуле 6 станет настоящим (Keychain, refresh)
  const [isInitialized] = useState(true);
  const [user] = useState(null);   // null = не authed → покажем AuthStack
  return { user, isInitialized, isAuthed: !!user, login: async () => {}, logout: async () => {} };
}
```

> 💡 **Зачем stub?** Чтобы навигация была готова, а модуль 6 только **заменит** реализацию — не трогая навигацию. Это «швы», которые мы проектируем с первого дня.

---

## 4. Практика: MainTabs (нижние вкладки)

```tsx
// src/navigation/MainTabs.tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { Text } from 'react-native';
import { MapStack } from './MapStack';
import { OrdersStack } from './OrdersStack';
import { ProfileStack } from './ProfileStack';
import { useTheme } from '@/theme/ThemeProvider';
import { MainTabsParamList } from './types';

const Tabs = createBottomTabNavigator<MainTabsParamList>();

export function MainTabs() {
  const t = useTheme();
  return (
    <Tabs.Navigator
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: t.colors.brand,
        tabBarInactiveTintColor: t.colors.textSecondary,
        tabBarStyle: {
          backgroundColor: t.colors.surface,
          borderTopColor: t.colors.border,
          // iOS: safe area (tab bar под home indicator)
        },
      }}
    >
      <Tabs.Screen name="Map" component={MapStack}
        options={{ tabBarLabel: 'Карта', tabBarIcon: ({color, size}) => <TabIcon name="🗺️" .../> }} />
      <Tabs.Screen name="Orders" component={OrdersStack}
        options={{ tabBarLabel: 'Поездки' }} />
      <Tabs.Screen name="Profile" component={ProfileStack}
        options={{ tabBarLabel: 'Профиль' }} />
    </Tabs.Navigator>
  );
}
```

> ⚠️ **Иконки:** используйте `@expo/vector-icons` (Ionicons): `<Ionicons name="map-outline" size={26} color={color} />`.

**Навигация во вложенный экран извне** (например: deep link «заказ 123» → Orders → RideDetail):
```ts
navigation.navigate('Main', { screen: 'Orders', params: { screen: 'RideDetail', rideId: '123' } });
```
(в типизированном мире — `NavigatorScreenParams` поможет).

---

## 5. Практика: MapStack (сердце приложения)

```tsx
// src/navigation/MapStack.tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { MapScreen } from '@/features/map/MapScreen';
import { AddressFromScreen } from '@/features/map/AddressFromScreen';
import { AddressToScreen } from '@/features/map/AddressToScreen';
import { ClassChoiceScreen } from '@/features/map/ClassChoiceScreen';
import { OrderScreen } from '@/features/ride/OrderScreen';
import { MapStackParamList } from './types';

const Stack = createNativeStackNavigator<MapStackParamList>();

export function MapStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Map" component={MapScreen} options={{ headerShown: false }} />
      <Stack.Screen name="AddressFrom" component={AddressFromScreen} options={{ title: 'Откуда' }} />
      <Stack.Screen name="AddressTo" component={AddressToScreen} options={{ title: 'Куда' }} />
      <Stack.Screen name="ClassChoice" component={ClassChoiceScreen} options={{ title: 'Поездка' }} />
      <Stack.Screen name="Order" component={OrderScreen}
        options={{ presentation: 'modal', animation: 'slide_from_bottom' }} />
    </Stack.Navigator>
  );
}
```

**Здесь пока заглушки экранов** (реальные — в модулях 4–9). Создайте `src/features/map/` с тремя простыми экранами-заглушками (Text + кнопка «Далее»).

---

## 6. Практика: Deep Links

### URI Scheme

```json
// app.json
{ "expo": { "scheme": "ridego" } }
```

→ приложение откликается на `ridego://...`. Примеры:
- `ridego://map` — открыться на карте
- `ridego://orders/123` — на заказ 123
- `ridego://order?rideId=123&status=arrived` — push (модуль 8)

### Универсальные ссылки (universal links)

```json
{
  "expo": {
    "scheme": "ridego",
    "ios": { "bundleIdentifier": "com.yourname.ridego", "config": { "usesNonExemptPermissions": [] } },
    "plugins": [
      [
        "expo-build-properties",
        {
          "ios": { "usesNonExemptPermissions": false },
          "android": { "package": "com.yourname.ridego" }
        }
      ]
    ]
  }
}
```

> ⚠️ Universal links требуют **ассоциированный домен** (Apple) и `well-known` файл (Android). Для курса достаточно **URI scheme** (проще, покрывает push/гео). Universal links — бонус.

### Настройка deep links на экранах

```tsx
// MapStack — добавляем href
<Stack.Screen name="Order" component={OrderScreen}
  options={{
    presentation: 'modal',
    href: '/order/:rideId',   // ridego://order/123
  }} />

<Stack.Screen name="Map" options={{ href: '/map' }} />

// OrdersStack
<Stack.Screen name="RideDetail" options={{ href: '/orders/:rideId' }} />
```

### Программный deep link (тест из терминала)

```bash
# iOS-симулятор
xcrun simctl openurl booted "ridego://map"

# Android-эмулятор
adb shell am start -a android.intent.action.VIEW -d "ridego://orders/123"
```

> 💡 **Deep link из terminated-приложения** (push, модуль 8): React Navigation обрабатывает при запуске — убедитесь, что `NavigationContainer` в корне.

### Обработка deep link «снаружи»

```tsx
// App.tsx
<NavigationContainer
  linking={{
    prefixes: ['ridego://', 'https://ridego.app'],   // scheme + universal
    config: {
      screens: {
        Root: {
          screens: {
            Main: {
              screens: {
                Map: { screens: { Map: { path: 'map' }, Order: { path: 'order/:rideId' } } },
                Orders: { screens: { RideDetail: { path: 'orders/:rideId' } } },
              },
            },
            Auth: { screens: { Register: { path: 'register' } } },
          },
        },
      },
    },
  }}
>
  <RootNavigator />
</NavigationContainer>
```

> ⚠️ **`config.screens`** дублирует `href` — но централизует (в одном месте). Выберите **один** подход (рекомендую централизованный `config`).

---

## 7. Практика: OrdersStack + ProfileStack (заглушки)

**OrdersStack:**
```tsx
const Stack = createNativeStackNavigator<OrdersStackParamList>();
export function OrdersStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="OrdersList" component={OrdersListScreen} options={{ title: 'Поездки' }} />
      <Stack.Screen name="RideDetail" component={RideDetailScreen} options={{ title: 'Детали' }} />
    </Stack.Navigator>
  );
}
```

**ProfileStack:**
```tsx
const Stack = createNativeStackNavigator<ProfileStackParamList>();
export function ProfileStack() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Profile" component={ProfileScreen} options={{ title: 'Профиль' }} />
      <Stack.Screen name="Settings" component={SettingsScreen} options={{ title: 'Настройки' }} />
    </Stack.Navigator>
  );
}
```

Подключите все три стека в `MainTabs`. Обновите `types.ts` с соответствующими param lists.

---

## 📝 Домашнее задание (Модуль 3)

### Обязательные задачи

1. **Полный граф навигации**
   - RootStack (Auth | Main)
   - AuthStack: Register, Login, Otp (заглушки экранов)
   - MainTabs: Map, Orders, Profile
   - MapStack: Map → AddressFrom → AddressTo → ClassChoice → Order (modal)
   - OrdersStack: OrdersList → RideDetail
   - ProfileStack: Profile → Settings
   - **Все экраны — заглушки** (Text с названием + кнопка перехода). Данные придут в модуле 4.

2. **Типизация (жёстко)**
   - `types.ts` со всеми param lists
   - Глобальный `declare` для `useNavigation()`
   - Каждый экран: `useNavigation<NativeStackNavigationProp<XxxParamList>>()`
   - **Проверьте:** `navigation.navigate('Order', { rideId: 123 })` — строка → ошибка TS. Значит типизация работает.

3. **Deep links**
   - URI scheme `ridego://` в `app.json`
   - `config.screens` в `NavigationContainer` (централизованный)
   - Проверка: `xcrun simctl openurl booted "ridego://orders/42"` → откроет RideDetail с `rideId=42`
   - Проверка: `adb shell am start -a android.intent.action.VIEW -d "ridego://map"` → откроет Map

4. **Auth guard (stub)**
   - `useAuth` (stub) с `isInitialized`, `user`, `login`, `logout`
   - RootStack переключается Auth ↔ Main
   - `login()` — переключает на Main (в модуле 6 — настоящий)
   - Проверка: «Не authed» → AuthStack, нажали «Войти» → MainTabs

5. **Опции экранов**
   - `headerShown: false` там, где нужен полный UI (Map)
   - `presentation: 'modal'` для Order
   - Анимации: `animation: 'slide_from_bottom'` для модалок
   - Кастомный header на ClassChoice (двойной input from/to)

### Дополнительные задачи

6. **Типизация вложенной навигации:** `navigate('Main', { screen: 'Map', params: { screen: 'Order', ... } })` — убедитесь, что TS «видит» вложенность
7. **Custom header** на MapScreen (поиск + кнопка) — вместо системного
8. **State preservation:** перезапустите deep link, пока приложение открыто — навигация должна корректно восстановить

### Критерии сдачи

- [ ] Полный граф (3 стека + 2 вложенных) — все переходы работают
- [ ] Типизация: опечатка в параметре = ошибка TS
- [ ] Deep link: `ridego://orders/42` → RideDetail (на обеих платформах)
- [ ] Auth guard: переключение Auth ↔ Main без «миганий»
- [ ] Модалка Order: `slide_from_bottom`, `presentation: 'modal'`
- [ ] Typecheck + Lint зелёные
- [ ] PR: `feat: navigation (root, tabs, stacks, deep links, auth guard stub)`

### Как проверить

1. Откройте приложение → AuthStack (stub не authed)
2. «Войти» → MainTabs (3 вкладки)
3. Вкладка Map → «Откуда» → «Куда» → «Класс» → «Вызвать» (модалка Order)
4. Вкладка Orders → «Детали» (RideDetail)
5. `xcrun simctl openurl booted "ridego://orders/42"` → переключится на Orders → RideDetail
6. `adb shell am start -a android.intent.action.VIEW -d "ridego://map"` → Map

---

## 💡 Полезные команды (модуль 3)

```bash
# Установка
npx expo install @react-navigation/native @react-navigation/native-stack @react-navigation/bottom-tabs react-native-screens react-native-safe-area-context

# Deep links
xcrun simctl openurl booted "ridego://map"
adb shell am start -a android.intent.action.VIEW -d "ridego://map"

# Отладка
# React Navigation: включите "Debug" (три точки в dev menu → "React Navigation")
#   → увидите текущий route, можно "Reset"
```

---

## 🐛 Частые ошибки и решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Cannot read property ... of undefined` при deep link | Нет `config.screens` для экрана | Добавьте в `linking.config` (и `href` на экране) |
| Double navigation bar (два tab bar) | Таб-вкладка в табе | Проверьте: таб-вкладка не вложена в другой таб |
| Модалка «под» таб-баром | `presentation: 'modal'` без safe-area | Убедитесь: `SafeAreaProvider` в корне; в модалке — свой header |
| Deep link из push не срабатывает | Приложения закрыто | `NavigationContainer` должен обрабатывать «холодный» старт (в корне App) |
| `NavigatorScreenParams` — тип не «видит» | Не подключён в `RootStackParamList` | Добавьте `import { NavigatorScreenParams } from '@react-navigation/native'` |
| Переход в «под» текущим экраном | `navigate` не работает | Используйте `push` (или проверьте, что имя экрана корректно) |
| Swipe-back «ломает» модалку | Нативный жест конфликтует | Для модалки: `gestureEnabled: false` (или оставьте — на усмотрение) |
| `useNavigation` «пустой» в компоненте | Компонент вне `NavigationContainer` | Убедитесь: компонент рендерится **внутри** навигатора |

---

## 💡 Советы на модуль

- **Типизация = ваш друг.** Опечатка в `navigate('Order')` = баг на проде. Типы ловят на этапе написания.
- **`navigate` vs `push`:** `navigate` — «перейти (или создать)», `push` — «точно создать новый». Для «экранов-деталей» — `push`, для «главных» — `navigate`.
- **Deep link — это контракт.** Документируйте все `href` (в `deep-links.md`). Frontend/бэкенд/маркетинг — все ссылаются на эти пути.
- **Гвард — один.** Не делайте 5 «защищённых» экранов. Один `RootNavigator` решает. Остальные — честно внутри.
- **Тестируйте обе платформы.** Swipe-back (iOS), back-кнопка (Android) — ведут себя по-разному.

**[→ Модуль 4: Данные, API, списки](../Module_04_Data_API_Lists/content.md)**
