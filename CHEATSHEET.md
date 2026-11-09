# 📎 Шпаргалка: React Native + Expo

**[← Назад к README](README.md)**

Компактная шпаргалка. Не заменяет документацию — напоминает синтаксис и паттерны.

---

## 1. Команды Expo

```bash
# Создание проекта
npx create-expo-app@latest my-app --template blank-typescript
npx create-expo-app@latest my-app --template default

# Запуск
npx expo start                 # dev server (Metro)
npx expo start --tunnel        # через туннель (без общего Wi-Fi)
npx expo start --dev-client    # для dev build
npx expo start --port 8082     # другой порт

# Установки
npx expo install <package>     # ⚠️ ВСЕГДА expo install, а не npm i (сверяет версии с SDK)
npx expo install react-native-maps expo-location

# Отладка / сборка
npx expo doctor                # диагностика окружения
npx expo export --platform android --platform ios   # production bundle (без app)
npx eas build -p ios --profile development
npx eas build -p android --profile preview
npx eas build -p all --profile production
npx eas update                 # OTA-публикация

# Подключение нативных модулей
npx expo prebuild              # создать ios/ + android/ из app.json
npx expo prebuild --clean      # пересоздать нативные папки (после config plugins)
npx expo run:ios               # собрать и запустить на Xcode
npx expo run:android           # собрать и запустить на эмуляторе/устройстве
```

> ⚠️ `npx expo install` — **золотое правило**. Он подтягивает версию, совместимую с текущим Expo SDK (иначе — конфликты нативных зависимостей).

---

## 2. Базовые компоненты (синтаксис)

```tsx
import { View, Text, StyleSheet, Image, TextInput, ScrollView, Pressable, Switch } from 'react-native';

<View style={styles.container}>
  <Text style={styles.title}>Hello, RideGo</Text>
  <Image source={require('./assets/car.png')} style={styles.car} />
  <TextInput
    placeholder="Куда?"
    value={q}
    onChangeText={setQ}
    style={styles.input}
    returnKeyType="go"
  />
  <Pressable
    onPress={go}
    style={({ pressed }) => [styles.btn, pressed && styles.btnPressed]}
    hitSlop={{ top: 8, right: 8, bottom: 8, left: 8 }}   // ⚠️ зона тапа больше видимой
    accessibilityLabel="Вызвать поездку"
  >
    <Text style={styles.btnText}>Вызвать</Text>
  </Pressable>
  <Switch value={on} onValueChange={setOn} />
</View>

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, gap: 12, backgroundColor: 'white' },
  title: { fontSize: 24, fontWeight: '700' },
  input: { borderWidth: 1, borderRadius: 12, padding: 12 },
  btn: { backgroundColor: '#111', borderRadius: 12, padding: 16, alignItems: 'center' },
  btnPressed: { opacity: 0.8 },
  btnText: { color: 'white', fontWeight: '600' },
});
```

---

## 3. Flexbox (то, что отличается от веба)

```
flexbox в RN = flexbox в CSS, НО:
  • По умолчанию flex-direction: column (не row!)
  • flex: 1 — главный инструмент (займи всё оставшееся место)
  • Нет position: fixed — используйте absolute
  • gap поддерживается с RN 0.71+

Частые паттерны:
  «по центру»:        { alignItems: 'center', justifyContent: 'center' }
  «колодка bottom»:   { position: 'absolute', bottom: 0, left: 0, right: 0 }
  «строка + spacer»:  <View style={{flexDirection:'row'}}><A /><View style={{flex:1}} /><B /></View>
```

### Safe Area (notch / Dynamic Island)

```tsx
import { SafeAreaView, Platform } from 'react-native';
// или лучше — react-native-safe-area-context (нужен provider в корне)
import { SafeAreaProvider, SafeAreaView as Safe } from 'react-native-safe-area-context';

<Safe edges={['top', 'bottom']}>...</Safe>   // учитывает вырезы
```

### Клавиатура

```tsx
import { KeyboardAvoidingView, Platform, Keyboard } from 'react-native';
<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : undefined}
  style={{ flex: 1 }}
>
  ...
</KeyboardAvoidingView>
Keyboard.dismiss();   // программно закрыть
```

---

