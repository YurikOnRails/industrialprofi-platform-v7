# ⚙️ Backend: Rails Controllers, Models, Services

> **Framework:** Ruby on Rails 8.0.1  
> **Pattern:** Service Objects + Fat Models  
> **Auth:** Rails 8 Authentication Generator

---

## 📁 Структура Backend

```
app/
├── controllers/
│   ├── application_controller.rb
│   ├── sessions_controller.rb          # Authentication
│   ├── registrations_controller.rb
│   ├── dashboard_controller.rb
│   ├── roadmaps_controller.rb          # Публичные roadmaps (read-only)
│   └── organizations/                  # Namespace для B2B функций
│       ├── roadmaps_controller.rb      # CRUD roadmaps
│       ├── skills_controller.rb        # CRUD навыков
│       ├── dependencies_controller.rb  # CRUD связей
│       ├── employees_controller.rb     # Список сотрудников
│       └── progress_controller.rb      # Трекинг прогресса
├── models/
│   ├── organization.rb
│   ├── user.rb
│   ├── roadmap.rb
│   ├── skill.rb
│   ├── skill_dependency.rb
│   ├── permit_template.rb
│   └── user_progress.rb
├── services/
│   ├── roadmap_fork_service.rb
│   ├── auto_layout_service.rb
│   ├── roadmap_import_service.rb
│   └── permit_expiration_checker.rb
├── serializers/
│   ├── roadmap_serializer.rb
│   ├── skill_serializer.rb
│   └── user_progress_serializer.rb
└── jobs/
    └── expiring_permits_notifier_job.rb
```

---

## 🎛️ Controllers

### `ApplicationController`

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Inertia::Controller
  
  # Multi-tenancy setup
  set_current_tenant_through_filter
  before_action :set_tenant
  before_action :authenticate_user!
  
  # Inertia shared data (доступно во всех React компонентах)
  inertia_share do
    {
      auth: {
        user: current_user&.as_json(only: [:id, :email, :full_name, :role]),
        organization: current_organization&.as_json(only: [:id, :name, :slug, :plan_type])
      },
      flash: {
        success: flash[:notice],
        error: flash[:alert]
      }
    }
  end
  
  private
  
  def set_tenant
    if user_signed_in? && current_user.organization.present?
      set_current_tenant(current_user.organization)
    end
  end
  
  def current_organization
    current_user&.organization
  end
  helper_method :current_organization
  
  # Authorization helpers
  def require_manager!
    unless current_user.manager? || current_user.owner?
      redirect_to root_path, alert: "Доступ запрещен"
    end
  end
  
  def require_owner!
    unless current_user.owner?
      redirect_to root_path, alert: "Только владелец организации"
    end
  end
end
```

---

### `SessionsController` (Authentication)

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:new, :create]
  
  def new
    render inertia: 'Auth/Login'
  end
  
  def create
    user = User.find_by(email: params[:email])
    
    if user&.authenticate(params[:password])
      session = user.sessions.create!(
        ip_address: request.remote_ip,
        user_agent: request.user_agent
      )
      cookies.signed.permanent[:session_token] = { value: session.id, httponly: true }
      
      redirect_to root_path, notice: "Добро пожаловать!"
    else
      redirect_to login_path, alert: "Неверный email или пароль"
    end
  end
  
  def destroy
    cookies.delete(:session_token)
    redirect_to login_path, notice: "Вы вышли из системы"
  end
end
```

---

### `DashboardController`

```ruby
# app/controllers/dashboard_controller.rb
class DashboardController < ApplicationController
  def index
    # Мои roadmaps (к которым есть прогресс)
    my_roadmaps = current_user.user_progresses
      .joins(:skill)
      .joins(skill: :roadmap)
      .select('roadmaps.*, COUNT(user_progresses.id) as progress_count')
      .group('roadmaps.id')
    
    # Истекающие допуски (за 30 дней)
    expiring_permits = current_user.user_progresses
      .joins(skill: :permit_template)
      .where('expires_at <= ?', 30.days.from_now)
      .where('expires_at > ?', Date.today)
      .order(:expires_at)
    
    # Просроченные допуски
    expired_permits = current_user.user_progresses
      .joins(skill: :permit_template)
      .where('expires_at < ?', Date.today)
    
    render inertia: 'Dashboard/Index', props: {
      myRoadmaps: my_roadmaps.as_json,
      expiringPermits: expiring_permits.as_json(include: { skill: { include: :permit_template } }),
      expiredPermits: expired_permits.as_json(include: { skill: { include: :permit_template } })
    }
  end
end
```

