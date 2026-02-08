# 📅 План Разработки MVP (5 Недель)

> **Срок:** 5 недель (25 рабочих дней)  
> **Режим:** Full-time (8 часов/день)  
> **Разработчик:** 1 человек (ты + AI-ассистент)

---

## 🎯 Общая Стратегия

### Правила Работы:
1. **Vertical Slicing:** Делаем функцию от БД до UI полностью, а не слоями
2. **Ship Early:** Каждая неделя заканчивается рабочей фичей (можно показать пользователю)
3. **No Perfectionism:** "Done is better than perfect" — доделаем после первых продаж
4. **Testing First:** Критичные фичи тестируем сразу, не в конце

### Daily Workflow:
```
09:00 - 09:15  Планирование дня (чек TODO list)
09:15 - 12:00  Глубокая работа (фокус, без отвлечений)
12:00 - 13:00  Обед
13:00 - 16:00  Глубокая работа
16:00 - 17:00  Тестирование + деплой на staging
17:00 - 17:30  Обновление TODO, коммит прогресса
```

---

## 📊 Roadmap Overview

```
Week 1: Foundation              ████████░░░░░░░░░░░░░░░░  (30%)
Week 2: Roadmaps Viewer         ███████████████░░░░░░░░░  (60%)
Week 3: Roadmap Editor          ████████████████████████  (90%)
Week 4: Progress Tracking       ████████████████████████  (95%)
Week 5: Polish & Deploy         ████████████████████████  (100%)
```

---

## 🗓️ НЕДЕЛЯ 1: Foundation (Дни 1-5)

### Цель Недели:
✅ Рабочая авторизация  
✅ Multi-tenancy настроен  
✅ Базовый UI с Layout  
✅ Seeds для публичных roadmaps

---

### День 1: Database & Models (PostgreSQL + paper_trail)

**⭐ Изменения:** Мигрировали на PostgreSQL с Day 1 для enterprise-ready архитектуры

**Задачи:**
- [ ] Запустить PostgreSQL (Docker Compose)
- [ ] Установить gems: `pg`, `acts_as_tenant`, `bcrypt`, `paper_trail`, `blueprinter`, `pagy`
- [ ] Миграции: `organizations`, `users`, `sessions`
- [ ] **НОВОЕ:** Установить paper_trail (audit trail)
- [ ] **НОВОЕ:** Добавить `external_1c_id` в Users (для интеграции 1С)
- [ ] Models с валидациями
- [ ] Seeds для тестовой организации

**Команды:**
```bash
# 1. Запустить PostgreSQL
docker-compose up -d

# 2. Установить gems
bundle install
# (Gemfile уже обновлен: pg, paper_trail, acts_as_tenant, bcrypt, blueprinter, pagy)

# 3. Создать БД
bin/rails db:create

# 4. Миграции основных таблиц
bin/rails g model Organization name:string slug:string:uniq plan_type:string subscription_status:string

# ⭐ ВАЖНО: User с полем для 1С интеграции!
bin/rails g model User organization:references email:string:uniq password_digest:string full_name:string role:string department:string external_1c_id:string synced_from_1c_at:datetime metadata:jsonb

bin/rails g model Session user:references ip_address:string user_agent:string

# 5. ⭐ НОВОЕ: Установить paper_trail (audit trail)
bin/rails generate paper_trail:install --with-changes

# 6. Запустить миграции
bin/rails db:migrate
```

**Проверка:**
```ruby
# В rails console
org = Organization.create!(name: "Test Corp", slug: "test", plan_type: "trial")
user = org.users.create!(
  email: "test@test.com", 
  password: "password123", 
  role: "owner",
  metadata: { personnel_number: "00001" }  # ⭐ JSONB работает!
)
user.authenticate("password123")  # Должно вернуть user

# ⭐ НОВОЕ: Проверка paper_trail
user.update(full_name: "Петр Петров")
user.versions.last.changeset
# => {"full_name"=>[nil, "Петр Петров"]}
```

**Тесты (написать в конце дня):**
- `test/models/organization_test.rb` (валидация slug uniqueness)
- `test/models/user_test.rb` (has_secure_password работает)
- **НОВОЕ:** `test/models/user_test.rb` (paper_trail tracking работает)

**Время:** ~7-8 часов (+1 час на PostgreSQL setup)

---

### День 2: Authentication (Rails 8 Generator)

