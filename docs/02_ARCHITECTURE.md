# 🏗️ Архитектура IndustrialPROFI

> **Паттерн:** Modern Monolith  
> **Философия:** KISS, YAGNI, No Premature Optimization

---

## 📐 Архитектурные Принципы

### 1. Modern Monolith (НЕ Микросервисы)

**Почему монолит:**
- ✅ Один разработчик
- ✅ Проще деплоить (один контейнер)
- ✅ Меньше сложности (нет network calls между сервисами)
- ✅ Easier debugging
- ✅ Подходит для 90% SaaS-бизнесов

**Когда переходить на микросервисы:**
- После 100K+ пользователей
- Когда команда больше 10 разработчиков
- Когда нужна независимая масштабируемость компонентов

### 2. SQLite в Production (Да, это Нормально)

**Rails 8 + SQLite = Production Ready:**
- WAL (Write-Ahead Logging) mode → поддержка concurrent reads
- Отлично для read-heavy приложений (наш случай)
- Backups через Litestream (continuous replication)
- Миграция на PostgreSQL занимает 1 день (если понадобится)

**Когда мигрировать на PostgreSQL:**
- Больше 1000 write операций/сек
- Нужен Full-Text Search (хотя SQLite FTS5 тоже хорош)
- Множество concurrent writers

### 3. Solid Stack (Без Redis)

**Rails 8 встроенные решения:**
- `Solid Queue` → background jobs (вместо Sidekiq)
- `Solid Cache` → кэширование (вместо Redis)
- `Solid Cable` → WebSockets (вместо Action Cable + Redis)

**Преимущества:**
- Меньше зависимостей
- Проще деплой
- Меньше инфраструктуры

---

## 🧱 Слои Архитектуры

```
┌─────────────────────────────────────────┐
│         Frontend (React 19)             │
│    ┌─────────────────────────────┐      │
│    │  Pages (Inertia Components) │      │
│    │  - Dashboard/Index.tsx      │      │
│    │  - Roadmaps/Show.tsx        │      │
│    │  - Roadmaps/Edit.tsx        │      │
│    └─────────────────────────────┘      │
│    ┌─────────────────────────────┐      │
│    │  Components (Reusable)      │      │
│    │  - RoadmapViewer            │      │
│    │  - SkillNode                │      │
│    │  - SkillSidebar             │      │
│    └─────────────────────────────┘      │
└─────────────────────────────────────────┘
                   ↕️
         (Inertia.js Protocol)
                   ↕️
┌─────────────────────────────────────────┐
│         Backend (Rails 8)               │
│    ┌─────────────────────────────┐      │
│    │  Controllers                │      │
│    │  - RoadmapsController       │      │
│    │  - UserProgressController   │      │
│    └─────────────────────────────┘      │
│    ┌─────────────────────────────┐      │
│    │  Services (Business Logic)  │      │
│    │  - RoadmapForkService       │      │
│    │  - AutoLayoutService        │      │
│    │  - PermitExpirationChecker  │      │
│    └─────────────────────────────┘      │
│    ┌─────────────────────────────┐      │
│    │  Models (ActiveRecord)      │      │
│    │  - Organization             │      │
│    │  - Roadmap                  │      │
│    │  - Skill                    │      │
│    │  - UserProgress             │      │
│    └─────────────────────────────┘      │
└─────────────────────────────────────────┘
                   ↕️
┌─────────────────────────────────────────┐
│       Database (SQLite)                 │
│    - organizations                      │
│    - roadmaps                           │
│    - skills                             │
│    - skill_dependencies                 │
│    - permit_templates (global)          │
│    - users                              │
│    - user_progresses                    │
│    - sessions (Rails 8 auth)            │
└─────────────────────────────────────────┘
```

---

## 🔄 Data Flow (Inertia Pattern)

### Стандартный Request Flow

```
1. User кликает ссылку:
   <Link href="/roadmaps/welder">Сварщик</Link>

2. Inertia.js отправляет XHR:
   GET /roadmaps/welder
   Headers: X-Inertia: true

3. Rails Controller:
   def show
     roadmap = Roadmap.find_by!(slug: params[:id])
     render inertia: 'Roadmaps/Show', props: {
       roadmap: serialize(roadmap),
       userProgress: current_user.progress_for(roadmap)
     }
   end

4. Inertia возвращает JSON:
   {
     "component": "Roadmaps/Show",
     "props": { ... },
     "url": "/roadmaps/welder"
   }

5. React рендерит компонент:
   import RoadmapsShow from '@/pages/Roadmaps/Show'
   <RoadmapsShow {...props} />
```

### Форма Submit Flow