---

### `RoadmapsController` (Публичные Roadmaps)

```ruby
# app/controllers/roadmaps_controller.rb
class RoadmapsController < ApplicationController
  skip_before_action :authenticate_user!, only: [:index, :show]
  
  def index
    # Публичные шаблоны
    public_roadmaps = Roadmap.where(visibility: 'public', is_template: true)
    
    # Если залогинен — показываем также roadmaps организации
    organization_roadmaps = if user_signed_in?
      current_organization&.roadmaps || []
    else
      []
    end
    
    render inertia: 'Roadmaps/Index', props: {
      publicRoadmaps: public_roadmaps.as_json,
      organizationRoadmaps: organization_roadmaps.as_json
    }
  end
  
  def show
    roadmap = Roadmap.find_by!(slug: params[:id])
    
    # Кэширование структуры графа
    graph_structure = Rails.cache.fetch(
      "roadmap/#{roadmap.id}/structure/v#{roadmap.updated_at.to_i}",
      expires_in: 1.hour
    ) do
      {
        id: roadmap.id,
        title: roadmap.title,
        description: roadmap.description,
        nodes: roadmap.skills.map do |skill|
          {
            id: skill.id.to_s,
            key: skill.key,
            type: 'skillNode',
            position: { x: skill.position_x, y: skill.position_y },
            data: {
              label: skill.title,
              description: skill.description,
              skillType: skill.skill_type,
              category: skill.category_label,
              categoryColor: skill.category_color,
              permitTemplate: skill.permit_template&.as_json(only: [:id, :title, :code, :expiration_months])
            }
          }
        end,
        edges: roadmap.skill_dependencies.map do |dep|
          {
            id: "e#{dep.from_skill_id}-#{dep.to_skill_id}",
            source: dep.from_skill_id.to_s,
            target: dep.to_skill_id.to_s,
            type: dep.kind == 'required' ? 'smoothstep' : 'step',
            animated: dep.kind == 'optional'
          }
        end
      }
    end
    
    # Прогресс пользователя (НЕ кэшируем)
    user_progress = if user_signed_in?
      current_user.user_progresses
        .where(skill_id: roadmap.skill_ids)
        .index_by(&:skill_id)
        .transform_values { |p| { status: p.status, expiresAt: p.expires_at } }
    else
      {}
    end
    
    render inertia: 'Roadmaps/Show', props: {
      roadmap: graph_structure,
      userProgress: user_progress
    }
  end
end
```

---

### `Organizations::RoadmapsController` (CRUD для Manager/Owner)

```ruby
# app/controllers/organizations/roadmaps_controller.rb
module Organizations
  class RoadmapsController < ApplicationController
    before_action :require_manager!, except: [:index, :show]
    
    def index
      roadmaps = current_organization.roadmaps.includes(:skills)
      
      render inertia: 'Organizations/Roadmaps/Index', props: {
        roadmaps: roadmaps.as_json(include: :skills)
      }
    end
    
    def new
      # Список публичных roadmaps для форка
      public_templates = Roadmap.where(visibility: 'public', is_template: true)
      
      render inertia: 'Organizations/Roadmaps/New', props: {
        publicTemplates: public_templates.as_json
      }
    end
    
    def create
      if params[:fork_from_id].present?
        # Fork публичного roadmap
        source = Roadmap.find(params[:fork_from_id])
        roadmap = RoadmapForkService.new(source, current_organization).call
        
        redirect_to edit_organizations_roadmap_path(roadmap), 
          notice: "Roadmap скопирован. Можете редактировать."
      else
        # Создание с нуля
        roadmap = current_organization.roadmaps.build(roadmap_params)
        
        if roadmap.save
          redirect_to edit_organizations_roadmap_path(roadmap)
        else
          redirect_back fallback_location: new_organizations_roadmap_path, 
            alert: roadmap.errors.full_messages.join(', ')
        end
      end
    end
    
    def edit
      roadmap = current_organization.roadmaps.find(params[:id])
      
      # Permit templates для добавления допусков
      permit_templates = PermitTemplate.where(country_code: 'RU', is_active: true)
      
      render inertia: 'Organizations/Roadmaps/Edit', props: {
        roadmap: RoadmapSerializer.new(roadmap).as_json(include_editor: true),
        permitTemplates: permit_templates.as_json
      }
    end
    
    def update
      roadmap = current_organization.roadmaps.find(params[:id])
      
      if roadmap.update(roadmap_params)
        # Очистить кэш структуры
        Rails.cache.delete("roadmap/#{roadmap.id}/structure/v#{roadmap.updated_at.to_i}")
        
        redirect_back fallback_location: edit_organizations_roadmap_path(roadmap),
          notice: "Сохранено"
      else
        redirect_back fallback_location: edit_organizations_roadmap_path(roadmap),
          alert: roadmap.errors.full_messages.join(', ')
      end
    end
    
    def destroy
      roadmap = current_organization.roadmaps.find(params[:id])
      roadmap.destroy!
      
      redirect_to organizations_roadmaps_path, notice: "Roadmap удален"
    end
    
    private
    
    def roadmap_params
      params.require(:roadmap).permit(:title, :slug, :description, :visibility, :theme_color)
    end
  end
end
```

