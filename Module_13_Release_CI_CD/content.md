# Модуль 13: CI/CD и релизы

**[← Назад к индексу](../index.md)** | **[Модуль 12: Тесты](../Module_12_Testing/content.md)** | **Далее: [Модуль 14: Продакшн](../Module_14_Observability_Release/content.md) →**

---

## 🎯 Цель модуля

- **GitHub Actions**: lint + typecheck + test + e2e на каждый PR
- **EAS Build**: внутренняя → beta → production; **canary** + **hotfix** треки
- **EAS Update (OTA)**: JS-фиксы без store review
- **Semver + changelog + tagging**
- **Store review** (App Store Connect + Play Console)
- **Цели** (из PROJECT): e2e на CI, canary, hotfix < 24ч, OTA для JS-фиксов

**Длительность: ~22 часа.**

---

## 📚 Теория

### 13.1. Пайплайн CI: каждый PR

```
PR → GitHub Actions:
  1. Lint (ESLint + Prettier check)        # 30с
  2. Typecheck (tsc --noEmit)              # 1–2м
  3. Unit + Integration (Jest + coverage)  # 2–5м
  4. Mutation (Stryker, critical paths)    # 2–5м (опц. на main)
  5. e2e (Maestro на эмуляторе/симуляторе) # 5–15м
  → Зелёный = можно мерджить
```

**Якоря качества** (из M1): `pre-commit` (lint-staged + typecheck), **CI повторяет** их + e2e. **Merge** только при зелёном (branch protection).

### 13.2. EAS Build: треки и profile

**EAS Build** — облачная сборка (Apple/Google credentials в облаке, не на вашей машине). **`eas.json`** — profile:

```json
{
  "cli": { "version": ">= 5" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": { "simulator": true },
      "android": { "gradleCommand": ":app:assembleDebug" }
    },
    "preview": {
      "channel": "preview",          # ← M13.3: OTA-канал
      "distribution": "internal",
      "ios": { "simulator": false },
      "android": { "buildType": "apk" }
    },
    "production": {
      "autoIncrement": true,         # ← автобамп версии
      "channel": "production",
      "distribution": "store"
    }
  },
  "submit": {
    "production": { "ios": { "appleTeamId": "..." }, "android": { "track": "production" } }
  }
}
```

**Треки:**
- **`development`** — dev client (M7), локально/внутренне
- **`preview`** (canary) — **internal**, для команды/QA, **всегда**
- **`production`** — store (TestFlight + Play internal/production)

```bash
eas build --profile development --platform ios      # dev client
eas build --profile preview --platform all          # canary → TestFlight internal + Play internal
eas build --profile production --platform all       # store
```

### 13.3. EAS Update (OTA): JS-фиксы без store

**Отдельные дистрибуции:**
- **Натив (APK/IPA)** — store review (M13.5), **медленно**
- **JS bundle (OTA)** — **EAS Update** → **без store review**, за минуты

```
eas update --channel production --message "Fix: цена не округлялась"
→ все клиенты (на этой нативной версии) обновляют JS при запуске
```

**Правила OTA:**
- **Только JS-изменения.** Нативные (новый модуль, permission) → **новый нативный билд** (M13.2)
- **Каналы** (M13.2): `production` + `preview` (canary). Canary — для раннего обнаружения
- **Роллбэк**: `eas update --branch <prev>` (M13.4) — **обратите** OTA за минуты
- **Runtime version**: OTA-бандл совместим с нативной версией. Новый нативный → новое runtime

> ⚠️ **OTA — не замена store.** Для «критичных» (безопасность, натив) — нативный hotfix (M13.4). Для JS-багов — OTA.

### 13.4. Версионирование: semver + canary + hotfix

**Semver (M13.6):** `major.minor.patch`
```
1.0.0 → 1.1.0 (фича) → 1.1.1 (hotfix) → 2.0.0 (breaking)
```
- `autoIncrement: true` (eas.json) — EAS сам bump-ит
- **Changelog** (M13.6): `CHANGELOG.md` (конвенция: Added/Changed/Fixed)

**Canary-трек (M13.2):** `preview`-канал → **всегда** билдится из `main`. Команда/QA — на canary **до** production. **Раннее обнаружение** багов.

