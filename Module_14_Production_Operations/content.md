# Модуль 14: Продакшн

**[← Назад к индексу](../index.md)** | **[Модуль 13: CI/CD](../Module_13_Release_CI_CD/content.md)** | **Далее: [Модуль 15: Hardening и v1.0](../Module_15_Hardening_V1/content.md) →**

---

## 🎯 Цель модуля

- **Observability**: Sentry (crash + performance), аналитика (события), logs, **метрики**
- **Цели** (из PROJECT): crash-free > 99.5%, error rate < 0.5%, P95 API < 300мс
- **Релиз**: staged rollout, **canary** метрики, **rollback**
- **Scale**: **ADR #3** (бэкенд), нагрузочные тесты
- **Privacy** (GDPR/152-ФЗ): consent, минимизация, удаление

**Длительность: ~22 часа. «Приложение в сторах — и вы за ним следите».**

---

## 📚 Теория

### 14.1. Три кита наблюдаемости

```
Crash/Performance   →  Sentry (M14.2)         — «где падает, где медленно»
Behavioral          →  Analytics (M14.3)       — «что делают, что не конвертят»
Runtime health      →  Logs + metrics (M14.4)  — «что происходит на сервере/клиенте»
```

**Цели (PROJECT):**
| Метрика | Целевое | Инструмент |
|---------|---------|-----------|
| **Crash-free** (users) | > 99.5% | Sentry / store crash reports |
| **Error rate** (не-crash) | < 0.5% | Sentry |
| **P95 API** | < 300мс | Sentry performance / APM |
| **P95 TTI** | < 2.5с | M10 |
| **Retention D7** | целевое | Analytics (M14.3) |

### 14.2. Sentry: crash + performance + release tracking

```bash
npx expo install @sentry/react-native
```

```ts
// src/shared/sentry.ts
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: __DEV__ ? 'dev' : 'production',
  tracesSampleRate: 0.1,                 // 10% трассировок (performance)
  release: `RideGo@${version}`,           // ← M13: версия (release tracking)
  dist: buildHash,                        // билд
  // Контекст (не PII — M14.7):
  // user: { id } — анонимный id (не email)
});
```

**Что ловим:**
- **Crash** (JS + нативные): `Sentry.captureException(e)` / авто (M1.2)
- **Performance**: трассировки (API, навигация, экраны) → **P95**
- **Breadcrumbs**: лог событий (M14.4)
- **Release tracking** (M13): по версии — «баг в 1.1.0, не в 1.0.0»
- **Issue** → алерт (M14.6)

**Правило (M14.7):** **никогда** PII в Sentry (email, геолокация, номера). Только **анонимный user id** + контекст.

### 14.3. Аналитика: события → воронка

```ts
// src/shared/analytics.ts
export function track(event: 'signup' | 'trip_start' | 'payment_success' | 'rating_given' | ..., props?: Record<string, unknown>) {
  // Production: Mixpanel/Amplitude/PostHog
  // Mок (M14): лог → бэкенд (для воронки)
  if (__DEV__) console.log('[analytics]', event, props);
}
```

**Ключевые события** (RideGo):
| Событие | Когда | Зачем |
|---------|-------|-------|
| `app_open` | запуск | DAU/retention |
| `signup_complete` | M6 | конверсия воронки |
| `trip_start` | M8 | активность |
| `payment_success` | M9 | **денежная** конверсия |
| `rating_given` | M9.8 | качество |
| `app_crash` | Sentry | (авто) |

**Воронка:** `app_open → signup → trip_start → payment_success → rating`. **Где утечка?** (M14.3). **Retention D1/D7** (M14.3).

### 14.4. Logs + метрики (сервер)

**Сервер** (M4): **структурные логи** (M14.5: pino/winston):
```
ts=... level=info event=trip_started ride_id=... class=ECONOMY ms=120
```
**Метрики** (Prometheus, опц.):
- **API**: latency (P50/P95/P99), error rate, RPS
- **WebSocket**: активные соединения, reconnect rate
- **Push**: delivered/failed
- **Платежи**: success/decline rate

**Алерты** (M14.6): по метрикам → канал (M14.6).

### 14.5. Конвенции логов (ADR #3 под-часть)

```
ts, level, event, ...ctx (без PII)
```
- **Уровни**: `debug` (dev), `info` (обычное), `warn` (аномалия), `error` (сбой)
- **Структурные** (JSON) → searchable
- **Без PII** (M14.7)
- **Корреляция**: `request_id` / `ride_id` (сквозной)

