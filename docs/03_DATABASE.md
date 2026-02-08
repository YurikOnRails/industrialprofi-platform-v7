# 🗄️ База Данных IndustrialPROFI

> **СУБД:** PostgreSQL 16+  
> **ORM:** ActiveRecord (Rails 8)  
> **Миграции:** db/migrate/  
> **Audit Trail:** paper_trail gem (версионирование изменений)

---

## 📊 Полная Схема БД

### Легенда типов данных

```
🌍 Глобальные таблицы (без organization_id)
🔒 Тенантные таблицы (с organization_id)
⚙️ Инфраструктурные таблицы (Solid Stack)
```

---

## 🌍 Глобальные Таблицы

### 1. `permit_templates` (Каталог Типовых Допусков)

**Назначение:** Глобальный справочник допусков для СНГ. Создается админом платформы.

```ruby
create_table :permit_templates do |t|
  t.string :title, null: false              # "Электробезопасность II группа"
  t.string :code, null: false, index: true  # "ELEC_SAFETY_II"
  t.text :description
  
  # Локализация
  t.string :country_code, default: 'RU'     # RU, KZ, BY
  t.string :regulatory_body                 # "Ростехнадзор", "Минтруд"
  
  # Сроки действия
  t.integer :expiration_months, null: false # 12, 24, 36
  t.boolean :requires_renewal, default: true
  
  # Категория
  t.string :category                        # "safety", "welding", "electrical"
  
  # Метаданные
  t.integer :usage_count, default: 0        # Сколько раз использован
  t.boolean :is_active, default: true
  
  t.timestamps
end

add_index :permit_templates, [:country_code, :code], unique: true
```

**Примеры записей:**
```ruby
PermitTemplate.create!(
  title: "Электробезопасность II группа",
  code: "ELEC_SAFETY_II",
  country_code: "RU",
  regulatory_body: "Ростехнадзор",
  expiration_months: 12,
  category: "electrical"
)

PermitTemplate.create!(
  title: "НАКС (Неразрушающий контроль)",
  code: "NAKS_NDT",
  country_code: "RU",
  regulatory_body: "НАКС",
  expiration_months: 24,
  category: "welding"
)
```

---

## 🔒 Организации и Пользователи

### 2. `organizations` (Компании-Клиенты)

```ruby
create_table :organizations do |t|
  t.string :name, null: false                    # "Северсталь", "АвтоВАЗ"
  t.string :slug, null: false, index: { unique: true }
  
  # Subscription
  t.string :plan_type, default: 'trial'          # trial, starter, professional, enterprise
  t.string :subscription_status, default: 'active' # active, suspended, cancelled
  t.datetime :trial_ends_at
  
  # Billing
  t.string :stripe_customer_id
  t.integer :employee_limit, default: 10         # Лимит по тарифу
  
  # On-Premise Support (для будущего)
  t.string :deployment_type, default: 'cloud'    # cloud, self_hosted
  t.string :license_key, index: { unique: true }
  
  # Настройки
  t.string :timezone, default: 'Europe/Moscow'
  t.string :locale, default: 'ru'
  
  t.timestamps
end
```

### 3. `job_titles` (Должности) 👔 NEW

**Назначение:** Справочник должностей организации с обязательными допусками и привязкой к картам профессий.

```ruby
create_table :job_titles do |t|
  t.references :organization, null: false, foreign_key: true
  
  # Основная информация
  t.string :title, null: false                # "Сварщик 5 разряда"
  t.text :description                         # Описание обязанностей
  
  # Карта профессии (опционально, для развития сотрудника)
  t.references :roadmap, foreign_key: true, null: true
  
  # Обязательные допуски (юридические требования)
  t.jsonb :required_permit_template_ids, default: []  # [1, 5, 7] - ID из permit_templates
  
  t.timestamps
end

add_index :job_titles, [:organization_id, :title], unique: true
add_index :job_titles, :roadmap_id
```