**Hotfix < 24ч (цель PROJECT):**
1. **JS-баг** → `eas update` (OTA) → **минуты** (канал production)
2. **Натив-баг** → ветка `hotfix/x.y.z` → `eas build --profile production` → store (часы)
3. **Роллбэк** OTA: `eas update --branch <prev>` (если OTA сломал)

**Ветки (Git Flow):**
```
main ──────────────── (production)
  │
  ├── release/1.1.0 ── (подготовка релиза, QA)
  │
  └── hotfix/1.1.1 ─── (критичный фикс)
```
(Для курса: `main` (prod) + `release/*` + `hotfix/*` + фичи в `feat/*` → PR → main.)

### 13.5. Store review: App Store + Play Console

**App Store Connect (iOS):**
1. `eas build --profile production --platform ios` → IPA
2. `eas submit --platform ios` → **App Store Connect**
3. Заполнить: скриншоты, описание, **категория**, age rating, **privacy** (M14.7!)
4. **TestFlight** (внутренний: 100 чел.; внешний: до 10 000) → QA → **Submit for Review** (1–3 дня)
5. **Review guidelines** (M13.6): не нарушайте (2.1: не «обходите», 4.x: платёж)

**Play Console (Android):**
1. `eas build --profile production --platform android` → AAB
2. `eas submit --platform android` → **Play Console**
3. **Трэк**: Internal (быстро) → **Closed testing** (список) → **Open testing** (20%) → **Production** (staged rollout)
4. **Staged rollout** (M13.5): 5% → 20% → 50% → 100% (по метрикам: crash-free M14)

**Общие:**
- **Скриншоты/видео** (обоих stores)
- **Privacy** (M14.7): disclosure (гео, биометрия, платежи, push)
- **Content rating** (12+: транспорт)
- **Realtime-метки** (M14.8)

### 13.6. Conventions: changelog + релиз-чеклист

```markdown
# CHANGELOG.md
## [1.1.0] - 2024-..
### Added
- ...
### Changed
- ...
### Fixed
- ...
```

**Релиз-чеклист (M13):**
- [ ] `main` зелёный (CI: lint+type+test+e2e)
- [ ] Version bump (semver) + changelog
- [ ] `git tag v1.1.0`
- [ ] **Canary** (`preview`): команда — QA (24ч)
- [ ] **Production** (`eas build --profile production`)
- [ ] **TestFlight** (внутренний) → **Submit** (iOS)
- [ ] **Play** (internal → staged rollout)
- [ ] **OTA-канал** `production` — готов
- [ ] **Smoke** на device (M12 e2e)
- [ ] **Sentry** (M14): следим за error rate первые 24ч

### 13.7. CI: GitHub Actions (скелет)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage   # unit + integration
      # (mutation — только на main, отдельный job)

  e2e:
    runs-on: macos-latest            # iOS simulator (или android: emulator)
    needs: quality
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx expo prebuild --platform ios
      - run: xcodebuild -workspace ios/RideGo.xcworkspace -scheme RideGo -destination 'platform=iOS Simulator,name=iPhone 15'
      - uses: reactivecircus/android-emulator-runner@v2   # для Android e2e
        with: api-level: 34
      - run: npx maestro test e2e/
```

**EAS Build на `main` (canary):**
```yaml
  canary:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - uses: expo/expo-github-action@v8
        with: eas-version: latest
      - run: eas build --profile preview --platform all --non-interactive --no-wait