```
1. User submits форму:
   <form onSubmit={() => router.post('/progress/123', data)}>

2. Inertia POST request:
   POST /progress/123
   Body: { status: 'completed' }

3. Rails Controller:
   def update
     progress.update!(params)
     redirect_back notice: "Сохранено"
   end

4. Inertia перезагружает текущую страницу с новыми props
   (или делает partial reload через `only: ['userProgress']`)
```

---

## 🗂️ Структура Проекта

```
v7-industrialprofi-platform-v7/
├── app/
│   ├── controllers/
│   │   ├── application_controller.rb
│   │   ├── roadmaps_controller.rb
│   │   ├── user_progress_controller.rb
│   │   └── organizations/
│   │       ├── roadmaps_controller.rb     # Namespace для B2B
│   │       ├── skills_controller.rb
│   │       └── employees_controller.rb
│   ├── models/
│   │   ├── organization.rb                # acts_as_tenant
│   │   ├── roadmap.rb
│   │   ├── skill.rb
│   │   ├── skill_dependency.rb
│   │   ├── permit_template.rb             # Глобальный каталог
│   │   ├── user.rb                        # belongs_to :organization
│   │   └── user_progress.rb
│   ├── services/
│   │   ├── roadmap_fork_service.rb        # Копирование графа
│   │   ├── auto_layout_service.rb         # Dagre integration
│   │   ├── roadmap_import_service.rb      # YAML → DB
│   │   └── permit_expiration_checker.rb   # Background job logic
│   ├── serializers/
│   │   ├── roadmap_serializer.rb          # Blueprinter
│   │   ├── skill_serializer.rb
│   │   └── user_progress_serializer.rb
│   ├── jobs/
│   │   └── expiring_permits_notifier_job.rb
│   └── frontend/
│       ├── entrypoints/
│       │   ├── application.js
│       │   ├── application.css            # Tailwind
│       │   └── inertia.jsx                # Inertia setup
│       ├── pages/
│       │   ├── Dashboard/
│       │   │   └── Index.tsx
│       │   ├── Roadmaps/
│       │   │   ├── Index.tsx              # Каталог roadmaps
│       │   │   ├── Show.tsx               # Viewer (read-only)
│       │   │   └── Edit.tsx               # Editor (Manager/Owner)
│       │   └── Organizations/
│       │       ├── Employees/
│       │       │   └── Index.tsx          # Список сотрудников
│       │       └── Settings/
│       │           └── Index.tsx
│       ├── components/
│       │   ├── ui/                        # shadcn/ui components
│       │   │   ├── Button.tsx
│       │   │   ├── Card.tsx
│       │   │   ├── Input.tsx
│       │   │   └── Modal.tsx
│       │   ├── layout/
│       │   │   ├── Navbar.tsx
│       │   │   ├── Sidebar.tsx
│       │   │   └── Layout.tsx
│       │   └── roadmap/
│       │       ├── RoadmapViewer.tsx      # React Flow wrapper
│       │       ├── SkillNode.tsx          # Custom node component
│       │       ├── SkillSidebar.tsx       # Details panel
│       │       ├── SkillForm.tsx          # CRUD form
│       │       └── PermitForm.tsx         # Форма допуска
│       ├── hooks/
│       │   ├── useCurrentUser.ts
│       │   └── useRoadmapLayout.ts        # Dagre wrapper
│       ├── types/
│       │   ├── models.ts                  # TypeScript interfaces
│       │   └── inertia.d.ts
│       └── utils/
│           ├── graphLayout.ts             # Dagre helper
│           └── dateFormat.ts
├── db/
│   ├── migrate/
│   ├── schema.rb
│   └── seeds/
│       ├── permit_templates.rb            # Типовые допуски СНГ
│       └── roadmaps/
│           ├── welder.yml
│           └── electrician.yml
├── config/
│   ├── routes.rb
│   ├── database.yml
│   └── deploy.yml                         # Kamal config
├── docs/                                  # ← Эта папка
└── package.json
```

---

## 🔐 Multi-Tenancy (acts_as_tenant)

### Схема работы

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :users
  has_many :roadmaps
end

# app/models/user.rb
class User < ApplicationRecord
  belongs_to :organization, optional: true  # NULL для super-admin
  acts_as_tenant :organization
end

# app/models/roadmap.rb
class Roadmap < ApplicationRecord
  belongs_to :organization, optional: true  # NULL = публичный roadmap
  acts_as_tenant :organization
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  set_current_tenant_through_filter
  before_action :set_tenant
  
  private
  
  def set_tenant
    if user_signed_in?
      set_current_tenant(current_user.organization)
    end
  end