**Примеры:**
```ruby
JobTitle.create!(
  organization_id: 1,
  title: "Сварщик 5 разряда",
  description: "Выполнение сварочных работ повышенной сложности",
  roadmap_id: 5,  # Опционально: Roadmap "Сварщик" для развития
  required_permit_template_ids: [1, 5, 7]  # НАКС, Электро II, Газосварщик
)

# Минимальная версия (только обязательные допуски)
JobTitle.create!(
  organization_id: 1,
  title: "Электрик 4 разряда",
  required_permit_template_ids: [2, 3],  # Электро III, Электро IV
  roadmap_id: nil  # Можно оставить пустым
)
```

**Логика:**
- `required_permit_template_ids` — массив ID допусков, **обязательных** для должности (юридика)
- `roadmap_id` — рекомендуемая карта развития (опционально, для обучения)
- Автоназначение: при назначении сотрудника на должность → автоматически назначается roadmap (если указан)

---

### 4. `users` (Сотрудники)

```ruby
create_table :users do |t|
  t.references :organization, null: true, foreign_key: true # NULL = super-admin
  
  # Аутентификация (Rails 8 Authentication)
  t.string :email, null: false
  t.string :password_digest, null: false
  
  # Профиль
  t.string :full_name
  t.string :role, default: 'employee'            # employee, manager, owner
  t.string :department                           # "Сварочный цех", "ОТК"
  t.references :job_title, foreign_key: true     # 👔 Связь с должностью
  
  # ⭐ Интеграция с 1С:ЗУП (MVP+)
  t.string :external_1c_id, index: true          # ID сотрудника в 1С
  t.datetime :synced_from_1c_at                  # Последняя синхронизация
  t.jsonb :metadata, default: {}                 # Дополнительные поля из 1С
  
  # Метаданные
  t.datetime :last_sign_in_at
  t.string :locale, default: 'ru'
  
  t.timestamps
end

add_index :users, :email, unique: true
add_index :users, [:organization_id, :email]
add_index :users, [:organization_id, :job_title_id]  # 👔 NEW
add_index :users, [:organization_id, :external_1c_id], unique: true, where: "external_1c_id IS NOT NULL"
```

**⭐ Новое для 1С интеграции:**
- `external_1c_id` — связь с `Справочник.Сотрудники` в 1С
- `synced_from_1c_at` — timestamp последней синхронизации
- `metadata` — JSONB поле для хранения дополнительных данных из 1С (табельный номер, подразделение, etc)

**Пример metadata:**
```json
{
  "personnel_number": "00001234",
  "1c_department_guid": "a3f4e5d6-1234-5678-abcd-ef1234567890",
  "1c_position_guid": "b4f5e6d7-2345-6789-bcde-f12345678901",
  "hire_date": "2020-01-15"
}
```

**⭐ Новое для 1С интеграции:**
- `external_1c_id` — связь с `Справочник.Сотрудники` в 1С
- `synced_from_1c_at` — timestamp последней синхронизации
- `metadata` — JSONB поле для хранения дополнительных данных из 1С (табельный номер, подразделение, etc)

**Пример metadata:**
```json
{
  "personnel_number": "00001234",
  "1c_department_guid": "a3f4e5d6-1234-5678-abcd-ef1234567890",
  "1c_position_guid": "b4f5e6d7-2345-6789-bcde-f12345678901",
  "hire_date": "2020-01-15"
}
```

**Роли:**
- `employee` — обычный сотрудник (читает roadmaps, отмечает прогресс)
- `manager` — руководитель отдела (+ редактирует roadmaps, видит прогресс отдела, **управляет должностями**)
- `owner` — владелец организации (+ управление подпиской, приглашения)

**👔 Связь с должностью:**
- При назначении сотруднику `job_title_id` → автоматически назначается roadmap (если указан в должности)
- Логика проверки compliance: у сотрудника должны быть все допуски из `job_title.required_permit_template_ids`

### 5. `sessions` (Rails 8 Authentication)

```ruby
create_table :sessions do |t|
  t.references :user, null: false, foreign_key: true
  t.string :ip_address
  t.string :user_agent
  
  t.timestamps
end

add_index :sessions, :user_id
```

