# Модуль 6: Авторизация и безопасность

**[← Назад к индексу](../index.md)** | **[Модуль 5: Состояние](../Module_05_State_Management/content.md)** | **Далее: [Модуль 7: Гео и карты](../Module_07_Geo_Maps/content.md) →**

---

## 🎯 Цель модуля

- **Полный auth-cycle**: телефон → OTP → JWT (access + refresh) → `/me` → внутрь
- **Keychain** для секретов, **биометрия** для повторного входа
- **Single-flight refresh**: 5 параллельных 401 → **один** refresh
- **Формы** на RHF + zod: валидация, UX ошибок, keyboard
- **Auth-guard** в навигации (наконец-то по-настоящему)
- **Релиз v0.2.0** (первый «настоящий»)

**Длительность: ~22 часа. После M6 → v0.2.0.**

---

## 📚 Теория

### 6.1. Модель токенов: access + refresh

```
Access token:  JWT, TTL 15 минут. В памяти (api-client). Даёт доступ к API.
Refresh token: JWT, TTL 30 дней. В Keychain. Даёт новые access-пары.
```

**Почему два:** access короткий — украсть = мало времени на злоупотребление. Refresh длинный — но лежит в безопасном хранилище и ротится (каждый refresh → новый, старый протухает).

**Формат JWT (знать):** `header.payload.signature`. Payload: `sub` (user id), `exp`, `iat`, `type: 'access' | 'refresh'`. **Никогда** не подписывайте «секреты» в payload — base64, не шифр.

### 6.2. Refresh flow: single-flight (самая ценная тема модуля)

**Проблема:** приложение делает 5 запросов параллельно. Access протух. Все 5 получают 401. Все 5 идут на `/refresh`. **Что плохо:** 5 refresh-запросов, а refresh-токен **ротируется** → 4 из 5 получают 401 «invalid refresh» → **логиним пользователя**, хотя всё было в порядке.

**Решение — single-flight (один «лёт»):**

```ts
// src/shared/api/auth-refresh.ts
type RefreshPromise = Promise<{ access: string; refresh: string }> | null;

class TokenManager {
  private access: string | null = null;
  private inflight: RefreshPromise = null;

  setTokens(access: string, refresh: string) {
    this.access = access;
    void secrets.save(refresh);   // Keychain
  }

  async getAccess(): Promise<string> {
    if (this.access) return this.access;
    return this.refresh().then((t) => t.access);
  }

  async refresh(): Promise<{ access: string; refresh: string }> {
    // КЛЮЧ: если refresh уже в полёте — возвращаем ТОТ ЖЕ promise
    if (this.inflight) return this.inflight;

    this.inflight = (async () => {
      try {
        const stored = await secrets.load();
        if (!stored) throw new AuthError('NO_CREDENTIALS');
        const res = await rawFetch('/auth/refresh', {
          method: 'POST',
          body: JSON.stringify({ refresh: stored }),   // БЕЗ access (его уже нет)
          // headers: application/json
        });
        if (!res.ok) throw new AuthError(res.status === 401 ? 'INVALID_REFRESH' : 'HTTP');
        const { access, refresh } = await res.json();
        this.access = access;
        await secrets.save(refresh);   // ротация: новый refresh в Keychain
        return { access, refresh };
      } finally {
        this.inflight = null;   // разблокировать следующих
      }
    })();

    return this.inflight;
  }

  clear() {
    this.access = null;
    this.inflight = null;
    void secrets.clear();
  }
}

export const tokenManager = new TokenManager();
```

**Interceptor (подключение к api-client M4):**

```ts
// src/shared/api/client.ts — добавляем
api.interceptors.request.use((config) => {
  // access в памяти: если есть — ставим; если нет — ленивый refresh
  // (делаем async-interceptor: ждём getAccess)
  return tokenManager.getAccess()
    .then((access) => { config.headers.Authorization = `Bearer ${access}`; return config; })
    .catch(() => config);   // без токена (public-эндпоинты)
});

api.interceptors.response.use(
  (r) => r,
  async (error) => {
    const { config, response } = error;
    if (response?.status === 401 && !config._retried && !config.url?.includes('/auth/')) {
      config._retried = true;
      try {
        await tokenManager.refresh();
        return api(config);              // повтор ТОГО ЖЕ запроса
      } catch {
        tokenManager.clear();
        store.dispatch(loggedOut('session_expired'));   // → auth slice
        return Promise.reject(new AuthError('SESSION_EXPIRED'));
      }
    }
    throw error;
  },
);
```

**Unit-тест single-flight (обязательный):**

