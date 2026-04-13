# Salary Configuration — Полная спецификация

> **"Хардкод не должен быть"** — все параметры расчёта зарплат настраиваются через интерфейс.

---

## Таблица `company_salary_settings`

Одна запись = одна настройка для конкретной **компании** + **роли**.

```
UNIQUE (company, role)
```

---

## Полная схема параметров

### 1. Идентификация

| Поле | Тип | Обязательно | Описание |
|---|---|---|---|
| `company` | string | ✅ | Название компании |
| `role` | `op` / `closer` / `rop` / `dop` / `admin_sales` | ✅ | Роль сотрудника |

---

### 2. Период расчёта

| Поле | Тип | По умолчанию | Описание |
|---|---|---|---|
| `period_type` | `day` / `month` / `per_check` | `day` | Как группируются продажи для подбора тира |

**Значения:**
- `day` — тир определяется по сумме продаж за **каждый день отдельно**
- `month` — тир определяется по **итогу за весь месяц**
- `per_check` — тир определяется по сумме **каждого чека отдельно**

**Примеры:**
- Акцент ОП: `month` (1.5 млн / 2.5 млн / 3.5 млн)
- Эксперт Closer: `per_check` (200к / 300к / 500к за чек)
- Туркей Closer: `per_check` (135к за чек)

---

### 3. Ступени (тиры)

Массив `salary_tiers`. Каждый элемент:

| Поле | Тип | Обязательно | Описание |
|---|---|---|---|
| `from` | ₸ (число ≥ 0) | ✅ | Нижняя граница суммы |
| `to` | ₸ / `null` | ✅ | Верхняя граница (`null` = ∞, последний тир) |
| `pct` | % (0–100) | ✅ | Процент от продаж в этом диапазоне |
| `base_salary` | ₸ (число ≥ 0) | — | Фиксированный оклад при достижении этого тира |

**Правила:**
- Диапазоны не должны перекрываться
- Последний тир обязан иметь `"to": null`
- Если `base_salary` не указан — считается как `0`
- Если тиры пустые (`[]`) — % по тирам не начисляется (для ДОП / Admin Sales)

**Пример — Alpha ОП:**
```json
[
  { "from": 0,       "to": 1500000, "pct": 6, "base_salary": 0     },
  { "from": 1500000, "to": 3000000, "pct": 7, "base_salary": 50000 },
  { "from": 3000000, "to": 5000000, "pct": 8, "base_salary": 0     },
  { "from": 5000000, "to": null,    "pct": 9, "base_salary": 0     }
]
```
→ При касса 1.5 млн → 7% + **50 000 ₸ оклад**

---

### 4. Пробные уроки

| Поле | Тип | По умолчанию | Описание |
|---|---|---|---|
| `trial_enabled` | bool | `true` | Считать ли пробные уроки вообще |
| `trial_min_amount` | ₸ | `0` | Минимальная сумма чека чтобы пробный засчитался |
| `trial_bonus_amount` | ₸ | `0` | Фиксированный бонус за каждый засчитанный пробный |
| `trial_in_day_sum` | bool | `false` | Включать ли пробный в дневную/месячную сумму для тиров |

**Примеры:**

| Компания / Роль | `trial_enabled` | `trial_min_amount` | `trial_bonus_amount` | `trial_in_day_sum` |
|---|---|---|---|---|
| Акцент Closer | `false` | — | — | — |
| Туркей МОП | `false` | — | — | — |
| Акцент ОП | `true` | `0` | `100` | `false` |
| Эксперт Closer | `true` | `0` | `400` | `false` |
| Alpha Closer | `true` | `0` | `400` | `false` |

> **Ранее был хардкод:** `amount >= 990` — теперь `trial_min_amount`

---

### 5. Бонус за каждый чек

| Поле | Тип | По умолчанию | Описание |
|---|---|---|---|
| `per_sale_bonus` | ₸ | `0` | Фиксированная сумма, прибавляемая к **каждой** продаже (не только пробной) |

**Примеры:**
- Туркей Closer: `per_sale_bonus: 1000` (каждый чек +1 000 ₸)
- Alpha Closer v1: `per_sale_bonus: 500`
- Акцент: `per_sale_bonus: 0`

> **Отличие от `trial_bonus_amount`:** `per_sale_bonus` — за любой чек, `trial_bonus_amount` — только за пробный.

---

### 6. Тип продажи "direct"

| Поле | Тип | По умолчанию | Описание |
|---|---|---|---|
| `direct_pct` | % | `10` | Процент от чека с типом `direct` (считается немедленно, без тиров) |