---

## 📚 Roadmaps и Навыки

### 6. `roadmaps` (Карты Профессий)

**Ключевая логика:** Roadmap может быть публичным (шаблон) или приватным (копия компании).

```ruby
create_table :roadmaps do |t|
  t.references :organization, null: true, foreign_key: true # NULL = публичный roadmap
  t.references :forked_from, null: true, foreign_key: { to_table: :roadmaps }
  
  # Основная информация
  t.string :title, null: false                   # "Сварщик", "Электрик"
  t.string :slug, null: false                    # "welder", "electrician"
  t.text :description
  
  # Видимость
  t.string :visibility, default: 'private'       # public, private, unlisted
  t.boolean :is_template, default: false         # Можно ли форкать
  
  # UI Настройки
  t.string :theme_color, default: '#3b82f6'      # Цвет темы
  t.string :icon                                 # Emoji или Font Awesome код
  
  # Статистика
  t.integer :fork_count, default: 0              # Сколько раз скопирован
  t.integer :skills_count, default: 0            # Counter cache
  
  t.timestamps
end

add_index :roadmaps, [:organization_id, :slug], unique: true
add_index :roadmaps, :visibility
add_index :roadmaps, :is_template
```

**Примеры:**
```ruby
# Публичный roadmap (создан админом платформы)
Roadmap.create!(
  organization_id: nil,
  title: "Сварщик",
  slug: "welder",
  visibility: "public",
  is_template: true
)

# Приватная копия (создана компанией)
Roadmap.create!(
  organization_id: 123,
  forked_from_id: 1,
  title: "Сварщик (для АвтоВАЗа)",
  slug: "welder-avtovaz",
  visibility: "private"
)
```

### 7. `skills` (Узлы Графа)

**Важно:** Skill может быть обычным навыком ИЛИ ссылкой на permit template.

```ruby
create_table :skills do |t|
  t.references :roadmap, null: false, foreign_key: true
  t.references :permit_template, null: true, foreign_key: true # Если это допуск
  
  # Уникальный ключ узла в графе
  t.string :key, null: false                     # "mig-welding", "electrical-safety-2"
  
  # Основная информация
  t.string :title, null: false                   # "MIG Сварка"
  t.text :description
  
  # Координаты на графе
  t.float :position_x, default: 0
  t.float :position_y, default: 0
  t.boolean :position_locked, default: false     # true = ручное позиционирование
  
  # Тип узла
  t.string :skill_type, default: 'skill'         # skill, permit, milestone
  
  # Категория (для группировки в UI)
  t.string :category_label                       # "Safety", "Welding", "Theory"
  t.string :category_color                       # "#ef4444"
  
  # Дополнительные поля
  t.integer :estimated_hours                     # Примерное время изучения
  t.string :difficulty_level                     # beginner, intermediate, advanced
  
  # Ссылки на материалы (простой JSON для MVP)
  t.json :resources, default: []                 # [{ title: "Видео", url: "..." }]
  
  t.timestamps
end

add_index :skills, [:roadmap_id, :key], unique: true
add_index :skills, :permit_template_id
add_index :skills, :skill_type
```

**Пример навыка:**
```ruby
Skill.create!(
  roadmap_id: 1,
  key: "mig-welding",
  title: "MIG Сварка",
  skill_type: "skill",
  category_label: "Welding",
  estimated_hours: 40,
  resources: [
    { title: "Видео-курс", url: "https://youtube.com/...", type: "video" },
    { title: "ГОСТ 14771-76", url: "https://docs.com/...", type: "document" }
  ]
)
```

**Пример допуска:**
```ruby
Skill.create!(
  roadmap_id: 1,
  permit_template_id: 1,  # Ссылка на PermitTemplate
  key: "electrical-safety-2",
  title: "Электробезопасность II группа",
  skill_type: "permit",
  category_label: "Safety"
)
```

### 8. `skill_dependencies` (Связи/Ребра Графа)