**Задачи:**
- [ ] `bin/rails generate authentication`
- [ ] Адаптировать под multi-tenancy (signup создает Organization)
- [ ] UI: Login/Register формы (Inertia)
- [ ] Tests: integration tests для sign up/login

**Файлы для создания:**
```
app/controllers/sessions_controller.rb
app/controllers/registrations_controller.rb
app/frontend/pages/Auth/Login.tsx
app/frontend/pages/Auth/Register.tsx
```

**Register Form (TSX):**
```tsx
<form onSubmit={handleSubmit}>
  <Input label="Название организации" name="organization[name]" />
  <Input label="Email" type="email" name="user[email]" />
  <Input label="Пароль" type="password" name="user[password]" />
  <Button type="submit">Зарегистрироваться</Button>
</form>
```

**Проверка:**
- Можно зарегистрироваться через форму
- После регистрации redirect на `/dashboard`
- Создается Organization + User (role: owner)

**Время:** ~7 часов

---

### День 3: UI Foundation (Layout, Tailwind, Components)

**Задачи:**
- [ ] Setup TypeScript config (`tsconfig.json`)
- [ ] Установить shadcn/ui компоненты (Button, Card, Input)
- [ ] Layout компонент (Navbar + Sidebar)
- [ ] Пустой Dashboard

**npm install:**
```bash
npm install @headlessui/react lucide-react clsx tailwind-merge date-fns
npm install -D @types/react @types/react-dom typescript
```

**Компоненты для создания:**
```
app/frontend/components/ui/Button.tsx
app/frontend/components/ui/Card.tsx
app/frontend/components/ui/Input.tsx
app/frontend/components/layout/Layout.tsx
app/frontend/components/layout/Navbar.tsx
app/frontend/components/layout/Sidebar.tsx
app/frontend/pages/Dashboard/Index.tsx
```

**Проверка:**
- После логина показывается Dashboard с Navbar/Sidebar
- Все UI компоненты рендерятся корректно
- TypeScript компилируется без ошибок (`npm run type-check`)

**Время:** ~6 часов

---

### День 4: Roadmaps & Skills Models + Seeds

**Задачи:**
- [ ] Миграции: `roadmaps`, `skills`, `skill_dependencies`, `permit_templates`, `job_titles` 👔
- [ ] Models с associations
- [ ] Seeds: Permit Templates (10 типовых допусков СНГ)
- [ ] Seeds: 1 публичный roadmap "Сварщик" (через YAML)

**Миграции:**
```bash
bin/rails g model Roadmap organization:references forked_from:references title:string slug:string visibility:string is_template:boolean
bin/rails g model Skill roadmap:references permit_template:references key:string title:string skill_type:string position_x:float position_y:float
bin/rails g model SkillDependency from_skill:references to_skill:references kind:string
bin/rails g model PermitTemplate title:string code:string country_code:string expiration_months:integer category:string

# 👔 НОВОЕ: Должности (Гибридная модель)
bin/rails g model JobTitle organization:references roadmap:references title:string description:text required_permit_template_ids:jsonb
```

**YAML Roadmap:**
```yaml
# db/seeds/roadmaps/welder.yml
title: "Сварщик"
slug: "welder"
visibility: "public"
is_template: true
nodes:
  - key: "safety_basics"
    title: "Техника безопасности"
    skill_type: "skill"
    position: [100, 50]
  - key: "electrical_safety_2"
    title: "Электробезопасность II группа"
    skill_type: "permit"
    permit_template_code: "ELEC_II"
    position: [100, 150]
edges:
  - from: "safety_basics"
    to: "electrical_safety_2"
```

**Service для импорта:**
```ruby
# app/services/roadmap_import_service.rb
class RoadmapImportService
  def import_from_yaml(file_path)
    data = YAML.load_file(file_path)
    # ... логика создания roadmap, skills, dependencies
  end
end
```

**Проверка:**
```ruby
# rails console
Roadmap.public_templates.count  # => 1
PermitTemplate.where(country_code: 'RU').count  # => 10

# 👔 НОВОЕ: Проверка JobTitle
job_title = JobTitle.create!(
  organization_id: Organization.first.id,
  title: "Сварщик 5 разряда",
  required_permit_template_ids: [1, 5, 7],  # Обязательные допуски
  roadmap_id: Roadmap.find_by(slug: 'welder')&.id  # Карта профессии (опционально)
)
job_title.required_permit_template_ids  # => [1, 5, 7]
```

**Время:** ~8 часов

---

### День 5: Multi-tenancy Setup + Tests