### 14.6. Алерты + runbook + on-call

**Алерты** (по целям M14.1):
| Метрика | Порог | Канал |
|---------|--------|-------|
| Crash-free | < 99.5% | Slack/Telegram |
| Error rate | > 0.5% | Slack |
| P95 API | > 500мс (2×цели) | Slack |
| Push failed | > 5% | Slack |
| 5xx | > 1% | **Пaging** (телефон) |

**Runbook** (M14.6): `docs/runbooks/`:
```
# CRASH SPIKE (v1.1.0)
1. Sentry: какой issue? частота? device?
2. Если 1.1.0 → M13: роллбэк OTA / остановить rollout
3. Hotfix (M13.4)
4. Postmortem (M14.6)
```

**On-call**: дежурный (красный экран → дежурный). **Postmortem** (blameless): «что случилось, почему, как не повторить».

### 14.7. Privacy: GDPR / 152-ФЗ

**Принципы:**
1. **Консент** (M14.8): явный выбор (гео, push, аналитика) — M6.6 уже есть
2. **Минимизация**: собираем **только** нужное (M14.1: без PII)
3. **Назначение**: гео — для поездки, не для рекламы
4. **Удаление**: «удалить аккаунт» → удалить данные (M14.8)

**Privacy-дисклейшн** (stores, M13.5): честно: «мы собираем геолокацию для отслеживания поездки, биометрию для входа, передаём Stripe платёж». **Store review** проверяет.

**Практика:**
```ts
// consent (M6.6) + удаление:
async function deleteAccount() {
  await api.delete('/me');   // бэкенд: удалить данные (GDPR)
  // локально: clearAsync (M5.5)
}
```

### 14.8. Scale: ADR #3 + нагрузочные тесты

**ADR #3 (бэкенд):**
| Критерий | Монолит (выбран) | Сервисы (no) |
|----------|------------------|--------------|
| Команда | 1 (курс) | команда |
| Сложность | Низкая | Высокая |
| Scale | Монолит до ~X RPS | — |
| **ADR #3: монолит** (Express/Fastify) **до** пределов; если RPS вырастет — **разделить** (ride/pay/geo) |

**Нагрузочные тесты** (autocannon/k6):
```bash
# k6: 1000 VU, 100мс → P95 < 300мс (цель M14.1)
k6 run load.js
```
**Проверить** (M14.8):
- API: P95 < 300мс при X RPS
- WebSocket: N активных соединений
- **Где предел** → ADR #3 (когда делить)

### 14.9. Release-операции: staged rollout + canary-метрики + rollback

**Staged rollout** (M13.5): **не 100% сразу**:
```
5% → (24ч, метрики M14) → 20% → 50% → 100%
```
**Canary-метрики** (M14.1): на 5%: **crash-free > 99.5%**, **error rate < 0.5%**, **P95 < 300мс**. **Зелёные** → дальше. **Красные** → **rollback** (M13.4) / **остановить rollout** (Play).

**Rollback** (M13.4):
- **JS** → OTA-роллбэк (минуты)
- **Натив** → остановить rollout (Play) / отозвать (App Store) → hotfix

**После релиза** (M13.6): **первые 24ч** — следим (Sentry, алерты). **Postmortem** при инциденте (M14.6).

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. Sentry (6ч)
- [ ] `@sentry/react-native` (M14.2): init (release, env, tracesSample)
- [ ] **Авто** (crash/JS) + ручной `captureException` (M14.2)
- [ ] **Release tracking** (M13): по версии
- [ ] **Performance** (P95 API): трассировки
- [ ] **Без PII** (M14.7): анонимный user id
- [ ] Проверить: «искусственный» crash → issue в Sentry

### Шаг 2. Аналитика (4ч)
- [ ] `track` (M14.3): ключевые события (M14.3)
- [ ] **Воронка** (M14.3): app_open → signup → trip → payment → rating
- [ ] **Retention** (M14.3): D1/D7
- [ ] (Мок → бэкенд; production → Mixpanel/Amplitude)

### Шаг 3. Logs + метрики + алерты (4ч)
- [ ] **Структурные логи** (M14.5) на бэкенде
- [ ] **Метрики** (API latency, WS, push, payments) (M14.4)
- [ ] **Алерты** (M14.6): пороги (M14.1) → канал
- [ ] **Runbook** (M14.6): 2 сценария (crash spike, payment failure)