> **Ранее был хардкод:** `lineAmount * 0.10` — теперь `direct_pct`

---

### 7. Командный бонус (РОП / ДОП / Admin Sales)

| Поле | Тип | По умолчанию | Описание |
|---|---|---|---|
| `team_pct` | % / `null` | `null` | Flat % от суммарных продаж **всей команды** (вместо или вместе с тирами) |
| `sub_rop_pct` | % | `1` | % от продаж суб-РОПов (РОПы, подчинённые этому РОПу) |
| `own_sales_role` | `op` / `closer` / `null` | `op` | По какой логике считать **личные продажи** РОПа. `null` = не считать |

> **Ранее были хардкоды:** `sub_rop_pct = 0.01` и `own_sales_role = 'op'`

**Примеры:**

| Компания / Роль | `team_pct` | `sub_rop_pct` | `own_sales_role` |
|---|---|---|---|
| Акцент РОП | `2` | `1` | `op` |
| Эксперт РОП | `2` | `null` | `null` |
| Туркей РОП | `2` | `null` | `null` |
| Акцент ДОП | `1` | `null` | `null` |
| Alpha ДОП | `1.5` | `null` | `null` |

---

### 8. Порог оклада РОП (прогрессивный оклад)

Для РОПов с окладом зависящим от кассы — используется `base_salary` в тирах (см. блок 3).

**Пример — Alpha РОП:**
```json
[
  { "from": 0,       "to": 7000000, "pct": 1, "base_salary": 0      },
  { "from": 7000000, "to": null,    "pct": 2, "base_salary": 200000 }
]
```
→ При 7 млн+ → 2% + **200 000 ₸ оклад**

---

## Полный JSON — пример настройки

```json
{
  "company": "Akcent",
  "role": "op",

  "period_type": "month",

  "salary_tiers": [
    { "from": 0,       "to": 1500000, "pct": 5, "base_salary": 0 },
    { "from": 1500000, "to": 2500000, "pct": 6, "base_salary": 0 },
    { "from": 2500000, "to": 3500000, "pct": 7, "base_salary": 0 },
    { "from": 3500000, "to": null,    "pct": 8, "base_salary": 0 }
  ],

  "trial_enabled": true,
  "trial_min_amount": 0,
  "trial_bonus_amount": 100,
  "trial_in_day_sum": false,

  "per_sale_bonus": 0,
  "direct_pct": 10,

  "team_pct": null,
  "sub_rop_pct": 1,
  "own_sales_role": "op"
}
```

---

## Матрица — кто что использует

| Параметр | ОП | Closer | РОП | ДОП | Admin Sales |
|---|---|---|---|---|---|
| `period_type` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `salary_tiers` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `base_salary` (в тире) | ✅ | ❌ | ✅ | ✅ | ❌ |
| `trial_enabled` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `trial_min_amount` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `trial_bonus_amount` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `trial_in_day_sum` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `per_sale_bonus` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `direct_pct` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `team_pct` | ❌ | ❌ | ✅ | ✅ | ✅ |
| `sub_rop_pct` | ❌ | ❌ | ✅ | ❌ | ❌ |
| `own_sales_role` | ❌ | ❌ | ✅ | ❌ | ❌ |

---

## Изменения в схеме БД

```sql
ALTER TABLE company_salary_settings
  ADD COLUMN IF NOT EXISTS trial_enabled      BOOLEAN        NOT NULL DEFAULT true,
  ADD COLUMN IF NOT EXISTS trial_min_amount   NUMERIC(12,2)  NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS trial_in_day_sum   BOOLEAN        NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS per_sale_bonus     NUMERIC(12,2)  NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS direct_pct         NUMERIC(5,2)   NOT NULL DEFAULT 10,
  ADD COLUMN IF NOT EXISTS team_pct           NUMERIC(5,2),
  ADD COLUMN IF NOT EXISTS sub_rop_pct        NUMERIC(5,2)   NOT NULL DEFAULT 1,
  ADD COLUMN IF NOT EXISTS own_sales_role     VARCHAR(50)    DEFAULT 'op';
-- base_salary добавляется внутрь JSON salary_tiers (уже JSONB)
```

---

## Файлы которые нужно изменить

| Файл | Что меняется |
|---|---|
| `db_schema.sql` | Новые колонки в `company_salary_settings` |
| `src/helpers/salary.js` | Убрать все хардкоды, использовать настройки |
| `src/routes/salarySettings.js` | Валидация и CRUD новых полей |
| `src/pages/SalarySettings.jsx` | UI для всех новых полей |