```ts
it('5 parallel 401 → 1 refresh', async () => {
  const refreshSpy = jest.spyOn(rawFetch, ...);   // mock /auth/refresh
  tokenManager.access = 'expired';
  // 5 запросов параллельно, все получают 401
  await Promise.all([api.get('/a'), api.get('/b'), api.get('/c'), api.get('/d'), api.get('/e')]);
  expect(refreshSpy).toHaveBeenCalledTimes(1);     // ← ключевое
});
```

### 6.3. Auth-слайс (RTK) + guard

```ts
// src/store/auth.ts
type AuthState = {
  status: 'idle' | 'loading' | 'authenticated' | 'unauthenticated';
  user: User | null;
};

export const authSlice = createSlice({
  name: 'auth',
  initialState: { status: 'idle', user: null } as AuthState,
  reducers: {
    authLoading: (s) => { s.status = 'loading'; },
    authSuccess: (s, a: PayloadAction<User>) => { s.status = 'authenticated'; s.user = a.payload; },
    loggedOut: (s, _a: PayloadAction<string>) => { s.status = 'unauthenticated'; s.user = null; },
  },
});
```

**Бутстрап при запуске** (в App, до навигации):

```ts
// src/features/auth/bootstrap.ts
export async function bootstrapAuth(): Promise<'authenticated' | 'unauthenticated'> {
  const refresh = await secrets.load();
  if (!refresh) return 'unauthenticated';

  store.dispatch(authLoading());
  try {
    await tokenManager.refresh();          // получаем свежий access
    const { data: me } = await api.get<User>('/me');
    store.dispatch(authSuccess(me));
    return 'authenticated';
  } catch {
    tokenManager.clear();
    store.dispatch(loggedOut('bootstrap_failed'));
    return 'unauthenticated';
  }
}
```

**Guard** (в App):

```tsx
const [ready, setReady] = useState(false);
useEffect(() => {
  bootstrapAuth().then((s) => {
    // если был refresh → предложить биометрию? (опционально)
    setReady(true);
  });
}, []);

if (!ready) return <SplashScreen />;   // M1 — теперь он «по делу»

return (
  <Navigation>
    {ready && (
      authStatus === 'authenticated' ? <MainTabs /> : <AuthStack />
    )}
  </Navigation>
);
```

**Сессия протухла в фоне** → interceptor (6.2) dispatch `loggedOut` → App переключает на AuthStack **сам**.

### 6.4. Формы: RHF + zod

```bash
npx expo install react-hook-form @hookform/resolvers zod
```

```tsx
// src/features/auth/forms/phoneSchema.ts
import { z } from 'zod';

export const phoneSchema = z.object({
  phone: z
    .string()
    .min(10, 'Введите номер')
    .regex(/^\+?[0-9]{10,15}$/, 'Неверный формат')
    .refine((v) => validRuPhone(v), 'Введите российский номер'),
});

export type PhoneForm = z.infer<typeof phoneSchema>;
```

```tsx
// src/features/auth/forms/PhoneForm.tsx
export function PhoneForm({ onSubmit }: { onSubmit: (phone: string) => Promise<void> }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<PhoneForm>({
    resolver: zodResolver(phoneSchema),
    defaultValues: { phone: '' },
  });

  return (
    <KeyboardAvoidingView behavior="padding" style={{ flex: 1 }}>
      <Text style={t.typo.h1}>Добро пожаловать в RideGo</Text>
      <Text style={t.typo.body}>Войдите по номеру телефона</Text>

      <Input
        label="Телефон"
        placeholder="+7 999 123-45-67"
        keyboardType="phone-pad"
        autoCapitalize="none"
        autoCorrect={false}
        testID="phone-input"
        {...register('phone')}
        error={errors.phone?.message}
      />

      <Button title="Продолжить" loading={isSubmitting} testID="phone-submit">
        {/* handleSubmit вызывает onSubmit с валидированными значениями */}
      </Button>
      {/* onPress={handleSubmit((d) => onSubmit(d.phone))} */}
    </KeyboardAvoidingView>
  );
}
```

**Почему RHF (а не controlled state):**
- **Uncontrolled** (React Native TextInput сам хранит значение) → меньше перерендеров (важно при клавиатуре)
- Валидация по событию (`onBlur`/`onChange`) — настраивается, а не пишется вручную
- `isSubmitting` из коробки (для loading-кнопок)
- `zodResolver` — единый source of truth для схем (M9-оценка, M7-адреса — те же схемы)

### 6.5. OTP: код в SMS

