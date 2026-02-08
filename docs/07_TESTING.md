# 🧪 Стратегия Тестирования MVP

> **Цель:** Гарантировать работоспособность критичных функций перед деплоем  
> **Подход:** Pragmatic Testing (не 100% coverage, а умные приоритеты)

---

## 🎯 Философия Тестирования для Indie Hacker

### Что НЕ НУЖНО тестировать в MVP:
❌ Внутренние методы (private methods)  
❌ Простые геттеры/сеттеры  
❌ Стандартные CRUD операции Rails (если не переопределены)  
❌ CSS/UI компоненты (визуальная проверка руками)  
❌ Граничные случаи (edge cases) которые маловероятны  

### Что ОБЯЗАТЕЛЬНО нужно тестировать:
✅ **Critical User Flows** (регистрация, логин, создание roadmap)  
✅ **Business Logic** (fork roadmap, auto-layout, расчет expires_at)  
✅ **Multi-tenancy изоляция** (пользователь НЕ видит чужие данные)  
✅ **Permissions** (Employee НЕ может редактировать roadmaps)  
✅ **Background Jobs** (email уведомления)  

---

## 📊 Типы Тестов

### 1. Unit Tests (Модели)

**Цель:** Проверить бизнес-логику моделей и валидации

**Инструменты:**
- `minitest` (встроенный в Rails)
- `factory_bot` для фикстур

**Что тестируем:**

#### Organization Model
```ruby
# test/models/organization_test.rb
class OrganizationTest < ActiveSupport::TestCase
  test "slug should be unique" do
    org1 = create(:organization, slug: 'acme')
    org2 = build(:organization, slug: 'acme')
    
    assert_not org2.valid?
    assert_includes org2.errors[:slug], "has already been taken"
  end
  
  test "trial is active within 14 days" do
    org = create(:organization, 
      plan_type: 'trial', 
      trial_ends_at: 10.days.from_now
    )
    
    assert org.trial_active?
  end
  
  test "can_add_employee respects limit" do
    org = create(:organization, employee_limit: 2)
    create_list(:user, 2, organization: org)
    
    assert_not org.can_add_employee?
  end
end
```

#### UserProgress Model (Критичный!)
```ruby
# test/models/user_progress_test.rb
class UserProgressTest < ActiveSupport::TestCase
  test "permit expires_at calculated automatically" do
    permit_template = create(:permit_template, expiration_months: 12)
    skill = create(:skill, permit_template: permit_template, skill_type: 'permit')
    user = create(:user)
    
    progress = user.user_progresses.create!(
      skill: skill,
      certificate_number: "ABC123",
      issued_at: Date.parse("2024-01-01")
    )
    
    assert_equal Date.parse("2025-01-01"), progress.expires_at
  end
  
  test "expired scope finds expired permits" do
    expired_progress = create(:user_progress, expires_at: 1.day.ago)
    valid_progress = create(:user_progress, expires_at: 1.day.from_now)
    
    expired = UserProgress.expired
    
    assert_includes expired, expired_progress
    assert_not_includes expired, valid_progress
  end
end
```

---

### 2. Integration Tests (Controllers)

**Цель:** Проверить что HTTP endpoints работают корректно

#### Authentication
```ruby
# test/integration/authentication_test.rb
class AuthenticationTest < ActionDispatch::IntegrationTest
  test "user can sign up and is redirected to dashboard" do
    post registrations_path, params: {
      organization: { name: "Test Corp" },
      user: { email: "owner@test.com", password: "password123" }
    }
    
    assert_response :redirect
    follow_redirect!
    assert_equal dashboard_path, path
    assert_equal "owner@test.com", User.last.email
    assert_equal "owner", User.last.role
  end
  
  test "user cannot login with wrong password" do
    user = create(:user, password: "correct")
    
    post sessions_path, params: {
      email: user.email,
      password: "wrong"
    }
    
    assert_response :redirect
    follow_redirect!
    assert_match /неверный email или пароль/i, response.body
  end
end
```