## 4. Платформенная специфика

```tsx
import { Platform, Appearance } from 'react-native';

const color = Platform.select({ ios: 'systemBlue', android: '#2196F3', default: '#007AFF' });
const file  = Platform.select({ ios: 'icon.ios.png', android: 'icon.png' });

// Папки по платформам: button.ios.tsx / button.android.tsx —Metro подхватит сам
// useColorScheme() — системная тема
const dark = useColorScheme() === 'dark';
```

---

## 5. Платформенные жесты и отклики

| Действие | iOS | Android |
|----------|-----|---------|
| Назад | swipe-back (системный) | системная кнопка ⬅ (обрабатывается) |
| Жест вниз на листе | свайп (pull-to-refresh) | свайп |
| Короткий тап | `onPressIn/Out` + haptics | `onPressIn/Out` + haptics |
| Долгий тап | `onLongPress` (500ms) | `onLongPress` |
| Вибрация | `HapticsImpact` (Reanimated) | `HapticsImpact` |

---

## 6. Состояние и хранилище

```tsx
// Локальный
const [q, setQ] = useState('');

// Context (медленно при частых изменениях — только для «редко меняющихся» тем)
const ThemeCtx = createContext<Theme>(lightTheme);

// TanStack Query (server-state)
const { data, isPending, isError, refetch } = useQuery({
  queryKey: ['drivers', { q, page }],
  queryFn: ({ signal }) => api.getDrivers({ q, page }, signal),   // AbortController!
  staleTime: 30_000,
});

const mutation = useMutation({
  mutationFn: toggleFavorite,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['favorites'] });
    const prev = queryClient.getQueryData(['favorites']);
    queryClient.setQueryData(['favorites'], (old) => optimistic(old, id));
    return { prev };
  },
  onError: (_e, _id, ctx) => ctx?.prev && queryClient.setQueryData(['favorites'], ctx.prev),  // rollback
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['favorites'] }),
});
```

---

## 7. Анимации (Reanimated)

```tsx
import Animated, { FadeIn, SlideInDown, useSharedValue, useAnimatedStyle, withTiming, withSpring, runOnUI } from 'react-native-reanimated';

const translateY = useSharedValue(0);
translateY.value = withSpring(100, { damping: 20, stiffness: 120 });
const style = useAnimatedStyle(() => ({ transform: [{ translateY: translateY.value }] }));
<Animated.View style={style}>...</Animated.View>

// Пресеты (сразу 60 fps)
<Animated.View entering={FadeIn.duration(200)}>...</Animated.View>
<Animated.View entering={SlideInDown.springify()}>...</Animated.View>

// Worklet-функция (выполняется в UI-потоке, не в JS!)
const anim = () => {
  'worklet';   // ← обязательная директива
  translateY.value = withTiming(0, { duration: 300 });
};
```

> ⚠️ **Worklets:** внутри `'worklet'` нельзя использовать JS-конструкции, недоступные в UI-потоке (fetch, console, setState, сложные объекты). Всё тяжёлое — `'worklet'`, всё «умное» — в JS.

---

## 8. Навигация (React Navigation)

```tsx
// Типизация маршрутов
export type RootStackParamList = {
  Map: undefined;
  Order: { id: string };
  Auth: undefined;
};
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// Stack
<Stack.Screen name="Map" component={MapScreen} />
<Stack.Screen name="Order" component={OrderScreen} options={{ presentation: 'modal' }} />

// Tabs
<Tabs.Screen name="Map" options={{ tabBarIcon: ... }} />

// Императивно
navigation.navigate('Order', { id: '123' });
navigation.goBack();
navigation.push / pop / replace

// Deep link
options={{ href: '/order/:id' }}   // ridego://order/123
```

---

## 9. Геолокация (expo-location)

```ts
import * as Location from 'expo-location';

// Разрешение
const { status } = await Location.requestForegroundPermissionsAsync();
if (status !== 'granted') throw new Error('Denied');

// Однократно
const pos = await Location.getCurrentPositionAsync({
  accuracy: Location.Accuracy.Balanced,   // Low | Balanced | High
});

// Подписка (cleanup обязателен!)
const sub = await Location.watchPositionAsync(
  { accuracy: Location.Accuracy.High, distanceInterval: 10 },
  (loc) => { ... }
);
useEffect(() => () => sub.remove(), []);
```