```tsx
// 4–6 полей, auto-advance, paste-support
function OtpInput({ value, onChange, onComplete, error }: OtpProps) {
  const refs = useRef<(TextInput | null)[]>([]);
  // onChange: цифра → текущее поле, авто-переход на следующее (refs[i+1].focus())
  // onDelete: пустое → focus предыдущее
  // onPaste: если вставили 6 цифр — заполнить все
  // value.length === 6 → onComplete(value)
}
```

**Правила UX OTP:**
- Поля маленькие, с автопереходом
- Таймер повторной отправки: **60 с** (disabled кнопка, обратный отсчёт)
- Ошибочный код → **3 попытки**, затем «запросить новый код» (с кулдауном)
- **Не логгеруем код** (ни console, ни Sentry)

### 6.6. Биометрия

```bash
npx expo install expo-local-authentication
```

```ts
// src/features/auth/biometrics.ts
import * as LocalAuthentication from 'expo-local-authentication';

export async function canUseBiometrics(): Promise<boolean> {
  try {
    if (!LocalAuthentication.isEnrolled()) return false;   // на устройстве нет Face/Touch ID
    const supported = await LocalAuthentication.supportedAuthenticationTypesAsync();
    return supported.length > 0;
  } catch { return false; }
}

export async function authenticateWithBiometrics(): Promise<boolean> {
  try {
    const result = await LocalAuthentication.authenticateAsync({
      promptMessage: 'Войдите в RideGo',
      cancelLabel: 'Ввести код',
      fallbackLabel: 'Ввести код',
      // ios: allowDeviceCredential как fallback — настраиваем
    });
    return result.success;
  } catch { return false; }
}
```

**Сценарий:** при запуске, если есть сохранённая сессия + пользователь включил «вход по Face ID» (настройка в профиле) → `authenticateWithBiometrics()` → успех = bootstrap без OTP; отказ = обычный bootstrap (OTP не нужен, у нас refresh).

> ⚠️ Биометрия **разблокирует доступ к Keychain** (securityLevel с biometry) — она не «авторизует» на сервере, а лишь разрешает прочесть refresh-токен. Это важно понять: биометрия = gate к локальному секрету.

### 6.7. Безопасность: чек-лист мидла

| Правило | Зачем |
|---------|-------|
| Токены — только в Keychain | Не читать из файлов/бэкапов |
| Access — в памяти, не в storage | Минимум времени жизни на диске |
| HTTPS + certificate pinning (опц.) | Защита от MITM |
| Никогда не логгеруем токены/коды | Логи уходят в Sentry/файлы |
| Refresh ротируется | Украденный refresh протухает |
| Rate-limit на OTP (бэкенд) | Брут-форс кода |
| Sanitize input перед отправкой | Базовая гигиена |
| Не доверяем client (валидация на сервере) | Client — UX, server — security |

### 6.8. Logout и «протухание»

```ts
export async function logout() {
  // 1. Опционально: PUT /auth/logout (сервер инвалидирует refresh) — fire-and-forget
  // 2. Локально: tokenManager.clear(); dispatch(loggedOut('user'));
  // 3. Очистка: rideDraft, (НЕ favorites — они синятся), push-токен → unsubscribe
  // 4. Навигация: navigation.reset({ routes: [{ name: 'Phone' }] })
}
```

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Мок-бэкенд: auth
```
POST /auth/register   { phone }          → { ok } (+ генерим код, логим в консоль)
POST /auth/otp        { phone, code }    → { access, refresh }   (код из лога/фикс 123456)
POST /auth/login      { phone, password? } → (для M6 — опционально; основной путь — OTP)
POST /auth/refresh    { refresh }        → { access, refresh }   (ротация!)
GET  /me              (Bearer)           → User
PUT  /me              { name, avatarUrl } → User
DELETE /auth/logout   (Bearer refresh)   → { ok }
```
- **Ротация refresh** в SQLite: таблица `refresh_tokens` (jti, user_id, expires_at, revoked). Новый refresh → старый `revoked=true`.
- Access: JWT HS256, TTL 15m. Refresh: TTL 30d, ротируется.
- Код OTP: **фикс 123456** (для dev) + лог в консоль (никогда не в 200-ответе!)

### Шаг 2. Auth-скрины
- `Phone` (M6.4): телефон + «Продолжить»
- `Otp`: 6 полей + таймер 60с + «повторить»
- После успеха → `/me` → authSuccess
- **Экраны в AuthStack** (M3): Phone, Otp (модалка)

### Шаг 3. Guards + bootstrap
- `bootstrapAuth()` (M6.3) + Splash до `ready`
- App: authenticated → MainTabs, else → AuthStack
- Interceptor single-flight (M6.2) + unit-тест «5×401 → 1 refresh»
- **Сценарий «протухание»:** вручную поставьте access с `exp` в прошлом → 5 параллельных запросов → 1 refresh → все ок (видите в логах)