#### Multi-Tenancy Isolation (КРИТИЧНО!)
```ruby
# test/integration/multi_tenancy_test.rb
class MultiTenancyTest < ActionDispatch::IntegrationTest
  test "user cannot access another organization's roadmap" do
    org1 = create(:organization)
    org2 = create(:organization)
    
    user1 = create(:user, organization: org1, role: 'manager')
    roadmap2 = create(:roadmap, organization: org2, visibility: 'private')
    
    sign_in user1
    
    get organizations_roadmap_path(roadmap2)
    
    assert_response :not_found  # или :forbidden
  end
  
  test "user can only see their organization's data in dashboard" do
    org1 = create(:organization)
    org2 = create(:organization)
    
    user1 = create(:user, organization: org1)
    user2 = create(:user, organization: org2)
    
    roadmap1 = create(:roadmap, organization: org1)
    roadmap2 = create(:roadmap, organization: org2)
    
    sign_in user1
    get dashboard_path
    
    assert_select "h2", text: roadmap1.title
    assert_select "h2", text: roadmap2.title, count: 0
  end
end
```

#### Permissions
```ruby
# test/integration/permissions_test.rb
class PermissionsTest < ActionDispatch::IntegrationTest
  test "employee cannot edit roadmap" do
    org = create(:organization)
    employee = create(:user, organization: org, role: 'employee')
    roadmap = create(:roadmap, organization: org)
    
    sign_in employee
    get edit_organizations_roadmap_path(roadmap)
    
    assert_response :redirect
    assert_match /доступ запрещен/i, flash[:alert]
  end
  
  test "manager can edit roadmap" do
    org = create(:organization)
    manager = create(:user, organization: org, role: 'manager')
    roadmap = create(:roadmap, organization: org)
    
    sign_in manager
    get edit_organizations_roadmap_path(roadmap)
    
    assert_response :success
  end
end
```

---

### 3. Service Object Tests

**Цель:** Проверить сложную бизнес-логику

#### RoadmapForkService (КРИТИЧНО!)
```ruby
# test/services/roadmap_fork_service_test.rb
class RoadmapForkServiceTest < ActiveSupport::TestCase
  test "forks roadmap with all skills and dependencies" do
    source = create(:roadmap, :with_skills, skills_count: 3)
    skill1, skill2, skill3 = source.skills.to_a
    
    create(:skill_dependency, from_skill: skill1, to_skill: skill2)
    create(:skill_dependency, from_skill: skill2, to_skill: skill3)
    
    target_org = create(:organization)
    
    forked = RoadmapForkService.new(source, target_org).call
    
    # Проверяем что roadmap скопирован
    assert_equal source.title, forked.title
    assert_equal target_org, forked.organization
    assert_equal source, forked.forked_from
    
    # Проверяем что skills скопированы
    assert_equal 3, forked.skills.count
    
    # Проверяем что dependencies скопированы (с новыми ID!)
    assert_equal 2, forked.skill_dependencies.count
    
    # Проверяем что координаты сохранены
    source_skill1 = skill1
    forked_skill1 = forked.skills.find_by(key: skill1.key)
    assert_equal source_skill1.position_x, forked_skill1.position_x
  end
  
  test "permit skills reference same permit_template" do
    permit_template = create(:permit_template)
    source = create(:roadmap)
    permit_skill = create(:skill, 
      roadmap: source, 
      skill_type: 'permit',
      permit_template: permit_template
    )
    
    target_org = create(:organization)
    forked = RoadmapForkService.new(source, target_org).call
    
    forked_permit = forked.skills.find_by(key: permit_skill.key)
    assert_equal permit_template, forked_permit.permit_template
  end
end
```

#### AutoLayoutService
```ruby
# test/services/auto_layout_service_test.rb
class AutoLayoutServiceTest < ActiveSupport::TestCase
  test "assigns coordinates to skills" do
    roadmap = create(:roadmap)
    skill1 = create(:skill, roadmap: roadmap, key: 'a')
    skill2 = create(:skill, roadmap: roadmap, key: 'b')
    
    create(:skill_dependency, from_skill: skill1, to_skill: skill2)
    
    positions = AutoLayoutService.new(roadmap).calculate
    
    # Проверяем что координаты рассчитаны
    assert positions['a'].present?
    assert positions['b'].present?
    
    # Проверяем что skill2 ниже skill1 (зависимость)
    assert positions['b'][:y] > positions['a'][:y]
    
    # Проверяем что координаты сохранены в БД
    skill1.reload
    assert_equal positions['a'][:x], skill1.position_x
  end
end
```

---

### 4. System Tests (End-to-End)

**Цель:** Проверить критичные user flows через браузер

**Инструменты:**
- Capybara + Selenium WebDriver
- Headless Chrome

