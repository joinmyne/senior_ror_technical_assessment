# Senior Ruby on Rails Developer Technical Assessment

## Overview
This technical assessment evaluates your proficiency in building production-ready Ruby on Rails applications. You will create a task management API with role-based access control, background job processing, and comprehensive testing.

**Time Allocation:** 4-6 hours  
**Ruby Version:** 3.2+  
**Rails Version:** 8.1+  
**Database:** SQLite

---

## Project Requirements

### Application: Task Management System API

Build a RESTful API for a task management system where users can create, assign, and manage tasks with different permission levels.

---

## Technical Requirements

### 1. Authentication & Authorization (Devise & Pundit)

#### User Authentication
- Implement user authentication using **Devise**
- Support email/password authentication
- Include password reset functionality
- Generate authentication tokens for API access

#### Role-Based Access Control
- Create three user roles: `admin`, `manager`, and `member`
- Use **Pundit** for authorization policies
- Implement the following permission structure:

| Role | Create Tasks | Edit Own Tasks | Edit Any Task | Delete Any Task | Assign Tasks | View All Tasks |
|------|--------------|----------------|---------------|-----------------|--------------|----------------|
| Admin | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Manager | ✓ | ✓ | ✓ | ✗ | ✓ | ✓ |
| Member | ✓ | ✓ | ✗ | ✗ | ✗ | Own only |

**Deliverables:**
- `User` model with role enum
- Pundit policies for `User` and `Task` resources
- Policy scopes for role-based record filtering

---

### 2. Models & Database Design

#### Required Models

**User**
```ruby
# Fields:
- email (string, required, unique)
- encrypted_password (string, Devise)
- role (enum: admin, manager, member, default: member)
- first_name (string)
- last_name (string)
- timestamps

# Associations:
- has_many :created_tasks (class_name: 'Task', foreign_key: 'creator_id')
- has_many :assigned_tasks (class_name: 'Task', foreign_key: 'assignee_id')
```

**Task**
```ruby
# Fields:
- title (string, required)
- description (text)
- status (enum: pending, in_progress, completed, archived)
- priority (enum: low, medium, high, urgent)
- due_date (datetime)
- creator_id (references User)
- assignee_id (references User, optional)
- completed_at (datetime)
- timestamps

# Associations:
- belongs_to :creator (class_name: 'User')
- belongs_to :assignee (class_name: 'User', optional: true)
- has_many :comments, dependent: :destroy
```

**Comment**
```ruby
# Fields:
- content (text, required)
- task_id (references Task)
- user_id (references User)
- timestamps

# Associations:
- belongs_to :task
- belongs_to :user
```

**Requirements:**
- Add appropriate indexes for foreign keys and frequently queried fields
- Add database constraints where applicable
- Include proper validations on models

---

### 3. Scopes & Query Optimization

Implement the following **scopes** on the `Task` model:

```ruby
# Required Scopes:
- by_status(status) - Filter tasks by status
- by_priority(priority) - Filter tasks by priority
- overdue - Tasks past due_date and not completed
- upcoming(days = 7) - Tasks due within specified days
- completed_between(start_date, end_date) - Tasks completed in date range
- assigned_to(user) - Tasks assigned to specific user
- created_by(user) - Tasks created by specific user
- recent - Order by created_at DESC
- high_priority - Priority high or urgent
```

**Query Optimization Requirements:**
- Implement N+1 query prevention using `includes` or `eager_load`
- Add counter cache for task counts per user
- Create appropriate database indexes
- Demonstrate use of `select`, `pluck`, and `exists?` for efficiency
- Show understanding of when to use `find_each` for batch processing

**Deliverable:**
Create an endpoint that demonstrates optimized queries:
```
GET /api/v1/tasks/dashboard
```
Returns:
- Total tasks count by status
- Overdue tasks count
- User's assigned incomplete tasks (with creator info, no N+1)
- Recent activity (last 10 tasks with assignee and creator, no N+1)

---

### 4. Service Objects