```ruby
create_table :skill_dependencies do |t|
  t.references :from_skill, null: false, foreign_key: { to_table: :skills }
  t.references :to_skill, null: false, foreign_key: { to_table: :skills }
  
  t.string :kind, default: 'required'            # required, optional
  
  t.timestamps
end

add_index :skill_dependencies, [:from_skill_id, :to_skill_id], unique: true, name: 'index_skill_deps_on_from_and_to'
```

**Пример:**
```ruby
# "Чтобы изучить MIG Сварку, нужно сначала пройти Технику безопасности"
SkillDependency.create!(
  from_skill_id: skill_safety.id,
  to_skill_id: skill_mig.id,
  kind: 'required'
)
```

---

## 📈 Прогресс Сотрудников

### 9. `user_progresses` (Трекинг Навыков и Допусков)

**Универсальная таблица** для навыков И допусков.

```ruby
create_table :user_progresses do |t|
  t.references :user, null: false, foreign_key: true
  t.references :skill, null: false, foreign_key: true
  
  # Статус прохождения
  t.string :status, default: 'todo'              # todo, in_progress, completed, expired
  
  # Данные Допуска (заполняется только для skill_type = 'permit')
  t.string :certificate_number                   # "АБВ-123456"
  t.date :issued_at                              # Дата выдачи
  t.date :expires_at                             # Рассчитывается автоматически
  t.string :issuing_authority                    # "Ростехнадзор МРО №1"
  
  # Подтверждение (для будущего, в MVP не используется)
  t.datetime :verified_at
  t.references :verified_by, foreign_key: { to_table: :users }
  
  # Комментарии
  t.text :notes
  
  # Метаданные
  t.datetime :started_at                         # Когда начал изучение
  t.datetime :completed_at                       # Когда завершил
  
  t.timestamps
end

add_index :user_progresses, [:user_id, :skill_id], unique: true
add_index :user_progresses, :status
add_index :user_progresses, :expires_at  # Для поиска просроченных
```

**Lifecycle:**

#### Навык:
```ruby
# Начал изучение
progress.update!(status: 'in_progress', started_at: Time.current)

# Изучил
progress.update!(status: 'completed', completed_at: Time.current)
```

#### Допуск:
```ruby
# Ввел данные допуска
progress.update!(
  status: 'completed',
  certificate_number: "АБВ-123456",
  issued_at: Date.parse("2024-01-15"),
  expires_at: Date.parse("2025-01-15"),  # issued_at + permit_template.expiration_months
  issuing_authority: "Ростехнадзор"
)

# Проверка истечения (background job)
if progress.expires_at < 30.days.from_now
  progress.update!(status: 'expiring_soon')
end

if progress.expires_at < Date.today
  progress.update!(status: 'expired')
end
```

---

## ⚙️ Инфраструктурные Таблицы (Solid Stack)

### 10. `solid_queue_jobs` (Background Jobs)

Создается автоматически gem'ом `solid_queue`.

```ruby
create_table :solid_queue_jobs do |t|
  t.string :queue_name, null: false
  t.string :class_name, null: false
  t.text :arguments
  t.integer :priority, default: 0
  t.string :active_job_id
  t.datetime :scheduled_at
  t.datetime :finished_at
  t.string :concurrency_key
  
  t.timestamps
end
```

**Примеры jobs:**
- `ExpiringPermitsNotifierJob` — отправка email за 30 дней до истечения
- `DailyStatsAggregatorJob` — агрегация статистики для дашбордов

### 11. `solid_cache_entries` (Cache Storage)

```ruby
create_table :solid_cache_entries do |t|
  t.binary :key, null: false, index: { unique: true }
  t.binary :value, null: false
  
  t.datetime :created_at, null: false
end
```

**Что кэшируем:**
- Структуру roadmap (nodes + edges)
- Список публичных roadmaps
- Статистику организации

---

## 🔍 Индексы и Оптимизация

### Критичные Индексы