**Задачи:**
- [ ] Настроить `acts_as_tenant` в ApplicationController
- [ ] Добавить tenant scopes в модели
- [ ] **КРИТИЧНО:** Tests для multi-tenancy isolation
- [ ] Staging deploy (Kamal setup)

**ApplicationController:**
```ruby
class ApplicationController < ActionController::Base
  include Inertia::Controller
  
  set_current_tenant_through_filter
  before_action :set_tenant
  
  private
  
  def set_tenant
    set_current_tenant(current_user.organization) if user_signed_in?
  end
end
```

**Tests:**
```ruby
# test/integration/multi_tenancy_test.rb
test "user cannot access another org's data" do
  org1 = create(:organization)
  org2 = create(:organization)
  
  user1 = create(:user, organization: org1)
  roadmap2 = create(:roadmap, organization: org2)
  
  sign_in user1
  get organizations_roadmap_path(roadmap2)
  
  assert_response :not_found
end
```

**Kamal Setup:**
```bash
# Первый раз настраиваем сервер
kamal setup

# Деплоим
kamal deploy
```

**Проверка:**
- Все тесты зеленые (`bin/rails test`)
- Staging доступен по https://YOUR_SERVER_IP
- Multi-tenancy работает (тесты проходят)

**Время:** ~7 часов

---

### ✅ Итог Недели 1:
- Рабочая авторизация ✓
- Multi-tenancy ✓
- Базовый UI ✓
- Публичные roadmaps в БД ✓
- Staging сервер запущен ✓

**Demo:** Можно зарегистрироваться, залогиниться, увидеть пустой дашборд

---

## 🗓️ НЕДЕЛЯ 2: Roadmaps Viewer (Дни 6-10)

### Цель Недели:
✅ React Flow интеграция  
✅ Просмотр публичных roadmaps  
✅ Skill Sidebar (детали узла)  
✅ Кэширование структуры графа

---

### День 6: React Flow Setup + Basic Viewer

**Задачи:**
- [ ] `npm install reactflow dagre @types/dagre`
- [ ] RoadmapViewer компонент (read-only)
- [ ] SkillNode компонент (custom node)
- [ ] RoadmapsController#show (отдает JSON для графа)

**Компоненты:**
```tsx
// app/frontend/components/roadmap/RoadmapViewer.tsx
// app/frontend/components/roadmap/SkillNode.tsx
// app/frontend/pages/Roadmaps/Show.tsx
```

**Controller:**
```ruby
# app/controllers/roadmaps_controller.rb
def show
  roadmap = Roadmap.find_by!(slug: params[:id])
  
  render inertia: 'Roadmaps/Show', props: {
    roadmap: {
      nodes: roadmap.skills.map { |s| { id: s.id.to_s, position: {x: s.position_x, y: s.position_y}, data: { label: s.title } } },
      edges: roadmap.skill_dependencies.map { |d| { source: d.from_skill_id.to_s, target: d.to_skill_id.to_s } }
    }
  }
end
```

**Проверка:**
- `/roadmaps/welder` показывает граф
- Узлы рендерятся
- Связи отображаются

**Время:** ~8 часов

---

### День 7: Skill Sidebar + Click Interactions

**Задачи:**
- [ ] SkillSidebar компонент (Slide-over panel)
- [ ] Клик по узлу → открывается sidebar
- [ ] Отображение: title, description, resources
- [ ] React State management (selectedSkillId)

**Логика:**
```tsx
const [selectedSkillId, setSelectedSkillId] = useState<string | null>(null);

<ReactFlow onNodeClick={(e, node) => setSelectedSkillId(node.id)} />

{selectedSkillId && (
  <SkillSidebar 
    skillId={selectedSkillId}
    onClose={() => setSelectedSkillId(null)}
  />
)}
```

**Проверка:**
- Клик по узлу открывает sidebar
- ESC или клик на X закрывает sidebar
- Кнопка "Назад" в браузере не ломает UI

**Время:** ~6 часов

---

### День 8: User Progress Integration (Read-Only)

**Задачи:**
- [ ] Миграция: `user_progresses`
- [ ] Model UserProgress
- [ ] Раскраска узлов по статусу (серый/желтый/зеленый)
- [ ] Seeds: прогресс для тестового пользователя

**Раскраска:**
```tsx
const statusColors = {
  todo: 'bg-gray-100 border-gray-300',
  in_progress: 'bg-yellow-100 border-yellow-400',
  completed: 'bg-green-100 border-green-400',
};

<div className={statusColors[data.status]}>
  {data.label}
</div>
```