Create service objects for complex business logic. Implement the **Command Pattern** or **Interactor Pattern**.

#### Required Services:

**TaskCreationService**
```ruby
# Responsibilities:
- Validate task parameters
- Create task record
- Assign default status
- Send notification to assignee (via Sidekiq)
- Return success/failure result object

# Usage:
result = TaskCreationService.call(user: current_user, params: task_params)
if result.success?
  # handle success
else
  # handle errors: result.errors
end
```

**TaskCompletionService**
```ruby
# Responsibilities:
- Mark task as completed
- Set completed_at timestamp
- Update related metrics
- Trigger notification to creator
- Archive if configured

# Usage:
result = TaskCompletionService.call(task: task, user: current_user)
```

**TaskAssignmentService**
```ruby
# Responsibilities:
- Check if user can assign tasks (authorization)
- Validate assignee exists
- Update task assignee
- Send assignment notification
- Log assignment history

# Usage:
result = TaskAssignmentService.call(
  task: task,
  assignee: new_assignee,
  assigned_by: current_user
)
```

**Requirements:**
- Each service should return a consistent result object
- Include error handling and validation
- Services should be testable in isolation
- Follow Single Responsibility Principle

---

### 5. Background Jobs (Sidekiq)

Implement **Sidekiq** for asynchronous job processing.

#### Required Jobs:

**TaskNotificationJob**
```ruby
# Trigger: When task is created or assigned
# Action: Send email notification to assignee
# Queue: :default
# Retry: 3 attempts
```

**TaskReminderJob**
```ruby
# Trigger: Scheduled daily (cron job)
# Action: Send reminders for tasks due in 24 hours
# Queue: :notifications
# Retry: 5 attempts
```

**TaskArchivalJob**
```ruby
# Trigger: Scheduled weekly
# Action: Archive completed tasks older than 30 days
# Queue: :low_priority
# Retry: 3 attempts
```

**DataExportJob**
```ruby
# Trigger: User request via API
# Action: Generate CSV of user's tasks and email download link
# Queue: :exports
# Retry: 2 attempts
# Track job progress using Sidekiq Pro features or custom implementation
```

**Requirements:**
- Configure multiple queues with different priorities
- Implement proper error handling and retry logic
- Show understanding of idempotency
- Add job-specific logging
- Configure Sidekiq middleware (optional but preferred)

---

### 6. API Design & Versioning

Create a RESTful API with proper versioning strategy.

#### API Structure:
```
/api/v1/*  - Version 1 endpoints
/api/v2/*  - Version 2 endpoints (demonstrate version differences)
```

#### Required Endpoints (V1):

**Authentication:**
```
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/signup
POST   /api/v1/auth/password/reset
```

**Users:**
```
GET    /api/v1/users          # List users (admin/manager only)
GET    /api/v1/users/:id      # Show user profile
PATCH  /api/v1/users/:id      # Update user
DELETE /api/v1/users/:id      # Delete user (admin only)
```

**Tasks:**
```
GET    /api/v1/tasks                    # List tasks (filtered by role)
POST   /api/v1/tasks                    # Create task
GET    /api/v1/tasks/:id                # Show task details
PATCH  /api/v1/tasks/:id                # Update task
DELETE /api/v1/tasks/:id                # Delete task
POST   /api/v1/tasks/:id/assign         # Assign task to user
POST   /api/v1/tasks/:id/complete       # Mark task complete
GET    /api/v1/tasks/dashboard          # Dashboard stats (optimized)
GET    /api/v1/tasks/overdue            # Overdue tasks
POST   /api/v1/tasks/:id/export         # Trigger export job
```

**Comments:**
```
GET    /api/v1/tasks/:task_id/comments
POST   /api/v1/tasks/:task_id/comments
DELETE /api/v1/tasks/:task_id/comments/:id
```

#### Version 2 Differences:
Create V2 with at least one breaking change to demonstrate versioning:
- Change response format (e.g., snake_case to camelCase)
- Add/remove fields from responses
- Change authentication mechanism
- Modify endpoint structure

