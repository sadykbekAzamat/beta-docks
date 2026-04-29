# Фича: Роль `team_lead` (Тимлид)

## Обзор

Добавление новой роли `team_lead` — промежуточное звено между ОП и РОП.  
Тимлид одновременно **управляет своими ОПами** (как РОП) и **сам делает продажи** (как ОП).  
Тимлид подчиняется РОПу — сам выбирает своего РОПа при регистрации.

---

## 1. Иерархия ролей (после изменений)

```
DOP
 └── ROP
      ├── TEAM_LEAD
      │    └── OP (подчинённые тимлида)
      └── OP (подчинённые РОПа напрямую)
```

ОП при регистрации выбирает **либо РОПа, либо Тимлида** в качестве своего руководителя.

---

## 2. Требования по ролям

| Характеристика | ROP | TEAM_LEAD | OP |
|---|---|---|---|
| Управляет ОПами | ✅ | ✅ | — |
| Делает личные продажи | опционально (own_sales_role) | ✅ всегда | ✅ |
| Выбирает РОПа | — | ✅ | — |
| Выбирает РОПа / Тимлида | — | — | ✅ |
| Собственные настройки зарплаты | ✅ | ✅ | ✅ |
| Видит своих ОПов на /sales | ✅ | ✅ | — |

---

## 3. Изменения в БД

### 3.1 `role_enum` — добавить значение `team_lead`

```sql
ALTER TYPE role_enum ADD VALUE IF NOT EXISTS 'team_lead';
```

### 3.2 `sales_users` — добавить колонку `tl_id`

Хранит ссылку на тимлида для ОПов, которые выбрали тимлида вместо РОПа.

```sql
ALTER TABLE sales_users ADD COLUMN IF NOT EXISTS tl_id INTEGER REFERENCES users(id);
```

> Логика: у ОПа либо `rop_id` (выбрал РОПа), либо `tl_id` (выбрал тимлида). Оба не заполнены быть не должны.

### 3.3 `company_salary_settings` — добавить колонку `tl_pct`

Процент, который РОП получает от продаж своих тимлидов.

```sql
ALTER TABLE company_salary_settings
  ADD COLUMN IF NOT EXISTS tl_pct NUMERIC(5,2) DEFAULT NULL;
```

Аналогично `sub_rop_pct` — но для тимлидов. Применяется только в расчёте зарплаты РОПа.

---

## 4. Изменения в Backend

### 4.1 `db_schema.sql`

- Добавить `'team_lead'` в `role_enum`
- Добавить `tl_id INTEGER REFERENCES users(id)` в `sales_users`
- Добавить `tl_pct NUMERIC(5,2)` в `company_salary_settings`

### 4.2 `src/helpers/salary.js`

#### `getSalarySettings`
- Добавить поле `tl_pct` в возвращаемый объект настроек:
  ```js
  tl_pct: row.tl_pct != null ? Number(row.tl_pct) : null,
  ```

#### Новая функция `calculateTeamLeadSalary`
- Аналогична `calculateRopSalary`
- Личные продажи — как ОП (`calculateOpSalary`)
- Командные — считает сумму продаж своих ОПов (по `tl_id = teamLeadUserId`)
- Применяет `team_pct` или тиры к командным продажам

```js
async function calculateTeamLeadSalary(tlUserId, company, dateFrom, dateTo, monthYear, salarySettingId) {
  // 1. Получить настройки для роли 'team_lead'
  // 2. Рассчитать личные продажи как OP (calculateOpSalary sub-call)
  // 3. Суммировать продажи ОПов у которых tl_id = tlUserId
  // 4. Применить team_pct или тиры к командным продажам
  // 5. Вернуть { total_sales, salary, own_salary, team_bonus, ... }
}
```

#### `calculateRopSalary` — добавить `tl_pct` бонус
- Запросить продажи тимлидов РОПа (у тимлидов `rop_id = ropUserId`)
- Посчитать суммарные продажи тимлидов (как персональные, так и командные)
- Применить `settings.tl_pct` к этой сумме