**Controller update:**
```ruby
def show
  # ... roadmap structure ...
  
  user_progress = current_user.user_progresses
    .where(skill_id: roadmap.skill_ids)
    .index_by(&:skill_id)
  
  render inertia: 'Roadmaps/Show', props: {
    roadmap: graph_structure,
    userProgress: user_progress
  }
end
```

**Проверка:**
- Узлы раскрашены по статусу
- Если не залогинен — все серые
- Если залогинен — видны статусы

**Время:** ~7 часов

---

### День 9: Catalog + Caching

**Задачи:**
- [ ] Страница каталога `/roadmaps` (список публичных roadmaps)
- [ ] Карточки roadmaps с превью
- [ ] Кэширование структуры графа (Solid Cache)
- [ ] Performance test (загрузка roadmap < 500ms)

**Catalog Page:**
```tsx
// app/frontend/pages/Roadmaps/Index.tsx
<div className="grid grid-cols-3 gap-4">
  {roadmaps.map(roadmap => (
    <Card key={roadmap.id}>
      <h3>{roadmap.title}</h3>
      <p>{roadmap.skills_count} навыков</p>
      <Button href={`/roadmaps/${roadmap.slug}`}>
        Открыть
      </Button>
    </Card>
  ))}
</div>
```

**Caching:**
```ruby
Rails.cache.fetch("roadmap/#{roadmap.id}/structure/v#{roadmap.updated_at.to_i}") do
  # ... генерация nodes/edges ...
end
```

**Проверка:**
- Каталог показывает roadmaps
- Первая загрузка графа медленная, вторая быстрая (кэш)
- `bin/rails dev:cache` включен

**Время:** ~6 часов

---

### День 10: Tests + Refactoring

**Задачи:**
- [ ] System test: просмотр roadmap
- [ ] Integration test: кэширование работает
- [ ] Рефакторинг: вынести serializers
- [ ] Code review + cleanup

**System Test:**
```ruby
test "user can view public roadmap and click node" do
  roadmap = create(:roadmap, :public, title: "Сварщик")
  skill = create(:skill, roadmap: roadmap, title: "Техника безопасности")
  
  visit roadmap_path(roadmap)
  
  assert_text "Сварщик"
  assert_selector ".react-flow"
  
  find(".react-flow__node", text: "Техника безопасности").click
  
  within(".skill-sidebar") do
    assert_text "Техника безопасности"
  end
end
```

**Serializers:**
```ruby
# app/serializers/roadmap_serializer.rb
class RoadmapSerializer < Blueprinter::Base
  identifier :id
  fields :title, :slug, :description
  
  view :with_graph do
    association :skills, blueprint: SkillSerializer
    association :skill_dependencies, blueprint: DependencySerializer
  end
end
```

**Проверка:**
- Все тесты зеленые
- Code coverage > 60%
- Нет дублирования кода

**Время:** ~6 часов

---

### ✅ Итог Недели 2:
- React Flow работает ✓
- Можно просматривать графы ✓
- Sidebar показывает детали ✓
- Кэширование ускоряет загрузку ✓

**Demo:** Можно открыть roadmap, кликнуть по узлу, увидеть детали

---

## 🗓️ НЕДЕЛЯ 3: Roadmap Editor (Дни 11-15)

### Цель Недели:
✅ Fork roadmap  
✅ CRUD для skills через формы  
✅ Создание edges (drag connection)  
✅ Auto-layout

---

### День 11: Fork Roadmap Service

**Задачи:**
- [ ] RoadmapForkService
- [ ] Organizations::RoadmapsController (namespace)
- [ ] UI: кнопка "Использовать шаблон" на карточке
- [ ] Tests: fork копирует skills + dependencies

**Service:**
```ruby
class RoadmapForkService
  def call
    ActiveRecord::Base.transaction do
      # 1. Copy roadmap
      # 2. Copy skills (+ ID mapping)
      # 3. Copy dependencies (new IDs)
    end
  end
end
```

**Test:**
```ruby
test "fork creates independent copy" do
  source = create(:roadmap, :with_skills, skills_count: 3)
  org = create(:organization)
  
  forked = RoadmapForkService.new(source, org).call
  
  assert_equal 3, forked.skills.count
  assert_not_equal source.skills.first.id, forked.skills.first.id
end
```

**Проверка:**
- Клик "Использовать шаблон" → создается копия
- Redirect на `/organizations/roadmaps/:id/edit`
- Skills скопированы с новыми ID