**Versioning Strategy Options (choose one):**
1. URL path versioning (preferred for this test)
2. Header-based versioning
3. Media type versioning

**Requirements:**
- Implement proper HTTP status codes
- Use serializers (ActiveModel::Serializers or jsonapi-serializer)
- Include pagination for list endpoints
- Add filtering and sorting capabilities
- Implement rate limiting (bonus)
- Return consistent error responses

**Error Response Format:**
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "You are not authorized to perform this action",
    "details": {}
  }
}
```

---

### 7. Testing (RSpec)

Write comprehensive tests covering:

#### Model Tests (models/*)
```ruby
# User model
- Validations (email format, presence)
- Role enum behavior
- Associations
- Custom methods

# Task model
- Validations
- Status transitions
- Scopes (test each scope)
- Callbacks
- Associations

# Comment model
- Validations
- Associations
```

#### Policy Tests (policies/*)
```ruby
# TaskPolicy
- Test permissions for each role
- Test scope filtering by role
- Test edge cases (nil user, soft-deleted records)

# UserPolicy
- Test admin-only actions
- Test self-modification permissions
```

#### Service Tests (services/*)
```ruby
# TaskCreationService
- Successful task creation
- Validation failures
- Job enqueueing
- Return value structure

# TaskCompletionService
- Status transition
- Timestamp updates
- Notification triggering

# TaskAssignmentService
- Authorization checks
- Assignment logic
- Notification delivery
```

#### Request Tests (requests/api/v1/*)
```ruby
# Tasks API
- Authentication required
- Authorization enforced
- CRUD operations for each role
- Query parameter filtering
- Pagination
- Error responses
- Response format

# Dashboard endpoint
- Correct data aggregation
- No N+1 queries (use db query counting)
```

#### Job Tests (jobs/*)
```ruby
# TaskNotificationJob
- Email delivery
- Correct recipients
- Retry behavior
- Error handling

# TaskReminderJob
- Correct task selection
- Batch processing
- Performance with large datasets
```

**Testing Requirements:**
- Achieve >90% code coverage
- Use FactoryBot for test data
- Use Faker for realistic fake data
- Test both happy paths and edge cases
- Use RSpec best practices (let, let!, context, describe)
- Mock external dependencies (email, API calls)
- Use `aggregate_failures` for multiple expectations
- Database cleaner configuration

**Required Gems:**
- rspec-rails
- factory_bot_rails
- faker
- shoulda-matchers
- database_cleaner-active_record
- simplecov (for coverage reports)

---

### 8. Routes Configuration

Demonstrate advanced routing:

```ruby
# Required routing features:

# 1. API versioning with namespace
namespace :api do
  namespace :v1 do
    # resources
  end
  namespace :v2 do
    # resources
  end
end

# 2. Nested resources
resources :tasks do
  resources :comments, only: [:index, :create, :destroy]
  
  member do
    post :assign
    post :complete
    post :export
  end
  
  collection do
    get :dashboard
    get :overdue
  end
end

# 3. Custom routes with constraints
# API version selection via header (alternative approach)

# 4. Route concerns for reusable route patterns
concern :commentable do
  resources :comments, only: [:index, :create, :destroy]
end

# 5. Shallow nesting where appropriate