---

### `Organizations::SkillsController` (CRUD Навыков)

```ruby
# app/controllers/organizations/skills_controller.rb
module Organizations
  class SkillsController < ApplicationController
    before_action :require_manager!
    before_action :set_roadmap
    
    def create
      skill = @roadmap.skills.build(skill_params)
      
      # Если не указаны координаты → auto-layout
      unless params[:position_x].present?
        positions = AutoLayoutService.new(@roadmap).calculate
        skill.position_x = positions.dig(skill.key, :x) || 0
        skill.position_y = positions.dig(skill.key, :y) || 0
      end
      
      if skill.save
        render json: skill.as_json, status: :created
      else
        render json: { errors: skill.errors.full_messages }, status: :unprocessable_entity
      end
    end
    
    def update
      skill = @roadmap.skills.find(params[:id])
      
      if skill.update(skill_params)
        render json: skill.as_json
      else
        render json: { errors: skill.errors.full_messages }, status: :unprocessable_entity
      end
    end
    
    def update_position
      skill = @roadmap.skills.find(params[:id])
      skill.update!(
        position_x: params[:x],
        position_y: params[:y],
        position_locked: true
      )
      
      head :ok
    end
    
    def destroy
      skill = @roadmap.skills.find(params[:id])
      skill.destroy!
      
      head :no_content
    end
    
    private
    
    def set_roadmap
      @roadmap = current_organization.roadmaps.find(params[:roadmap_id])
    end
    
    def skill_params
      params.require(:skill).permit(
        :key, :title, :description, :skill_type, :category_label, :category_color,
        :permit_template_id, :estimated_hours, :difficulty_level,
        :position_x, :position_y,
        resources: [:title, :url, :type]
      )
    end
  end
end
```

---

### `Organizations::ProgressController` (Трекинг Прогресса)

```ruby
# app/controllers/organizations/progress_controller.rb
module Organizations
  class ProgressController < ApplicationController
    def update
      skill = Skill.find(params[:skill_id])
      progress = current_user.user_progresses.find_or_initialize_by(skill: skill)
      
      if skill.skill_type == 'permit'
        # Форма допуска
        progress.assign_attributes(permit_params)
        
        # Автоматический расчет expires_at
        if progress.issued_at.present? && skill.permit_template.present?
          progress.expires_at = progress.issued_at + skill.permit_template.expiration_months.months
        end
        
        progress.status = 'completed'
      else
        # Обычный навык
        progress.status = params[:status] || 'completed'
        progress.completed_at = Time.current if progress.status == 'completed'
      end
      
      if progress.save
        render json: progress.as_json
      else
        render json: { errors: progress.errors.full_messages }, status: :unprocessable_entity
      end
    end
    
    private
    
    def permit_params
      params.require(:progress).permit(
        :certificate_number, :issued_at, :expires_at, :issuing_authority, :notes
      )
    end
  end
end
```

---

## 🧱 Models

### `Organization`