```

### 13.8. Метрики релиза (связь с M14)

**После релиза** (M14): **crash-free > 99.5%**, **error rate < 0.5%** (Sentry), **staged rollout** — по этим метрикам. Если **crash > 0.5%** → **остановить rollout** (Play) / **отозвать** (App Store) → **hotfix** (M13.4).

---

## 🛠️ Практика на RideGo (шаги)

### Шаг 1. CI: GitHub Actions (6ч)
- [ ] `ci.yml`: lint + typecheck + test (coverage) (M13.7)
- [ ] **e2e** на sim/emulator (Maestro, M12.6)
- [ ] **Branch protection**: merge только при зелёном
- [ ] Прогнать на PR: зелёный/красный (сломать → ловит)

### Шаг 2. EAS: profile + треки (6ч)
- [ ] `eas.json`: development / preview (canary) / production (M13.2)
- [ ] `eas build --profile preview --platform all` → TestFlight internal + Play internal
- [ ] **Canary**: команда на canary (M13.3)

### Шаг 3. EAS Update: OTA (4ч)
- [ ] `eas update` setup: каналы (production + preview) (M13.3)
- [ ] **Продемонстрировать**: JS-фикс → `eas update --channel production` → клиент получил (без store)
- [ ] **Роллбэк**: `eas update --branch <prev>` (M13.4)

### Шаг 4. Версии + changelog + релиз v0.3.0 (4ч)
- [ ] Semver + `CHANGELOG.md` + `git tag` (M13.6)
- [ ] **Релиз v0.3.0** (M9): canary → QA → production → stores (M13.5)
- [ ] **Staged rollout** (Play): 5% → 100% (по метрикам)

### Шаг 5. Hotfix-сценарий (2ч)
- [ ] **Сценарий**: «найден JS-баг» → `eas update` (OTA) → **< 24ч** (показать тайминг)
- [ ] **Сценарий**: «натив-баг» → `hotfix` ветка → `eas build --profile production`
- [ ] **Роллбэк** OTA — продемонстрировать

### Шаг 6. Store review (2ч)
- [ ] App Store Connect: скриншоты, описание, privacy (M14.7), TestFlight
- [ ] Play Console: internal → staged
- [ ] (Если store нет — **подготовить** все артефакты: иконки, описания, privacy)

### Шаг 7. Отчёт (2ч) — `docs/progress/M13.md`
- Пайплайн (CI): скрин (зелёный/красный)
- Треки: canary + production
- OTA: скрин «улучшили JS без store»
- Релиз v0.3.0: canary → prod → stores
- Hotfix: тайминг (< 24ч)
- Ошибки: 3+

---

## ✅ Практическое задание модуля

### Задача: «CI зелёный + canary + production + OTA + hotfix < 24ч + v0.3.0»

**1. CI (6ч)** — lint+type+test+e2e на PR; branch protection

**2. EAS (6ч)** — profile (dev/canary/prod); canary на `main`

**3. OTA (4ч)** — каналы; JS-фикс без store; роллбэк

**4. Релиз (4ч)** — semver+changelog+tag; **v0.3.0**: canary→prod→stores; staged rollout

**5. Hotfix (2ч)** — JS-баг → OTA < 24ч; натив-баг → build; роллбэк

**6. Store (2ч)** — App Store + Play (артефакты; TestFlight)

**7. Отчёт (2ч)**

**Критерии (из PROJECT):**
- [ ] **CI**: каждый PR → lint+type+test+e2e (зелёный); merge-защита
- [ ] **Canary**: билдится с `main`, команда на canary
- [ ] **Production**: TestFlight + Play (staged rollout)
- [ ] **OTA**: JS-фикс без store review; роллбэк
- [ ] **Hotfix < 24ч**: (JS → OTA; натив → build)
- [ ] **v0.3.0** (M9) — released (stores/TestFlight)
- [ ] Semver + changelog + tag на релизе

---

## 📦 Итог

**Путь кода → пользователи** автоматизирован: PR → CI (качество) → canary (QA) → production (stores) → **OTA** (JS-фиксы) → **hotfix** (критичные). **v0.3.0** в сторах. **M14** — следим, что после релиза.

---

## 🐛 Частые ошибки

| Ошибка | Следствие | Решение |
|--------|-----------|---------|
| Merge без CI | Баги в main | Branch protection (M13.1) |
| e2e на PR всегда | Медленно | e2e на PR (кратко) + main (полный) |
| OTA для нативных изменений | Сломано (runtime) | Натив = новый билд (M13.3) |
| Нет canary | Баги ловятся в prod | Canary-трек (M13.2) |
| Роллбэк — только store | Часы/дни | OTA-роллбэк (M13.4) |
| Нет privacy в store | Отказ review | Privacy (M14.7) |
| Rollout 100% сразу | Баг → всем | Staged rollout (M13.5) |
| Нет changelog | Неясно что в версии | Conventions (M13.6) |

---

## 💡 Советы на модуль

- **CI = якорь.** Не мерджим красный.
- **Canary — сетка безопасности.** Команда ловит баги **до** production.
- **OTA ≠ store.** JS → OTA (минуты); натив → билд (часы).
- **Hotfix < 24ч** — цель. OTA делает это **минуты**.
- **Staged rollout** — не 100% сразу. По метрикам (M14).

**[→ Модуль 14: Продакшн](../Module_14_Observability_Release/content.md)**