**Время:** ~8 часов

---

### День 12-13: Roadmap Editor UI (2 дня)

**Задачи:**
- [ ] Layout редактора (3 колонки: Sidebar | Canvas | Form)
- [ ] Skill CRUD через боковую панель
- [ ] Organizations::SkillsController (API endpoints)
- [ ] React Forms (добавление/редактирование skill)

**Layout:**
```tsx
<div className="flex h-screen">
  {/* Left: Skill List */}
  <aside className="w-64 border-r">
    <button onClick={() => setShowAddForm(true)}>+ Навык</button>
    <ul>
      {skills.map(skill => (
        <li onClick={() => setEditingSkill(skill)}>
          {skill.title}
        </li>
      ))}
    </ul>
  </aside>
  
  {/* Center: React Flow Canvas */}
  <main className="flex-1">
    <ReactFlow nodes={nodes} edges={edges} />
  </main>
  
  {/* Right: Edit Form */}
  {editingSkill && (
    <aside className="w-96 border-l">
      <SkillForm skill={editingSkill} />
    </aside>
  )}
</div>
```

**API Endpoints:**
```ruby
# app/controllers/organizations/skills_controller.rb
namespace :organizations do
  resources :roadmaps do
    resources :skills, only: [:create, :update, :destroy]
  end
end
```

**Проверка:**
- Можно добавить skill через форму
- Skill появляется на графе
- Можно редактировать/удалять skill

**Время:** ~14 часов (2 дня)

---

### День 14: Edge Creation + Auto-Layout

**Задачи:**
- [ ] React Flow: onConnect handler (создание edges)
- [ ] Organizations::DependenciesController
- [ ] AutoLayoutService (Dagre integration)
- [ ] Кнопка "Auto-arrange" в редакторе

**onConnect:**
```tsx
const onConnect = useCallback((params) => {
  router.post(`/organizations/roadmaps/${roadmap.id}/dependencies`, {
    from_skill_id: params.source,
    to_skill_id: params.target,
  });
}, []);

<ReactFlow onConnect={onConnect} />
```

**AutoLayoutService:**
```ruby
class AutoLayoutService
  def calculate
    # Dagre layout algorithm
    # Returns positions hash: { skill_key: { x:, y: } }
  end
end
```

**Проверка:**
- Drag от узла к узлу создает связь
- Кнопка "Auto-arrange" пересчитывает координаты
- Координаты сохраняются в БД

**Время:** ~7 часов

---

### День 15: Tests + Permissions

**Задачи:**
- [ ] Permissions: только Manager/Owner может редактировать
- [ ] Integration tests: Employee НЕ может редактировать
- [ ] System test: полный flow (fork → add skill → save)

**Permissions:**
```ruby
before_action :require_manager!, except: [:index, :show]

def require_manager!
  unless current_user.manager? || current_user.owner?
    redirect_to root_path, alert: "Доступ запрещен"
  end
end
```

**System Test:**
```ruby
test "manager can fork and edit roadmap" do
  manager = create(:user, role: 'manager')
  public_roadmap = create(:roadmap, :public)
  
  sign_in manager
  visit roadmaps_path
  
  click_button "Использовать шаблон"
  
  # В редакторе
  click_button "+ Навык"
  fill_in "Название", with: "Новый навык"
  click_button "Сохранить"
  
  assert_text "Новый навык"
end
```

**Проверка:**
- Employee НЕ видит кнопку "Редактировать"
- Попытка открыть URL напрямую → 403/redirect
- Все тесты зеленые

**Время:** ~7 часов

---

### ✅ Итог Недели 3:
- Fork roadmap работает ✓
- Можно добавлять/редактировать skills ✓
- Создание edges работает ✓
- Auto-layout функционирует ✓
- Permissions настроены ✓

**Demo:** Manager может скопировать roadmap и кастомизировать под свою компанию

---

## 🗓️ НЕДЕЛЯ 4: Progress Tracking (Дни 16-20)

### Цель Недели:
✅ Employee отмечает прогресс  
✅ Форма ввода допусков  
✅ Dashboard с виджетами  
✅ Email уведомления

---

### День 16: Progress Controller + Skill Completion

**Задачи:**
- [ ] Organizations::ProgressController
- [ ] Кнопки в Sidebar: "Начать изучение" / "Изучил"
- [ ] Обновление узла на графе (цвет меняется)