```ruby
# app/models/organization.rb
class Organization < ApplicationRecord
  has_many :users, dependent: :destroy
  has_many :roadmaps, dependent: :destroy
  
  validates :name, :slug, presence: true
  validates :slug, uniqueness: true, format: { with: /\A[a-z0-9\-]+\z/ }
  
  enum plan_type: { trial: 'trial', starter: 'starter', professional: 'professional', enterprise: 'enterprise' }
  enum subscription_status: { active: 'active', suspended: 'suspended', cancelled: 'cancelled' }
  
  # Проверка лимитов по тарифу
  def can_add_employee?
    users.count < employee_limit
  end
  
  def trial_active?
    plan_type == 'trial' && trial_ends_at.present? && trial_ends_at > Time.current
  end
end
```

---

### `User`

```ruby
# app/models/user.rb
class User < ApplicationRecord
  belongs_to :organization, optional: true
  has_many :sessions, dependent: :destroy
  has_many :user_progresses, dependent: :destroy
  
  has_secure_password
  
  acts_as_tenant :organization
  
  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :role, inclusion: { in: %w[employee manager owner] }
  
  enum role: { employee: 'employee', manager: 'manager', owner: 'owner' }
  
  # Проверка прав
  def can_edit_roadmaps?
    manager? || owner?
  end
  
  def progress_for(roadmap)
    user_progresses.where(skill_id: roadmap.skill_ids).index_by(&:skill_id)
  end
end
```

---

### `Roadmap`

```ruby
# app/models/roadmap.rb
class Roadmap < ApplicationRecord
  belongs_to :organization, optional: true  # NULL = публичный roadmap
  belongs_to :forked_from, class_name: 'Roadmap', optional: true
  
  has_many :skills, dependent: :destroy
  has_many :skill_dependencies, through: :skills, source: :dependencies_from
  has_many :forks, class_name: 'Roadmap', foreign_key: 'forked_from_id'
  
  acts_as_tenant :organization
  
  validates :title, :slug, presence: true
  validates :slug, uniqueness: { scope: :organization_id }
  validates :visibility, inclusion: { in: %w[public private unlisted] }
  
  enum visibility: { public: 'public', private: 'private', unlisted: 'unlisted' }
  
  # Scopes
  scope :public_templates, -> { where(organization_id: nil, visibility: 'public', is_template: true) }
  scope :for_organization, ->(org) { where(organization: org) }
  
  # Counter cache
  after_create :increment_fork_count, if: :forked_from_id?
  
  private
  
  def increment_fork_count
    forked_from.increment!(:fork_count)
  end
end
```

---

### `Skill`

```ruby
# app/models/skill.rb
class Skill < ApplicationRecord
  belongs_to :roadmap
  belongs_to :permit_template, optional: true
  
  has_many :dependencies_from, class_name: 'SkillDependency', foreign_key: 'from_skill_id', dependent: :destroy
  has_many :dependencies_to, class_name: 'SkillDependency', foreign_key: 'to_skill_id', dependent: :destroy
  has_many :prerequisite_skills, through: :dependencies_to, source: :from_skill
  has_many :dependent_skills, through: :dependencies_from, source: :to_skill
  
  has_many :user_progresses, dependent: :destroy
  
  validates :key, :title, :skill_type, presence: true
  validates :key, uniqueness: { scope: :roadmap_id }
  validates :skill_type, inclusion: { in: %w[skill permit milestone] }
  
  enum skill_type: { skill: 'skill', permit: 'permit', milestone: 'milestone' }
  
  # Callbacks
  before_validation :set_permit_data, if: :permit_template_id_changed?
  
  private
  
  def set_permit_data
    return unless permit_template.present?
    
    self.title ||= permit_template.title
    self.skill_type = 'permit'
    self.category_label ||= 'Допуски'
    self.category_color ||= '#ef4444'
  end
end
```

---

### `UserProgress`