---

## 10. Push (expo-notifications)

```ts
import * as Notifications from 'expo-notifications';

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldPlaySound: false,
    shouldSetBadge: false,
    shouldShowBanner: true,      // foreground: наш баннер
    shouldShowList: true,
  }),
});

const sub = Notifications.addNotificationResponseReceivedListener((resp) => {
  const data = resp.notification.request.content.data;
  // data.orderId → deep link на экран заказа
});
```

---

## 11. Хранилище

```ts
// MMKV (быстро, синхронно)
import { MMKV } from 'react-native-mmkv';
export const storage = new MMKV();
storage.set('favorites', JSON.stringify([...ids]));
storage.getString('favorites');

// Keychain/Keystore (ТОЛЬКО токены и секреты)
import { SECURITY_LEVEL, getGenericPassword, setGenericPassword } from 'react-native-keychain';
await setGenericPassword('ridego-user', refreshToken, {
  service: 'com.yourname.ridego',
  securityLevel: SECURITY_LEVEL.KSECURITY_CLASS_KSF,   // не экспортируется в бэкапы
});
const creds = await getGenericPassword({ service: 'com.yourname.ridego' });
```

---

## 12. Тестирование (кратко)

```tsx
// RNTL (render + userEvent)
import { render, screen } from '@testing-library/react-native';
import userEvent from '@testing-library/user-event';

it('logs in', async () => {
  render(<LoginForm />);
  await userEvent.type(screen.getByPlaceholderText('Телефон'), '+79991234567');
  await userEvent.press(screen.getByRole('button', { name: /войти/i }));
  expect(await screen.findByText('Привет')).toBeOnTheScreen();
});
```

```yaml
# Maestro (E2E)
- launchApp
- tapOn: "Войти"
- inputText: "+79991234567"
- assert:
    visible: "Введите код"
```

---

## 13. Частые конфиги

### `app.json` — permissions (на примере geo + push)

```json
{
  "expo": {
    "plugins": [
      [
        "expo-location",
        {
          "locationWhenInUsePermission": "RideGo использует геолокацию, чтобы показать вашу позицию и построить маршрут."
        }
      ],
      "expo-notifications"
    ],
    "ios": {
      "infoPlist": {
        "UIBackgroundModes": ["location", "fetch"]   // фон: трекинг
      }
    },
    "android": {
      "permissions": ["ACCESS_COARSE_LOCATION", "ACCESS_FINE_LOCATION", "FOREGROUND_SERVICE"]
    }
  }
}
```

### `babel.config.js` — Reanimated

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: ['react-native-reanimated/plugin'],   // ⚠️ должен быть ПОСЛЕДНИМ
  };
};
```

### `tsconfig.json` — строгость (шаблон уже даёт)

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "paths": { "@/*": ["./src/*"] }   // алиасы
  }
}
```

---

## 14. Диагностика: быстрая таблица

| Симптом | Куда смотреть |
|---------|--------------|
| Белая/чёрная страница | Терминал Metro → Console (нативные логи) |
| Ошибка только на Android | `adb logcat` |
| Ошибка только на iOS | Xcode → Devices → Console |
| Лags (не crash) | React DevTools Profiler + Android Profiler (CPU/ANR) |
| Утечка памяти | Memory snapshot в DevTools до/после сессии |
| Ошибка метаданных (scheme, permissions) | `npx expo prebuild` + читать нативные файлы ios/ + android/ |
| Metro не видит изменения | `npx expo start -c` (очистить кэш) |

---

## 15. Границы OTA (EAS Update) — что можно/нельзя обновить

| Можно по OTA | Нельзя по OTA |
|--------------|---------------|
| JS/TS-код | Нативные модули (package.json native) |
| Ресурсы JS (изображения в bundle) | Новый permission (Info.plist/manifest) |
| Тексты, логики, стили | Версию нативного модуля |
| | Изменения app.json (иконка, name) |

**Вывод:** хотфикс UI/логики — OTA (минуты). Новая нативная зависимость — стор-релиз (часы/дни review).

---

**[Наверх ↑](#-шпаргалка-react-native--expo)**