**Controller:**
```ruby
def update
  progress = current_user.user_progresses.find_or_initialize_by(skill_id: params[:skill_id])
  progress.update!(status: params[:status])
  
  render json: progress
end
```

**UI:**
```tsx
<Button onClick={() => {
  router.post(`/progress/${skill.id}`, { status: 'completed' }, {
    preserveScroll: true,
    onSuccess: () => {
      // Узел становится зеленым
    }
  });
}}>
  Отметить изученным
</Button>
```

**Проверка:**
- Клик "Изучил" → узел зеленеет
- БД обновляется
- Reload страницы → статус сохранен

**Время:** ~6 часов

---

### День 17: Permit Form + Expiration Calculation

**Задачи:**
- [ ] PermitForm компонент
- [ ] Логика расчета expires_at на backend
- [ ] Валидация дат
- [ ] Сохранение данных допуска

**PermitForm:**
```tsx
<form onSubmit={handleSubmit}>
  <Input label="Номер удостоверения" name="certificate_number" required />
  <Input type="date" label="Дата выдачи" name="issued_at" required />
  <Input label="Кем выдано" name="issuing_authority" />
  
  <div className="text-sm text-gray-500">
    Срок действия: {skill.permitTemplate.expiration_months} месяцев
  </div>
  
  <Button type="submit">Сохранить</Button>
</form>
```

**Backend:**
```ruby
def update
  # ...
  if skill.permit?
    progress.assign_attributes(permit_params)
    progress.expires_at = progress.issued_at + skill.permit_template.expiration_months.months
  end
  # ...
end
```

**Проверка:**
- Форма допуска показывается для permit узлов
- expires_at рассчитывается корректно
- Валидация работает (issued_at < expires_at)

**Время:** ~7 часов

---

### День 18: Employee Dashboard

**Задачи:**
- [ ] Виджет "Мои roadmaps" (прогресс)
- [ ] Виджет "Истекающие допуски"
- [ ] Виджет "Просроченные допуски"
- [ ] Queries для дашборда (оптимизация N+1)

**Dashboard:**
```tsx
<div className="grid grid-cols-2 gap-4">
  {/* Мои roadmaps */}
  <Card>
    <h3>Мои roadmaps</h3>
    {myRoadmaps.map(roadmap => (
      <div key={roadmap.id}>
        <h4>{roadmap.title}</h4>
        <ProgressBar value={roadmap.progress_percent} />
        <p>{roadmap.completed_count} из {roadmap.total_count}</p>
      </div>
    ))}
  </Card>
  
  {/* Истекающие допуски */}
  <Card className="border-orange-400">
    <h3>⚠️ Истекают через 30 дней</h3>
    {expiringPermits.map(permit => (
      <div>{permit.skill.title} - {permit.expires_at}</div>
    ))}
  </Card>
</div>
```

**Controller:**
```ruby
def index
  @expiring_permits = current_user.user_progresses
    .expiring_soon
    .includes(skill: :permit_template)
  
  # ...
end
```

**Проверка:**
- Dashboard показывает актуальные данные
- Прогресс рассчитывается корректно
- Нет N+1 запросов (проверить через Bullet gem)

**Время:** ~7 часов

---

### День 19: Email Notifications (Background Job)

**Задачи:**
- [ ] ExpiringPermitsNotifierJob
- [ ] Mailer: UserMailer (permit_expiring_soon, permit_expiring_urgent)
- [ ] Email templates
- [ ] Тест job + маילера

**Job:**
```ruby
class ExpiringPermitsNotifierJob < ApplicationJob
  def perform
    # Находим permit истекающие через 30 дней
    UserProgress.expiring_soon.find_each do |progress|
      UserMailer.permit_expiring_soon(progress).deliver_later
    end
    
    # Помечаем истекшие
    UserProgress.where('expires_at < ?', Date.today).update_all(status: 'expired')
  end
end
```

**Mailer:**
```ruby
class UserMailer < ApplicationMailer
  def permit_expiring_soon(progress)
    @progress = progress
    @user = progress.user
    
    mail(
      to: @user.email,
      subject: "Допуск истекает через 30 дней"
    )
  end
end
```

**Template:**
```erb
<h1>Здравствуйте, <%= @user.full_name %>!</h1>

<p>
  Ваш допуск "<%= @progress.skill.title %>" истекает <%= @progress.expires_at.strftime('%d.%m.%Y') %>.
</p>

<p>Пожалуйста, продлите его своевременно.</p>
```