```ruby
# app/models/user_progress.rb
class UserProgress < ApplicationRecord
  belongs_to :user
  belongs_to :skill
  belongs_to :verified_by, class_name: 'User', optional: true
  
  validates :status, inclusion: { in: %w[todo in_progress completed expired expiring_soon] }
  validates :skill_id, uniqueness: { scope: :user_id }
  
  # Валидация допусков
  validates :certificate_number, :issued_at, :expires_at, presence: true, if: -> { skill.permit? }
  validate :expires_after_issued, if: -> { issued_at.present? && expires_at.present? }
  
  enum status: {
    todo: 'todo',
    in_progress: 'in_progress',
    completed: 'completed',
    expired: 'expired',
    expiring_soon: 'expiring_soon'
  }
  
  # Scopes
  scope :expiring_soon, -> { where('expires_at <= ? AND expires_at > ?', 30.days.from_now, Date.today) }
  scope :expired, -> { where('expires_at < ?', Date.today) }
  
  private
  
  def expires_after_issued
    if expires_at <= issued_at
      errors.add(:expires_at, 'должен быть позже даты выдачи')
    end
  end
end
```

---

## 🛠️ Service Objects

### `RoadmapForkService`

```ruby
# app/services/roadmap_fork_service.rb
class RoadmapForkService
  def initialize(source_roadmap, target_organization)
    @source = source_roadmap
    @organization = target_organization
  end
  
  def call
    ActiveRecord::Base.transaction do
      # 1. Копируем roadmap
      forked_roadmap = @source.dup
      forked_roadmap.assign_attributes(
        organization: @organization,
        forked_from_id: @source.id,
        slug: generate_unique_slug,
        visibility: 'private',
        is_template: false
      )
      forked_roadmap.save!
      
      # 2. Копируем skills + создаем mapping ID
      id_mapping = {}
      @source.skills.each do |skill|
        new_skill = skill.dup
        new_skill.roadmap = forked_roadmap
        new_skill.save!
        id_mapping[skill.id] = new_skill.id
      end
      
      # 3. Копируем dependencies с новыми ID
      @source.skill_dependencies.each do |dep|
        SkillDependency.create!(
          from_skill_id: id_mapping[dep.from_skill_id],
          to_skill_id: id_mapping[dep.to_skill_id],
          kind: dep.kind
        )
      end
      
      forked_roadmap
    end
  end
  
  private
  
  def generate_unique_slug
    base_slug = @source.slug
    slug = base_slug
    counter = 1
    
    while @organization.roadmaps.exists?(slug: slug)
      slug = "#{base_slug}-#{counter}"
      counter += 1
    end
    
    slug
  end
end
```

---

### `AutoLayoutService`

```ruby
# app/services/auto_layout_service.rb
require 'json'

class AutoLayoutService
  NODE_WIDTH = 200
  NODE_HEIGHT = 80
  RANK_SEP = 100
  NODE_SEP = 50
  
  def initialize(roadmap)
    @roadmap = roadmap
  end
  
  def calculate
    # Простой алгоритм hierarchical layout
    # В продакшене можно использовать Dagre.js на клиенте
    
    levels = build_levels
    positions = {}
    
    levels.each_with_index do |level_skills, level_index|
      y = level_index * (NODE_HEIGHT + RANK_SEP)
      
      level_skills.each_with_index do |skill, skill_index|
        x = skill_index * (NODE_WIDTH + NODE_SEP)
        positions[skill.key] = { x: x, y: y }
      end
    end
    
    # Обновляем координаты в БД
    positions.each do |key, pos|
      skill = @roadmap.skills.find_by(key: key)
      skill.update_columns(position_x: pos[:x], position_y: pos[:y], position_locked: false)
    end
    
    positions
  end
  
  private
  
  def build_levels
    # Topological sort по зависимостям
    skills_by_level = {}
    visited = Set.new
    
    # Находим корневые узлы (без prerequisites)
    root_skills = @roadmap.skills.select { |s| s.prerequisite_skills.empty? }
    
    assign_level(root_skills, 0, skills_by_level, visited)
    
    # Конвертируем в массив массивов
    max_level = skills_by_level.keys.max || 0
    (0..max_level).map { |level| skills_by_level[level] || [] }
  end
  
  def assign_level(skills, level, skills_by_level, visited)
    skills.each do |skill|
      next if visited.include?(skill.id)
      
      skills_by_level[level] ||= []
      skills_by_level[level] << skill
      visited.add(skill.id)
      
      assign_level(skill.dependent_skills, level + 1, skills_by_level, visited)
    end
  end
end
```

---

**Продолжение следует:** `05_FRONTEND.md`, `06_FEATURES.md`, `07_TESTING.md`, `08_DEPLOYMENT.md`, `09_DEVELOPMENT_PLAN.md`