```js
// Добавить в calculateRopSalary:
const { rows: tlSales } = await query(
  `SELECT COALESCE(SUM(frl.amount), 0) AS total
   FROM finance_report_lines frl
   JOIN manager_reports mr ON ...
   JOIN sales_users su ON su.user_id = mr.user_id
   WHERE su.rop_id = $1 AND su.position = 'team_lead'
     AND mr.status = 'verified' AND frl.category <> 'Возврат'
     ${dateFilter}`,
  params
);
const tlBonus = settings.tl_pct ? Number(tlSales[0].total) * settings.tl_pct / 100 : 0;
// включить tlBonus в итоговую зарплату РОПа
```

#### `SUMMARY_ROLES` — добавить `'team_lead'`

```js
const SUMMARY_ROLES = ['finance', 'admin', 'rop', 'dop', 'admin_sales', 'team_lead'];
```

### 4.3 `src/routes/sales.js`

#### `getEffectiveCompanyFilter`
- Добавить `team_lead` в список ролей с принудительной компанией:
  ```js
  if (req.user.role === 'dop' || req.user.role === 'rop' || req.user.role === 'team_lead') { ... }
  ```

#### `GET /op-summary`
- Для `team_lead` добавить фильтр: ОПы у которых `su.tl_id = req.user.id` (вместо rop_id)
  ```js
  const tlFilter = req.user.role === 'team_lead' ? req.user.id : null;
  // в handleOpSummary:
  if (tlFilter) {
    params.push(tlFilter);
    conditions.push(`su.tl_id = $${params.length}`);
  }
  ```

#### Новый endpoint `GET /tl-summary`
- Показывает список тимлидов с их зарплатой (аналогично `/rop-summary`)
- Доступен для: `finance`, `admin`, `rop`, `dop`
- Для `rop`: только тимлиды у которых `rop_id = req.user.id`

#### `GET /summary-by-rop`
- РОП теперь видит своих тимлидов (отдельная секция или новое поле)

#### `GET /user-salary`
- При `su.position = 'team_lead'` — вызывать `calculateTeamLeadSalary`

### 4.4 `src/routes/auth.js`

#### `POST /auth/register` (публичная регистрация ОПа)
- Принимать `tl_id` (опционально) вместо или вместе с `rop_id`
- Валидация: если передан `tl_id` — проверить что пользователь существует и роль `team_lead`
- Если передан `tl_id` — в `sales_users` сохранить `tl_id`, `rop_id` = NULL (или заполнить автоматически через `tl.rop_id`)
- Добавить endpoint `GET /auth/team-leads?company=...` — список тимлидов для выбора при регистрации

#### `POST /auth/create-user-extended`
- Для ОПа: принимать `tl_id` (опционально)
- Для `team_lead`: принимать `rop_id` (обязательно)

### 4.5 `src/routes/users.js`

#### `GET /users/merged`
- Добавить `su.tl_id` в SELECT
- Добавить `LEFT JOIN sales_users tl_su ON tl_su.user_id = su.tl_id` для получения имени тимлида

#### `PUT /users/merged/:userId`
- Обрабатывать `tl_id` при обновлении ОПа

### 4.6 `src/routes/salarySettings.js`

#### `POST /salary-settings` и `PUT /salary-settings/:id`
- Принимать `tl_pct` (nullable number) в теле запроса
- Сохранять в БД

---

## 5. Изменения в Frontend

### 5.1 `src/pages/Login.jsx` / `Register.jsx` (регистрация ОПа)

- При выборе роли `op`: показывать два selectable-списка: **"Выберите РОПа"** и **"Выберите Тимлида"**
- Или: один select с объединённым списком (РОПы и Тимлиды), с отображением роли
- Логика: при выборе тимлида — передать `tl_id`, `rop_id = null`; при выборе РОПа — передать `rop_id`, `tl_id = null`
- Запрос: `GET /auth/rops?company=...` (уже есть) + `GET /auth/team-leads?company=...` (новый)

### 5.2 `src/pages/Users.jsx` (создание/редактирование пользователей)

- При создании ОПа: показывать select **"Тимлид"** (в дополнение к существующему select "РОП")
- При создании `team_lead`: показывать select **"РОП"** (обязательно)
- Добавить `team_lead` в список ролей
- В таблице: отображать колонку "Руководитель" (РОП или Тимлид)
- `loadSalarySettings(company, 'team_lead')` при выборе роли team_lead