**Проверка:**
- Job запускается через `bin/rails runner "ExpiringPermitsNotifierJob.perform_now"`
- Email отправляется (Letter Opener в dev)
- Просроченные permits помечаются как expired

**Время:** ~7 часов

---

### День 20: Manager Dashboard + Матрица Навыков + Управление Должностями 👔

**Задачи:**
- [ ] Страница "Сотрудники отдела"
- [ ] Матрица навыков (таблица)
- [ ] Фильтр по отделу
- [ ] 👔 CRUD для должностей (Organizations::JobTitlesController)
- [ ] 👔 UI для выбора обязательных допусков и карты профессии
- [ ] Export в CSV (опционально)

**Матрица:**
```tsx
<table>
  <thead>
    <tr>
      <th>Сотрудник</th>
      {skills.map(skill => <th>{skill.title}</th>)}
    </tr>
  </thead>
  <tbody>
    {employees.map(employee => (
      <tr>
        <td>{employee.full_name}</td>
        {skills.map(skill => (
          <td>
            {getStatusIcon(employee.progress[skill.id])}
          </td>
        ))}
      </tr>
    ))}
  </tbody>
</table>
```

**Controller:**
```ruby
def index
  @employees = current_organization.users.where(department: params[:department])
  @roadmap = current_organization.roadmaps.find(params[:roadmap_id])
  
  # Eager load progress
  @progress_matrix = UserProgress
    .where(user_id: @employees.ids, skill_id: @roadmap.skill_ids)
    .group_by(&:user_id)
    .transform_values { |p| p.index_by(&:skill_id) }
end
```

**Проверка:**
- Таблица показывает матрицу навыков
- Фильтр по отделу работает
- Менеджер видит только свой отдел
- 👔 Можно создать должность с обязательными допусками
- 👔 Можно назначить карту профессии должности (опционально)

**Время:** ~8 часов

---

### 👔 НОВОЕ: Управление Должностями (Встроено в День 20)

**JobTitle Form (UI):**
```tsx
<form onSubmit={handleSubmit}>
  <Input label="Название должности" name="title" required />
  <Textarea label="Описание обязанностей" name="description" />
  
  {/* Обязательные допуски (критично для compliance) */}
  <fieldset>
    <legend>Обязательные допуски</legend>
    {permitTemplates.map(permit => (
      <Checkbox 
        key={permit.id}
        label={permit.title}
        checked={selectedPermits.includes(permit.id)}
        onChange={() => togglePermit(permit.id)}
      />
    ))}
  </fieldset>
  
  {/* Карта профессии (опционально, для развития) */}
  <Select 
    label="Карта профессии (опционально)"
    name="roadmap_id"
    options={roadmaps}
    placeholder="Не выбрано"
  />
  
  <Button type="submit">Сохранить</Button>
</form>
```

**Backend:**
```ruby
# app/controllers/organizations/job_titles_controller.rb
class Organizations::JobTitlesController < Organizations::BaseController
  before_action :require_manager!
  
  def create
    @job_title = current_organization.job_titles.build(job_title_params)
    
    if @job_title.save
      redirect_to organizations_job_titles_path, notice: "Должность создана"
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  private
  
  def job_title_params
    params.require(:job_title).permit(
      :title, 
      :description, 
      :roadmap_id,
      required_permit_template_ids: []
    )
  end
end
```

**Логика автоназначения roadmap:**
```ruby
# app/models/user.rb
class User < ApplicationRecord
  belongs_to :job_title, optional: true
  
  after_update :assign_roadmap_from_job_title, if: :job_title_id_changed?
  
  private
  
  def assign_roadmap_from_job_title
    return unless job_title&.roadmap_id
    
    # Автоматически назначаем roadmap из должности
    RoadmapAssignmentService.new(self, job_title.roadmap).call
  end
end
```

---

### ✅ Итог Недели 4:
- Сотрудники отмечают прогресс ✓
- Форма допусков работает ✓
- Dashboard показывает виджеты ✓
- Email уведомления настроены ✓
- Матрица навыков для менеджера ✓

**Demo:** Полный user flow от регистрации до заполнения допуска

---

## 🗓️ НЕДЕЛЯ 5: Polish & Deploy (Дни 21-25)

### Цель Недели:
✅ Bug fixes  
✅ Performance optimization  
✅ Production deploy  
✅ Documentation

---

### День 21: Bug Hunting + System Tests

**Задачи:**
- [ ] Полный manual testing checklist (см. `07_TESTING.md`)
- [ ] Найти и исправить критичные баги
- [ ] System tests для критичных flows
- [ ] Code review (рефакторинг)