#### Critical Flow: Sign Up → Fork Roadmap → Add Skill
```ruby
# test/system/roadmap_creation_test.rb
class RoadmapCreationTest < ApplicationSystemTestCase
  driven_by :selenium, using: :headless_chrome
  
  test "manager can fork public roadmap and add skill" do
    # Setup: создаем публичный roadmap
    public_roadmap = create(:roadmap, 
      visibility: 'public', 
      is_template: true,
      title: "Сварщик"
    )
    create(:skill, roadmap: public_roadmap, title: "Техника безопасности")
    
    # 1. Регистрация
    visit register_path
    fill_in "Название организации", with: "Test Corp"
    fill_in "Email", with: "owner@test.com"
    fill_in "Пароль", with: "password123"
    click_button "Зарегистрироваться"
    
    # 2. Переход в каталог roadmaps
    visit roadmaps_path
    assert_text "Сварщик"
    
    # 3. Fork roadmap
    within(".roadmap-card", text: "Сварщик") do
      click_button "Использовать шаблон"
    end
    
    # 4. Редактор открылся
    assert_current_path %r{/organizations/roadmaps/\d+/edit}
    assert_text "Техника безопасности"  # Skill скопирован
    
    # 5. Добавляем новый skill
    click_button "+ Навык"
    fill_in "Название", with: "MIG Сварка"
    fill_in "Ключ", with: "mig-welding"
    select "Навык", from: "Тип"
    click_button "Сохранить"
    
    # 6. Проверяем что skill добавлен
    assert_text "MIG Сварка"
    
    # 7. Проверяем что граф рендерится (React Flow)
    assert_selector ".react-flow", visible: true
  end
end
```

#### Critical Flow: Employee Marks Permit Complete
```ruby
# test/system/permit_completion_test.rb
class PermitCompletionTest < ApplicationSystemTestCase
  test "employee can complete permit and see expiration" do
    permit_template = create(:permit_template, 
      title: "Электробезопасность II группа",
      expiration_months: 12
    )
    
    org = create(:organization)
    employee = create(:user, organization: org, role: 'employee')
    roadmap = create(:roadmap, organization: org)
    permit_skill = create(:skill,
      roadmap: roadmap,
      skill_type: 'permit',
      permit_template: permit_template
    )
    
    sign_in employee
    visit roadmap_path(roadmap)
    
    # Клик по permit узлу
    find(".react-flow__node", text: permit_template.title).click
    
    # Sidebar открывается
    within(".skill-sidebar") do
      assert_text "Данные допуска"
      
      fill_in "Номер удостоверения", with: "ABC-123456"
      fill_in "Дата выдачи", with: "2024-01-01"
      fill_in "Кем выдано", with: "Ростехнадзор"
      
      click_button "Сохранить"
    end
    
    # Проверяем что прогресс сохранен
    progress = employee.user_progresses.last
    assert_equal "completed", progress.status
    assert_equal Date.parse("2025-01-01"), progress.expires_at
    
    # Узел стал зеленым
    assert_selector ".react-flow__node.node-completed", text: permit_template.title
  end
end
```

---

### 5. Background Job Tests

```ruby
# test/jobs/expiring_permits_notifier_job_test.rb
class ExpiringPermitsNotifierJobTest < ActiveJob::TestCase
  test "sends email for permits expiring in 30 days" do
    user = create(:user)
    progress = create(:user_progress,
      user: user,
      expires_at: 30.days.from_now.to_date
    )
    
    assert_enqueued_emails 1 do
      ExpiringPermitsNotifierJob.perform_now
    end
    
    email = ActionMailer::Base.deliveries.last
    assert_equal user.email, email.to.first
    assert_match /истекает через 30 дней/i, email.subject
  end
  
  test "marks expired permits" do
    expired_progress = create(:user_progress,
      status: 'completed',
      expires_at: 1.day.ago
    )
    
    ExpiringPermitsNotifierJob.perform_now
    
    expired_progress.reload
    assert_equal 'expired', expired_progress.status
  end
end
```

---

## 🚀 Запуск Тестов

### Локально (Dev Environment)

```bash
# Все тесты
bin/rails test

# Только модели
bin/rails test:models

# Только контроллеры
bin/rails test:controllers

# Только system tests
bin/rails test:system

# Конкретный файл
bin/rails test test/models/user_test.rb

# Конкретный тест
bin/rails test test/models/user_test.rb:10
```