end
```

### Два типа данных

#### 🌍 Глобальные (Shared):
**Без `organization_id`** — одинаковые для всех:
- `permit_templates` (типовые допуски)
- `roadmaps` где `organization_id = NULL` (публичные шаблоны)

#### 🔒 Тенантные (Tenant-Specific):
**С `organization_id`** — приватные данные клиента:
- `users`
- `user_progresses`
- `roadmaps` где `organization_id != NULL` (приватные копии)

---

## 🚀 Performance Optimization

### 1. Кэширование Структуры Графа

```ruby
# app/controllers/roadmaps_controller.rb
def show
  roadmap = Roadmap.find_by!(slug: params[:id])
  
  # Кэш структуры (nodes + edges) БЕЗ user progress
  graph_structure = Rails.cache.fetch(
    "roadmap/#{roadmap.id}/structure/v#{roadmap.updated_at.to_i}",
    expires_in: 1.hour
  ) do
    {
      nodes: roadmap.skills.as_json(only: [:id, :key, :title, :skill_type, :position_x, :position_y]),
      edges: roadmap.skill_dependencies.as_json(only: [:from_skill_id, :to_skill_id, :kind])
    }
  end
  
  # User progress — НЕ кэшируем (персональные данные)
  user_progress = current_user.user_progresses
    .where(skill_id: roadmap.skill_ids)
    .index_by(&:skill_id)
  
  render inertia: 'Roadmaps/Show', props: {
    roadmap: graph_structure,
    userProgress: user_progress
  }
end
```

### 2. Eager Loading (N+1 Prevention)

```ruby
# ❌ ПЛОХО (N+1 query)
roadmaps = Roadmap.all
roadmaps.each { |r| r.skills.count }  # N запросов

# ✅ ХОРОШО
roadmaps = Roadmap.includes(:skills)
roadmaps.each { |r| r.skills.count }  # 2 запроса
```

### 3. Partial Props Reload

```ruby
// Обновляем только прогресс, НЕ перезагружаем roadmap
router.post('/progress/123', data, {
  only: ['userProgress'],  // Inertia перезагрузит только этот проп
  preserveScroll: true
})
```

---

## 🔄 State Management

### React State (БЕЗ Redux/Zustand)

**Для MVP достаточно:**
- `useState` для локального state
- `useContext` для shared state (если нужно)
- Inertia props как source of truth

```tsx
// pages/Roadmaps/Show.tsx
const RoadmapsShow = ({ roadmap, userProgress }) => {
  const [selectedSkillId, setSelectedSkillId] = useState(null);
  
  return (
    <>
      <RoadmapViewer 
        nodes={roadmap.nodes}
        edges={roadmap.edges}
        onNodeClick={(id) => setSelectedSkillId(id)}
      />
      
      {selectedSkillId && (
        <SkillSidebar 
          skillId={selectedSkillId}
          progress={userProgress[selectedSkillId]}
          onClose={() => setSelectedSkillId(null)}
        />
      )}
    </>
  );
};
```

**Когда добавлять Redux:**
- После MVP
- Когда state становится слишком сложным
- Когда нужен undo/redo

---

## 📦 Deployment Architecture (Kamal 2)

```
Internet
    ↓
[Cloudflare DNS]
    ↓
[VPS Server (Hetzner)]
    ↓
[Traefik (Reverse Proxy)]
    ↓
┌─────────────────────────────────┐
│  Docker Container: Rails App    │
│  ┌───────────────────────────┐  │
│  │  Puma (Web Server)        │  │
│  │  Port: 3000               │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │  Solid Queue (Jobs)       │  │
│  │  Separate process         │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │  SQLite DB                │  │
│  │  Volume: /storage         │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
    ↓
[Litestream → Backups to S3]
```

---

## 🛡️ Security Best Practices

### 1. Multi-Tenancy Security
```ruby
# ВСЕГДА используй scoped queries через acts_as_tenant
# ❌ ОПАСНО
Roadmap.find(params[:id])  # Может получить чужой roadmap!

# ✅ БЕЗОПАСНО
current_organization.roadmaps.find(params[:id])
```

### 2. Strong Parameters
```ruby
def roadmap_params
  params.require(:roadmap).permit(:title, :description, :visibility)
end
```

### 3. CSRF Protection
Inertia.js автоматически включает CSRF token в каждый request.

### 4. Content Security Policy (CSP)
```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self
  policy.script_src  :self, :unsafe_inline  # Для Vite HMR
  policy.style_src   :self, :unsafe_inline
end
```

---

## 📊 Мониторинг (Post-MVP)

### Инструменты:
- **Error Tracking:** Sentry
- **Uptime Monitoring:** UptimeRobot
- **Performance:** Skylight / New Relic
- **Logs:** Papertrail

---

## 🔧 Dev vs Production Config

### Development:
- SQLite в `:memory:` или файл
- Vite HMR (Hot Module Replacement)
- Letter Opener для email preview

### Production:
- SQLite с WAL mode
- Precompiled assets (Vite build)
- SMTP для email (Postmark / SendGrid)

---

**Следующий документ:** `03_DATABASE.md` (детальная схема БД)