```ruby
# 1. Multi-tenancy (быстрый поиск по организации)
add_index :users, [:organization_id, :email]
add_index :roadmaps, [:organization_id, :slug], unique: true
add_index :job_titles, [:organization_id, :title], unique: true  # 👔 NEW

# 2. Связи с должностями
add_index :users, [:organization_id, :job_title_id]  # 👔 NEW
add_index :job_titles, :roadmap_id  # 👔 NEW

# 3. Поиск истекающих допусков (для дашборда)
add_index :user_progresses, [:status, :expires_at]

# 4. Выборка навыков для roadmap (основной запрос)
add_index :skills, :roadmap_id

# 5. Поиск зависимостей (построение графа)
add_index :skill_dependencies, [:from_skill_id, :to_skill_id], unique: true

# 6. Поиск прогресса пользователя
add_index :user_progresses, [:user_id, :skill_id], unique: true
```

### Counter Caches

```ruby
# В roadmaps
add_column :roadmaps, :skills_count, :integer, default: 0

# В organizations
add_column :organizations, :users_count, :integer, default: 0

# Обновляются автоматически через belongs_to counter_cache: true
```

---

## 📝 Примеры Миграций

### Создание таблицы organizations

```ruby
# db/migrate/20260208000001_create_organizations.rb
class CreateOrganizations < ActiveRecord::Migration[8.0]
  def change
    create_table :organizations do |t|
      t.string :name, null: false
      t.string :slug, null: false
      t.string :plan_type, default: 'trial'
      t.string :subscription_status, default: 'active'
      t.datetime :trial_ends_at
      t.string :stripe_customer_id
      t.integer :employee_limit, default: 10
      t.string :deployment_type, default: 'cloud'
      t.string :license_key
      t.string :timezone, default: 'Europe/Moscow'
      t.string :locale, default: 'ru'
      
      t.timestamps
    end
    
    add_index :organizations, :slug, unique: true
    add_index :organizations, :license_key, unique: true
  end
end
```

### Создание таблицы job_titles

```ruby
# db/migrate/20260208000003_create_job_titles.rb
class CreateJobTitles < ActiveRecord::Migration[8.0]
  def change
    create_table :job_titles do |t|
      t.references :organization, null: false, foreign_key: true
      t.string :title, null: false
      t.text :description
      t.references :roadmap, foreign_key: true, null: true
      t.jsonb :required_permit_template_ids, default: []
      
      t.timestamps
    end
    
    add_index :job_titles, [:organization_id, :title], unique: true
    add_index :job_titles, :roadmap_id
  end
end
```

### Создание таблицы skills

```ruby
# db/migrate/20260208000005_create_skills.rb
class CreateSkills < ActiveRecord::Migration[8.0]
  def change
    create_table :skills do |t|
      t.references :roadmap, null: false, foreign_key: true
      t.references :permit_template, null: true, foreign_key: true
      
      t.string :key, null: false
      t.string :title, null: false
      t.text :description
      
      t.float :position_x, default: 0
      t.float :position_y, default: 0
      t.boolean :position_locked, default: false
      
      t.string :skill_type, default: 'skill'
      t.string :category_label
      t.string :category_color
      
      t.integer :estimated_hours
      t.string :difficulty_level
      t.json :resources, default: []
      
      t.timestamps
    end
    
    add_index :skills, [:roadmap_id, :key], unique: true
    add_index :skills, :permit_template_id
    add_index :skills, :skill_type
  end
end
```

---

## 🌱 Seeds (Начальные Данные)

### Типовые Допуски СНГ