**Фокус на:**
- Multi-tenancy leaks (КРИТИЧНО!)
- N+1 queries
- Permission bypasses
- Frontend ошибки в console

**Время:** ~8 часов

---

### День 22: Performance Optimization

**Задачи:**
- [ ] Добавить индексы в БД (если пропущены)
- [ ] Russian Doll Caching для дашбордов
- [ ] Eager loading везде где есть N+1
- [ ] Frontend: React.memo для тяжелых компонентов

**Метрики:**
- Roadmap show < 500ms (с кэшом < 100ms)
- Dashboard < 300ms
- No N+1 queries (Bullet gem)

**Время:** ~6 часов

---

### День 23: Production Deploy + pgBackRest Setup

**Задачи:**
- [ ] Kamal config (`config/deploy.yml`)
- [ ] PostgreSQL backups setup (pg_dump cron или pgBackRest)
- [ ] Environment variables (secrets)
- [ ] First production deploy
- [ ] Smoke tests на production

**Deploy:**
```bash
kamal setup  # Первый раз
kamal deploy
kamal app exec 'bin/rails db:seed'
```

**Проверка:**
- Production открывается по https://
- SSL сертификат работает
- Seeds загружены (публичные roadmaps есть)
- PostgreSQL backups настроены (cron job работает)

**Время:** ~8 часов

---

### День 24: Monitoring + Error Tracking

**Задачи:**
- [ ] Setup Sentry (error tracking)
- [ ] Healthcheck endpoint (`/up`)
- [ ] UptimeRobot (uptime monitoring)
- [ ] Production logs analysis

**Sentry:**
```bash
bundle add sentry-ruby sentry-rails

# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV['SENTRY_DSN']
  config.environment = Rails.env
end
```

**Проверка:**
- Sentry получает ошибки
- UptimeRobot пингует каждые 5 минут
- Healthcheck возвращает 200

**Время:** ~5 часов

---

### День 25: Documentation + Launch Prep

**Задачи:**
- [ ] README.md (для GitHub)
- [ ] User Guide (как пользоваться)
- [ ] Admin Guide (как добавить публичный roadmap)
- [ ] Prepare landing page (простая страница на `/`)

**README:**
```markdown
# IndustrialPROFI

B2B SaaS для управления квалификацией промышленного персонала.

## Возможности
- Интерактивные карты профессий
- Трекинг допусков и сертификатов
- Матрица навыков команды

## Tech Stack
- Rails 8 + PostgreSQL 16
- React 19 + Inertia.js
- React Flow
- Kamal 2
```

**Landing Page:**
```tsx
// app/frontend/pages/Landing/Index.tsx
<div className="hero">
  <h1>IndustrialPROFI</h1>
  <p>Управление квалификацией промышленного персонала</p>
  <Button href="/register">Попробовать бесплатно</Button>
</div>
```

**Проверка:**
- README актуален
- Документация понятна
- Landing page выглядит прилично

**Время:** ~6 часов

---

### ✅ Итог Недели 5:
- MVP полностью работает ✓
- Deploy в production ✓
- Мониторинг настроен ✓
- Документация готова ✓

**Demo:** Рабочий продукт на https://industrialprofi.ru

---

## 🎉 MVP Готов! Что дальше?

### Immediate Next Steps:
1. **Beta Testing:** Пригласить 2-3 компании для тестирования
2. **Feedback Loop:** Собрать фидбек, исправить баги
3. **First Sales:** Продать первые 3-5 подписок
4. **Iterate:** Улучшать на основе реального использования

### Post-MVP Features (v2):
- Drag-and-drop узлов в редакторе
- Подтверждение навыков менеджером
- Загрузка скан-копий сертификатов
- Export матрицы в PDF/Excel
- API для интеграций

### Scaling (v3):
- Английская локализация
- On-Premise deployment option
- Mobile app
- Advanced analytics

---

## 📊 Метрики Успеха

### Week 1:
- ✅ Можно зарегистрироваться и залогиниться

### Week 2:
- ✅ Можно просматривать roadmaps

### Week 3:
- ✅ Можно редактировать roadmaps (Manager)

### Week 4:
- ✅ Можно отмечать прогресс (Employee)

### Week 5:
- ✅ MVP в production, готов к первым клиентам

---

**Удачи! 🚀**

**P.S.** Помни: "Done is better than perfect". Главное — запустить и начать продавать. Всё остальное можно доделать по ходу.