### 5.3 `src/pages/Sales.jsx`

- Тимлид: всегда вкладка "ОП" (его подчинённые), без переключателя (как РОП)
- Добавить вкладку "Тимлиды" (аналог вкладки "РОПы") — видна для admin/finance/rop/dop
- Для РОПа: в его секции показывать тимлидов + их ОПов

### 5.4 `src/pages/Dashboard.jsx`

- Тимлид — аналогично РОПу: видит карточку "Мои продажи" (личные продажи как ОП)
- Добавить `'team_lead'` в `PERSONAL_SALES_ROLES`

### 5.5 `src/pages/SalarySettings.jsx`

- Добавить поле `tl_pct` (Процент от тимлидов) в форму — отображать только для роли `rop`
- Подпись: _"% от продаж тимлидов РОПа"_

---

## 6. Порядок реализации

| # | Задача | Файлы |
|---|---|---|
| 1 | БД: добавить enum, колонки tl_id и tl_pct | `db_schema.sql` + миграция на сервере |
| 2 | salary.js: getSalarySettings + tl_pct, calculateTeamLeadSalary | `helpers/salary.js` |
| 3 | salary.js: calculateRopSalary += tl_pct бонус | `helpers/salary.js` |
| 4 | sales.js: team_lead фильтры, /tl-summary, /user-salary | `routes/sales.js` |
| 5 | auth.js: регистрация ОПа с tl_id, /team-leads endpoint, create-user-extended | `routes/auth.js` |
| 6 | users.js: tl_id поддержка | `routes/users.js` |
| 7 | salarySettings.js: tl_pct поле | `routes/salarySettings.js` |
| 8 | SalarySettings.jsx: поле tl_pct для роли rop | `pages/SalarySettings.jsx` |
| 9 | Users.jsx: team_lead роль + tl_id select для ОПа | `pages/Users.jsx` |
| 10 | Register.jsx: выбор тимлида при регистрации ОПа | `pages/Register.jsx` (или Login.jsx) |
| 11 | Sales.jsx: тимлид фиксированная вкладка, новая вкладка "Тимлиды" | `pages/Sales.jsx` |
| 12 | Dashboard.jsx: team_lead → PERSONAL_SALES_ROLES | `pages/Dashboard.jsx` |
| 13 | Деплой бэкенда + проверка | `./a.sh` |

---

## 7. API изменения

### Новые endpoints

| Метод | URL | Описание |
|---|---|---|
| GET | `/auth/team-leads?company=...` | Список тимлидов для регистрации ОПа |
| GET | `/sales/tl-summary` | Сводка продаж по тимлидам |

### Изменённые endpoints

| Метод | URL | Что меняется |
|---|---|---|
| POST | `/auth/register` | Принимает `tl_id` (опционально) |
| POST | `/auth/create-user-extended` | Принимает `tl_id` для ОПа, `rop_id` для team_lead |
| GET | `/sales/op-summary` | team_lead фильтрует своих ОПов по `tl_id` |
| GET | `/sales/user-salary` | Для team_lead вызывает `calculateTeamLeadSalary` |
| POST/PUT | `/salary-settings` | Принимает поле `tl_pct` |

---

## 8. Ключевые бизнес-правила

1. **ОП выбирает руководителя** — либо РОП, либо Тимлид. Нельзя выбрать обоих. Нельзя не выбрать никого.
2. **Тимлид обязательно имеет РОПа** — при создании тимлида `rop_id` обязателен.
3. **Тимлид видит только своих ОПов** на странице /sales (фильтр по `tl_id`).
4. **РОП видит тимлидов** на странице /sales отдельной вкладкой (аналог РОПов у ДОПа).
5. **РОП получает `tl_pct`** от суммарных продаж своих тимлидов (личные + командные продажи тимлида).
6. **Тимлид получает `team_pct`** от продаж своих ОПов (настройка в `company_salary_settings` для роли `team_lead`).
7. **Зарплата тимлида** = (личные продажи по настройкам OP) + (командный бонус от своих ОПов).