# 6. Custom error pages for 404, 500 (API JSON responses)
```

**Deliverables:**
- Organized routes file
- Use of concerns to DRY up routes
- Appropriate constraints
- RESTful design principles
- Comments explaining complex routing decisions

---

## Bonus Points

Implement any of the following for additional consideration:

1. **Caching Strategy**
   - Implement Redis caching for frequently accessed data
   - Cache dashboard statistics
   - Fragment caching in serializers

2. **Full-Text Search**
   - Search across title, description, and comments

3. **Real-time Updates**
   - WebSocket support with ActionCable
   - Broadcast task updates to relevant users

4. **API Documentation**
   - Swagger/OpenAPI documentation
   - Interactive API explorer

5. **Performance Monitoring**
   - Implement Bullet gem to detect N+1 queries
   - Add custom performance instrumentation

6. **Docker Setup**
   - Dockerize the application
   - docker-compose with Redis, PostgreSQL, and Sidekiq

8. **Webhook System**
   - Allow external systems to register webhooks
   - Trigger webhooks on task events

---

## Deliverables

Submit the following:

1. **GitHub Repository** containing:
   - Complete Rails application source code
   - README with setup instructions
   - Database schema documentation
   - API documentation (Postman collection or OpenAPI spec)
   - Environment variable template (.env.example)

2. **Documentation** (separate file or comprehensive README):
   - Architecture decisions and rationale
   - How to run the application locally
   - How to run the test suite
   - API endpoint examples with curl commands
   - Database ERD (Entity Relationship Diagram)
   - Explanation of service object design decisions
   - Sidekiq configuration and queue management

3. **Test Suite**:
   - All tests passing
   - Coverage report (>90% preferred)
   - Performance test examples

4. **Database Seeds**:
   - Seed file with sample data
   - At least 3 users (one of each role)
   - At least 20 tasks with various statuses
   - Comments on multiple tasks

---

## Evaluation Criteria

You will be evaluated on:

| Criteria | Weight | Details |
|----------|--------|---------|
| **Code Quality** | 25% | Clean code, Rails conventions, DRY principle, readability |
| **Architecture** | 20% | Service objects, separation of concerns, maintainability |
| **Authorization** | 15% | Proper Pundit implementation, security considerations |
| **Testing** | 15% | Comprehensive test coverage, test quality, edge cases |
| **Query Optimization** | 10% | N+1 prevention, appropriate indexing, efficient queries |
| **API Design** | 10% | RESTful principles, versioning strategy, error handling |
| **Background Jobs** | 5% | Proper Sidekiq usage, error handling, queue management |

---

## Setup Checklist

Before starting:
- [ ] Ruby 3.2+ installed
- [ ] Rails 8.1+ installed
- [ ] PostgreSQL running
- [ ] Redis running (for Sidekiq)
- [ ] Git initialized

Recommended gems to include:
```ruby
# Gemfile

# Core functionality
gem 'devise'                    # Authentication
gem 'pundit'                    # Authorization
gem 'sidekiq'                   # Background jobs
gem 'redis'                     # Cache and Sidekiq

# API
gem 'jsonapi-serializer'        # or active_model_serializers
gem 'rack-cors'                 # CORS
gem 'rack-attack'               # Rate limiting (bonus)

# Database
gem 'pg'                        # PostgreSQL
gem 'counter_culture'           # Counter cache

group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'pry-rails'
  gem 'bullet'                  # N+1 detection
end

group :test do
  gem 'shoulda-matchers'
  gem 'database_cleaner-active_record'
  gem 'simplecov', require: false
  gem 'webmock'                 # HTTP request stubbing
end
```

---

## Time Management Suggestions

- **Hour 1**: Setup, models, migrations, associations, validations
- **Hour 2**: Devise setup, Pundit policies, basic controllers
- **Hour 3**: Service objects, scopes, query optimization
- **Hour 4**: API endpoints, serializers, versioning
- **Hour 5**: Sidekiq jobs, background processing
- **Hour 6**: Comprehensive testing, documentation, cleanup

---

## Questions & Clarifications

If you have questions during the assessment:

1. Make reasonable assumptions and document them in your README
2. Focus on demonstrating your knowledge of the required technologies
3. Quality over quantity - well-implemented core features are better than incomplete bonus features

---

## Submission Instructions

1. Push your code to a public GitHub repository
2. Include a comprehensive README.md with setup instructions
3. Ensure all tests pass in CI (GitHub Actions or similar)
4. Email the repository link to: mkassab@joinmyne.com
5. Include in your email:
   - Total time spent
   - Any assumptions made
   - Challenges faced and how you overcame them
   - What you would improve with more time

---

**Good luck! We look forward to reviewing your submission.**