```ruby
# db/seeds/permit_templates.rb

# Электробезопасность
PermitTemplate.create!([
  { title: "Электробезопасность I группа", code: "ELEC_I", country_code: "RU", expiration_months: 12, category: "electrical" },
  { title: "Электробезопасность II группа", code: "ELEC_II", country_code: "RU", expiration_months: 12, category: "electrical" },
  { title: "Электробезопасность III группа", code: "ELEC_III", country_code: "RU", expiration_months: 12, category: "electrical" },
  { title: "Электробезопасность IV группа", code: "ELEC_IV", country_code: "RU", expiration_months: 12, category: "electrical" },
  { title: "Электробезопасность V группа", code: "ELEC_V", country_code: "RU", expiration_months: 12, category: "electrical" }
])

# Сварочные работы
PermitTemplate.create!([
  { title: "НАКС (Неразрушающий контроль)", code: "NAKS_NDT", country_code: "RU", expiration_months: 24, category: "welding" },
  { title: "Газосварщик", code: "GAS_WELDER", country_code: "RU", expiration_months: 12, category: "welding" }
])

# Промышленная безопасность
PermitTemplate.create!([
  { title: "Работы на высоте (1 группа)", code: "HEIGHT_1", country_code: "RU", expiration_months: 36, category: "safety" },
  { title: "Работы на высоте (2 группа)", code: "HEIGHT_2", country_code: "RU", expiration_months: 36, category: "safety" },
  { title: "Стропальщик", code: "RIGGING", country_code: "RU", expiration_months: 12, category: "safety" }
])
```

---

## 📜 Audit Trail (paper_trail)

### Таблица `versions` (Версионирование Изменений)

**Назначение:** Tracking всех изменений критичных данных для compliance.

```ruby
create_table :versions do |t|
  t.string   :item_type, null: false         # "UserProgress", "Skill", etc
  t.bigint   :item_id,   null: false
  t.string   :event,     null: false         # create, update, destroy
  t.string   :whodunnit                      # user_id кто изменил
  t.jsonb    :object                         # Состояние ДО изменения
  t.jsonb    :object_changes                 # Что изменилось
  t.jsonb    :metadata                       # Дополнительные данные (organization_id, ip_address)
  t.datetime :created_at
end

add_index :versions, [:item_type, :item_id]
add_index :versions, :whodunnit
add_index :versions, :created_at
add_index :versions, :metadata, using: :gin  # PostgreSQL GIN index для JSONB
```

**Какие модели отслеживаются:**
- ✅ `UserProgress` — критично! (кто поставил допуск, когда)
- ✅ `Skill` — изменения в roadmaps
- ✅ `User` — изменения ролей, email
- ❌ `Session` — не нужно (слишком много записей)

**Использование в моделях:**
```ruby
class UserProgress < ApplicationRecord
  has_paper_trail on: [:create, :update, :destroy],
                  ignore: [:updated_at],
                  meta: { 
                    organization_id: :organization_id,
                    ip_address: :current_ip 
                  }
end
```

**Retention policy:**
- Хранить минимум **2 года** (российское законодательство по охране труда)
- Партиционирование по годам (PostgreSQL table partitioning)
- Архивация старых версий в S3 Glacier

**Пример запроса истории:**
```ruby
# В контроллере
@history = @user_progress.versions.includes(:whodunnit).order(created_at: :desc)

# В UI
@history.each do |v|
  user = User.find(v.whodunnit)
  puts "#{v.created_at}: #{user.full_name} изменил #{v.changeset}"
end
```

---

## 🔄 PostgreSQL Преимущества

**Почему PostgreSQL с Day 1:**

1. **JSONB** — быстрые запросы по `metadata`, `resources`, `required_permit_template_ids`
2. **GIN indexes** — полнотекстовый поиск по русским навыкам
3. **Table partitioning** — масштабирование `versions` таблицы (audit trail)
4. **pgaudit** — готовность к enterprise compliance (Ростехнадзор)
5. **Concurrent writes** — 100+ сотрудников одновременно отмечают прогресс
6. **Row-level security** — дополнительный уровень защиты multi-tenancy
7. **Репликация** — встроенная master-slave для on-premise HA

**Setup в проекте:**
```ruby
# Gemfile
gem 'pg', '~> 1.5'
gem 'paper_trail'  # Audit trail

# config/database.yml
production:
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  database: industrialprofi_production
  username: industrialprofi
  password: <%= ENV['DATABASE_PASSWORD'] %>
```

---

**Следующий документ:** `04_BACKEND.md` (Rails Controllers, Services, Models)