### Шаг 4. Privacy (3ч)
- [ ] **Consent** (M6.6): гео/push/аналитика (явный)
- [ ] **Минимизация** (M14.7): нет PII в Sentry/logs
- [ ] **Удаление** (M14.8): «удалить аккаунт» → данные удалены
- [ ] **Privacy-дисклейшн** (M13.5): для stores

### Шаг 5. Scale: ADR #3 + нагрузка (3ч)
- [ ] **ADR #3** (M14.8): монолит + «когда делить» → `docs/adr/0003-backend-arch.md`
- [ ] **k6** (M14.8): P95 < 300мс при X RPS
- [ ] **WebSocket**: N соединений
- [ ] **Где предел** → ADR #3

### Шаг 6. Release-операции (2ч)
- [ ] **Staged rollout** (M14.9): 5% → 100% (по метрикам)
- [ ] **Canary-метрики** (M14.1): зелёные/красные
- [ ] **Rollback** (M13.4): продемонстрировать (OTA-роллбэк)
- [ ] **Postmortem** (M14.6): 1 блеймлесс

### Шаг 7. Отчёт (2ч) — `docs/progress/M14.md`
- Метрики: crash-free, error rate, P95 (до/после, цели)
- Воронка (M14.3): где утечка
- Sentry: issue + release tracking
- Privacy: consent + удаление + дисклейшн
- ADR #3 + нагрузка (предел)
- Oшибки: 3+

---

## ✅ Практическое задание модуля

### Задача: «Наблюдаемость + метрики-цели + privacy + scale + release-опы»

**1. Sentry (6ч)** — crash+perf+release; **без PII**; issue

**2. Аналитика (4ч)** — события; **воронка**; retention

**3. Logs+метрики+алерты (4ч)** — структурные; метрики; **алерты**; 2 runbook

**4. Privacy (3ч)** — consent; минимизация; удаление; дисклейшн

**5. Scale (3ч)** — **ADR #3**; k6 (P95); WS; **предел**

**6. Release-опы (2ч)** — staged rollout; canary-метрики; rollback; postmortem

**7. Отчёт (2ч)**

**Критерии (из PROJECT):**
- [ ] **Crash-free > 99.5%**, **error rate < 0.5%**, **P95 API < 300мс** (Sentry)
- [ ] **Воронка** (M14.3) видна; retention D7
- [ ] **Алерты** по целям + 2 runbook
- [ ] **Privacy**: consent + минимизация + удаление + дисклейшн
- [ ] **ADR #3** (бэкенд) + нагрузочные (P95 при X RPS) + **предел**
- [ ] **Staged rollout** + canary-метрики + rollback
- [ ] **Postmortem** (blameless)

---

## 📦 Итог

**RideGo в сторах** (M13) — и вы **следите** (Sentry + analytics + logs), **реагируете** (алерты + runbook + rollback), **защищаете** (privacy), **масштабируете** (ADR #3). **Приложение живёт** — и вы управляете ним. **M15** — последние штрихи → v1.0.

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| PII в Sentry | Нарушение GDPR | Анонимный id (M14.2) |
| Нет release tracking | Не знаете, в какой версии баг | release (M13) (M14.2) |
| Алерт на всё | Шум → игнор | Пороги по целям (M14.6) |
| Нет runbook | Паника при инциденте | Runbook (M14.6) |
| Нет консента | Отказ store / GDPR | Явный consent (M6.6) |
| Нет «удалить аккаунт» | GDPR | Удаление (M14.8) |
| Rollout 100% сразу | Баг → всем | Staged (M14.9) |
| Монолит → «всегда масштабируется» | Ложь | ADR #3: предел (M14.8) |

---

## 💡 Советы на модуль

- **Наблюдаемость = 3 кита.** Sentry (where) + analytics (what) + logs (how).
- **Цели — цифры.** Не «быстро», а «P95 < 300мс».
- **Privacy — с первого дня.** Не «доделать». (M6.6)
- **Scale = ADR, не интуиция.** Монолит **до** предела, потом — делить (M14.8).
- **Release-опы = метрики + runbook + rollback.** Не «надеюсь, всё ок».

**[→ Модуль 15: Hardening и v1.0](../Module_15_Hardening_V1/content.md)**