### Шаг 4. Профиль + logout + биометрия
- Экран «Профиль»: аватар, имя (редактирование → RHF+zod → `PUT /me`), «О приложении»
- Переключатель «Вход по Face ID / Touch ID» (MMKV) + `canUseBiometrics`
- Кнопка «Выйти» → `logout()` (M6.8)
- Биометрия при запуске (M6.6) — по настройке

### Шаг 5. Push-токен (задел M8)
- (После M7-devbuild) `expo-notifications`: получить токен → `POST /notifications/token` (объект M8) — **stub** здесь

### Шаг 6. Релиз v0.2.0
- EAS build (production) → TestFlight + Play internal
- Чек: полный цикл (телефон → OTP → карта → избранные) работает на реальном устройстве

---

## ✅ Практическое задание модуля

### Задача: «Полный auth-cycle + v0.2.0»

**1. Мок-bэкенд (4ч)**
- [ ] 6 auth-эндпоинтов + ротация refresh + JWT
- [ ] Фикс OTP 123456 + лог; rate-limit (3 попытки/минуту на номер)

**2. Формы (4ч)**
- [ ] Phone (RHF+zod, phone-pad, маскирование) + Otp (6 полей, auto-advance, paste, 60с)
- [ ] Unit-тесты схем: `phoneSchema` (корректные/некорректные номера), OTP-логика (авто-переход, paste)

**3. Токены и refresh (5ч)**
- [ ] `tokenManager` (M6.2) + `secrets` (Keychain, M5.8) + interceptor
- [ ] **Unit-тест single-flight: 5×401 → 1 refresh** (критичный)
- [ ] Unit-тесты: rotation, invalid refresh → clear → loggedOut

**4. Гварды и bootstrap (3ч)**
- [ ] `bootstrapAuth` + Splash-до-ready + переключение AuthStack/MainTabs
- [ ] Тест «сессия протухла в фоне → сам на экран входа»

**5. Профиль и биометрия (3ч)**
- [ ] Профиль (аватар, имя RHF+zod → PUT /me), переключатель биометрии, logout
- [ ] Биометрия при запуске (настройка) + fallback на OTP

**6. Релиз v0.2.0 (2ч)**
- [ ] EAS production → TestFlight + Play internal
- [ ] Smoke-чек на реальном устройстве: полный цикл

**7. Отчёт (2ч)** — `docs/progress/M6.md`:
- Auth-диаграмма (access/refresh/single-flight/rotation)
- Скриншоты: телефон, OTP, профиль, «протухание»
- Результат single-flight теста (лог: 5 запросов, 1 refresh)
- Ошибки: 3+ (включая «почему не биометрия = серверная авторизация»)

---

## 📦 Итоговая структура после M6

```
src/features/auth/
├── AuthProvider.tsx      # bootstrap + состояние
├── bootstrap.ts          # M6.3
├── refresh.ts            # single-flight (tokenManager)
├── biometrics.ts
├── logout.ts
├── forms/
│   ├── PhoneForm.tsx
│   ├── OtpScreen.tsx     # + OtpInput
│   └── schemas.ts        # zod
└── ProfileScreen.tsx
src/store/
└── auth.ts               # + в index.ts
src/shared/api/
└── client.ts             # + interceptors (M6.2)
src/shared/keychain.ts    # (из M5 — теперь используется)
```

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| N параллельных refresh | 4 из 5 → 401 → логаут | Single-flight (M6.2) |
| Токен в AsyncStorage/MMKV | Украсть из файла | Keychain |
| Refresh без ротации | Украденный refresh живёт вечно | Ротация на бэкенде |
| Логгеруем код/токен | Утечка в лог/Sentry | Не логгеруем; санитайз |
| Bootstrap без Splash | Мигание UI | Splash до `ready` |
| Биометрия = «авторизация» | Не понимаете модель | Биометрия = gate к Keychain |
| Controlled TextInput в форме | Перерендеры при клавиатуре | RHF uncontrolled |
| Logout не чистит push-токен | Приходишь push в «вышедшего» | Unsubscribe в logout |

---

## 💡 Советы на модуль

- **Single-flight — вопрос №1 на RN-собес по auth.** Умейте объяснить за 2 минуты (M6.2).
- **Два токена, разные дома:** access — память (15м), refresh — Keychain (30д, ротируется).
- **Формы = UX.** Ошибка под полем, loading-кнопка, keyboard — 3 кита.
- **v0.2.0 — первый реальный релиз.** Относитесь как к прод: smoke-чек на живом устройстве.

**[→ Модуль 7: Гео и карты](../Module_07_Geo_Maps/content.md)**