### Перед Деплоем (Checklist)

```bash
# 1. Все тесты зеленые
bin/rails test
# Ожидаем: 0 failures, 0 errors

# 2. TypeScript компилируется
npm run type-check
# Ожидаем: 0 errors

# 3. Frontend билдится
npm run build
# Ожидаем: успешный build без ошибок

# 4. Database seeds работают
bin/rails db:reset
bin/rails db:seed
# Ожидаем: публичные roadmaps созданы, permit templates заполнены

# 5. Запуск в production mode локально
RAILS_ENV=production bin/rails assets:precompile
RAILS_ENV=production bin/rails server
# Открываем http://localhost:3000 и проверяем руками
```

---

## 📝 Test Coverage (Цель для MVP)

Не гонимся за 100%, но:

| Компонент | Цель Coverage | Приоритет |
|-----------|---------------|-----------|
| Models | 70%+ | Высокий |
| Services | 90%+ | Критичный |
| Controllers (critical) | 60%+ | Высокий |
| Background Jobs | 80%+ | Высокий |
| Frontend Components | 0% | Низкий (визуальная проверка) |

**Инструмент:** SimpleCov

```ruby
# Gemfile (test group)
gem 'simplecov', require: false

# test/test_helper.rb
require 'simplecov'
SimpleCov.start 'rails' do
  add_filter '/test/'
  add_filter '/config/'
end
```

---

## 🐛 Debugging Failed Tests

### Полезные команды:

```ruby
# В тесте: остановить выполнение
binding.break  # или binding.pry с pry gem

# Посмотреть SQL запросы
ActiveRecord::Base.logger = Logger.new(STDOUT)

# Посмотреть response body в integration test
puts response.body

# System test: сохранить screenshot при падении
take_screenshot  # автоматически сохраняется в tmp/screenshots/
```

---

## ✅ Manual Testing Checklist (Перед Деплоем)

### Регистрация и Логин
- [ ] Регистрация новой организации работает
- [ ] Email unique constraint работает
- [ ] Logout работает
- [ ] Redirect после логина на dashboard

### Roadmaps
- [ ] Каталог публичных roadmaps показывается
- [ ] Клик по roadmap открывает граф
- [ ] React Flow рендерится корректно
- [ ] Minimap работает
- [ ] Zoom/Pan работают

### Fork и Редактирование
- [ ] Fork roadmap создает копию со всеми skills
- [ ] Редактор открывается для Manager/Owner
- [ ] Добавление skill через форму работает
- [ ] Создание edge (drag connection) работает
- [ ] Auto-arrange рассчитывает координаты
- [ ] Сохранение roadmap работает

### Прогресс
- [ ] Employee может отметить skill completed
- [ ] Employee может заполнить форму permit
- [ ] Expires_at рассчитывается автоматически
- [ ] Узлы меняют цвет по статусу
- [ ] Dashboard показывает прогресс

### Permissions
- [ ] Employee НЕ видит кнопку "Редактировать roadmap"
- [ ] Employee НЕ может открыть `/organizations/roadmaps/:id/edit` напрямую
- [ ] Manager видит только свой отдел в матрице
- [ ] Multi-tenancy: пользователь НЕ видит чужие roadmaps

### Email (Production-like)
- [ ] Letter Opener работает в dev
- [ ] Email шаблон рендерится корректно
- [ ] Background job отправляет email за 30 дней

---

## 🔥 Критичные Баги (Найти ДО деплоя!)

### Top-5 потенциальных багов:

1. **Multi-tenancy leak**
   - Пользователь видит roadmaps другой организации
   - Проверка: попробовать открыть чужой URL напрямую

2. **N+1 queries**
   - Страница дашборда делает 100+ запросов
   - Проверка: `Bullet` gem в dev mode

3. **Permit expires_at не рассчитывается**
   - Забыли вызвать расчет после сохранения
   - Проверка: заполнить форму допуска, проверить БД

4. **React Flow граф не рендерится**
   - Vite не собрал CSS или JS не загрузился
   - Проверка: открыть DevTools → Console (не должно быть ошибок)

5. **Fork roadmap дублирует permit_templates**
   - Вместо ссылки создаются новые записи
   - Проверка: после форка `PermitTemplate.count` не увеличился

---

**Следующий документ:** `08_DEPLOYMENT.md`
