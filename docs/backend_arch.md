# 🏗️ Beautiful Life - Golang Backend Architecture

*Comprehensive architecture documentation for the cancer support platform backend*

---

## 📋 Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [System Architecture Diagram](#system-architecture-diagram)
3. [Project Structure](#project-structure)
4. [Layer Architecture](#layer-architecture)
5. [Database Architecture](#database-architecture)
6. [API Design](#api-design)
7. [Security Architecture](#security-architecture)
8. [Integration Architecture](#integration-architecture)
9. [Deployment Architecture](#deployment-architecture)
10. [Monitoring & Observability](#monitoring--observability)

---

## 🎯 Architecture Overview

### Design Principles
- **Clean Architecture**: Separation of concerns with clear layer boundaries
- **Domain-Driven Design**: Business logic organized around core domains
- **Microservice-Ready**: Modular design for future service decomposition
- **API-First**: RESTful APIs with OpenAPI documentation
- **Security by Design**: HIPAA compliance and data protection built-in
- **Scalability**: Horizontal scaling and load balancing ready

### Technology Stack
```
┌─────────────────────────────────────────────────────────┐
│                    TECHNOLOGY STACK                     │
├─────────────────────────────────────────────────────────┤
│ Runtime:           Go 1.21+                            │
│ Web Framework:     Gin (HTTP Router & Middleware)       │
│ Database:          PostgreSQL 15+ with GORM ORM        │
│ Cache:             Redis 7+ (Sessions, Queues)         │
│ Message Queue:     Redis Streams / RabbitMQ            │
│ File Storage:      AWS S3 / MinIO                      │
│ Authentication:    JWT with RS256 signing              │
│ API Documentation: Swagger/OpenAPI 3.0                 │
│ Logging:           Structured logging with Logrus      │
│ Monitoring:        Prometheus + Grafana                │
│ Testing:           Testify + GoMock                     │
└─────────────────────────────────────────────────────────┘
```

---

## 🏛️ System Architecture Diagram

```mermaid
graph TB
    %% External Systems
    subgraph "External Systems"
        NEXTJS[Next.js Frontend<br/>SSR + API Routes]
        MOBILE[Mobile App<br/>React Native]
        WEBHOOK[Webhook Consumers<br/>Payment Gateways]
    end

    %% Load Balancer
    LB[Load Balancer<br/>Nginx/ALB]

    %% API Gateway Layer
    subgraph "API Gateway Layer"
        GATEWAY[API Gateway<br/>Rate Limiting<br/>Authentication]
    end

    %% Application Layer
    subgraph "Golang Backend Application"
        
        %% Web Layer
        subgraph "Web Layer (Port: 8080)"
            GIN[Gin Web Server]
            MIDDLEWARE[Middleware Stack<br/>- CORS<br/>- Auth<br/>- Logging<br/>- Rate Limiting]
            ROUTES[Route Handlers<br/>- REST APIs<br/>- Health Checks]
        end

        %% Service Layer
        subgraph "Service Layer (Business Logic)"
            AUTH_SVC[Authentication Service]
            USER_SVC[User Management Service]
            COMMUNITY_SVC[Community Service]
            MEDICAL_SVC[Medical Records Service]
            REMINDER_SVC[Reminder Service]
            AI_SVC[AI Assistant Service]
            PAYMENT_SVC[Payment Service]
            NOTIFICATION_SVC[Notification Service]
        end

        %% Repository Layer
        subgraph "Repository Layer (Data Access)"
            USER_REPO[User Repository]
            COMMUNITY_REPO[Community Repository]
            MEDICAL_REPO[Medical Repository]
            REMINDER_REPO[Reminder Repository]
            PAYMENT_REPO[Payment Repository]
        end

    end

    %% Background Services
    subgraph "Background Services"
        WORKER[Background Workers<br/>- Reminder Scheduler<br/>- Email Processor<br/>- File Processor]
        SCHEDULER[Cron Scheduler<br/>- Health Checks<br/>- Data Cleanup]
    end

    %% Data Layer
    subgraph "Data Layer"
        POSTGRES[(PostgreSQL<br/>Primary Database)]
        REDIS[(Redis<br/>Cache & Sessions)]
        S3[(AWS S3<br/>File Storage)]
    end

    %% External Services
    subgraph "External Services"
        OPENAI[OpenAI API<br/>AI Assistant]
        RAZORPAY[Razorpay<br/>Payment Gateway]
        TWILIO[Twilio<br/>SMS/Email]
        GOOGLE[Google APIs<br/>Calendar/Meet]
        FIREBASE[Firebase<br/>Push Notifications]
    end

    %% Monitoring
    subgraph "Monitoring & Observability"
        PROMETHEUS[Prometheus<br/>Metrics Collection]
        GRAFANA[Grafana<br/>Dashboards]
        LOKI[Loki<br/>Log Aggregation]
        JAEGER[Jaeger<br/>Distributed Tracing]
    end

    %% Flow Connections
    NEXTJS --> LB
    MOBILE --> LB
    WEBHOOK --> LB
    
    LB --> GATEWAY
    GATEWAY --> GIN
    
    GIN --> MIDDLEWARE
    MIDDLEWARE --> ROUTES
    
    ROUTES --> AUTH_SVC
    ROUTES --> USER_SVC
    ROUTES --> COMMUNITY_SVC
    ROUTES --> MEDICAL_SVC
    ROUTES --> REMINDER_SVC
    ROUTES --> AI_SVC
    ROUTES --> PAYMENT_SVC
    ROUTES --> NOTIFICATION_SVC
    
    AUTH_SVC --> USER_REPO
    USER_SVC --> USER_REPO
    COMMUNITY_SVC --> COMMUNITY_REPO
    MEDICAL_SVC --> MEDICAL_REPO
    REMINDER_SVC --> REMINDER_REPO
    PAYMENT_SVC --> PAYMENT_REPO
    
    USER_REPO --> POSTGRES
    COMMUNITY_REPO --> POSTGRES
    MEDICAL_REPO --> POSTGRES
    REMINDER_REPO --> POSTGRES
    PAYMENT_REPO --> POSTGRES
    
    AUTH_SVC --> REDIS
    NOTIFICATION_SVC --> REDIS
    REMINDER_SVC --> REDIS
    
    MEDICAL_SVC --> S3
    
    AI_SVC --> OPENAI
    PAYMENT_SVC --> RAZORPAY
    NOTIFICATION_SVC --> TWILIO
    COMMUNITY_SVC --> GOOGLE
    NOTIFICATION_SVC --> FIREBASE
    
    GIN --> PROMETHEUS
    WORKER --> PROMETHEUS
    PROMETHEUS --> GRAFANA
    
    GIN --> LOKI
    WORKER --> LOKI
    
    WORKER --> POSTGRES
    WORKER --> REDIS
    SCHEDULER --> POSTGRES

    %% Styling
    classDef external fill:#e1f5fe
    classDef web fill:#f3e5f5
    classDef service fill:#e8f5e8
    classDef data fill:#fff3e0
    classDef monitor fill:#fce4ec
    
    class NEXTJS,MOBILE,WEBHOOK external
    class GIN,MIDDLEWARE,ROUTES web
    class AUTH_SVC,USER_SVC,COMMUNITY_SVC,MEDICAL_SVC,REMINDER_SVC,AI_SVC,PAYMENT_SVC,NOTIFICATION_SVC service
    class POSTGRES,REDIS,S3 data
    class PROMETHEUS,GRAFANA,LOKI,JAEGER monitor
```

---

## 📁 Project Structure

```
backend/
├── cmd/
│   └── server/
│       ├── main.go                    # Application entry point
│       └── config.yaml               # Configuration file
├── internal/
│   ├── config/                       # Configuration management
│   │   ├── config.go                 # Config structure and loading
│   │   └── env.go                    # Environment variables
│   ├── domain/                       # Domain models (business entities)
│   │   ├── user/
│   │   │   ├── entity.go             # User domain entities
│   │   │   ├── repository.go         # User repository interface
│   │   │   └── service.go            # User service interface
│   │   ├── community/
│   │   │   ├── entity.go             # Community domain entities
│   │   │   ├── repository.go         # Community repository interface
│   │   │   └── service.go            # Community service interface
│   │   ├── medical/
│   │   │   ├── entity.go             # Medical records entities
│   │   │   ├── repository.go         # Medical repository interface
│   │   │   └── service.go            # Medical service interface
│   │   └── shared/
│   │       ├── errors.go             # Domain-specific errors
│   │       └── valueobjects.go       # Value objects
│   ├── infrastructure/               # External concerns (frameworks, DBs)
│   │   ├── database/
│   │   │   ├── postgres.go           # PostgreSQL connection
│   │   │   ├── migrations/           # Database migrations
│   │   │   └── seeds/                # Database seed data
│   │   ├── cache/
│   │   │   └── redis.go              # Redis connection and utilities
│   │   ├── storage/
│   │   │   ├── s3.go                 # AWS S3 client
│   │   │   └── local.go              # Local file storage (development)
│   │   ├── external/
│   │   │   ├── openai.go             # OpenAI API client
│   │   │   ├── razorpay.go           # Razorpay payment client
│   │   │   ├── twilio.go             # Twilio SMS/Email client
│   │   │   └── google.go             # Google APIs client
│   │   └── queue/
│   │       ├── redis_queue.go        # Redis-based message queue
│   │       └── worker.go             # Background worker implementation
│   ├── application/                  # Application services (use cases)
│   │   ├── auth/
│   │   │   ├── service.go            # Authentication service implementation
│   │   │   ├── jwt.go                # JWT token management
│   │   │   └── middleware.go         # Authentication middleware
│   │   ├── user/
│   │   │   ├── service.go            # User management service
│   │   │   ├── profile.go            # Profile management
│   │   │   └── verification.go       # User verification logic
│   │   ├── community/
│   │   │   ├── service.go            # Community service implementation
│   │   │   ├── posts.go              # Post management
│   │   │   ├── chat.go               # Chat functionality
│   │   │   └── video.go              # Video session management
│   │   ├── medical/
│   │   │   ├── service.go            # Medical records service
│   │   │   ├── upload.go             # File upload handling
│   │   │   └── sharing.go            # Secure sharing logic
│   │   ├── reminder/
│   │   │   ├── service.go            # Reminder service implementation
│   │   │   ├── scheduler.go          # Reminder scheduling logic
│   │   │   └── ai_suggestions.go     # AI-powered suggestions
│   │   └── notification/
│   │       ├── service.go            # Notification service
│   │       ├── email.go              # Email notifications
│   │       ├── sms.go                # SMS notifications
│   │       └── push.go               # Push notifications
│   ├── interfaces/                   # Interface adapters (controllers, presenters)
│   │   ├── http/
│   │   │   ├── server.go             # HTTP server setup
│   │   │   ├── middleware/
│   │   │   │   ├── auth.go           # Authentication middleware
│   │   │   │   ├── cors.go           # CORS middleware
│   │   │   │   ├── logging.go        # Request logging
│   │   │   │   ├── recovery.go       # Panic recovery
│   │   │   │   └── ratelimit.go      # Rate limiting
│   │   │   ├── handlers/
│   │   │   │   ├── auth.go           # Authentication endpoints
│   │   │   │   ├── users.go          # User management endpoints
│   │   │   │   ├── community.go      # Community endpoints
│   │   │   │   ├── medical.go        # Medical records endpoints
│   │   │   │   ├── reminders.go      # Reminder endpoints
│   │   │   │   ├── payments.go       # Payment endpoints
│   │   │   │   └── health.go         # Health check endpoints
│   │   │   ├── dto/
│   │   │   │   ├── request.go        # Request DTOs
│   │   │   │   ├── response.go       # Response DTOs
│   │   │   │   └── validation.go     # Input validation
│   │   │   └── routes/
│   │   │       ├── api.go            # API route definitions
│   │   │       └── swagger.go        # Swagger documentation
│   │   └── repository/               # Repository implementations
│   │       ├── postgres/
│   │       │   ├── user.go           # User repository implementation
│   │       │   ├── community.go      # Community repository implementation
│   │       │   ├── medical.go        # Medical repository implementation
│   │       │   └── reminder.go       # Reminder repository implementation
│   │       └── redis/
│   │           ├── session.go        # Session management
│   │           └── cache.go          # Caching implementation
│   └── pkg/                          # Shared packages
│       ├── utils/
│       │   ├── crypto.go             # Cryptographic utilities
│       │   ├── validation.go         # Validation helpers
│       │   ├── time.go               # Time utilities
│       │   └── strings.go            # String utilities
│       ├── logger/
│       │   └── logger.go             # Structured logging setup
│       ├── errors/
│       │   ├── types.go              # Error types
│       │   └── handler.go            # Error handling utilities
│       └── constants/
│           ├── roles.go              # User roles constants
│           ├── permissions.go        # Permission constants
│           └── status.go             # Status constants
├── scripts/
│   ├── migrate.sh                    # Database migration script
│   ├── seed.sh                       # Database seeding script
│   └── deploy.sh                     # Deployment script
├── docs/
│   ├── api/                          # API documentation
│   │   └── swagger.yaml              # OpenAPI specification
│   └── deployment/                   # Deployment guides
├── tests/
│   ├── integration/                  # Integration tests
│   ├── unit/                         # Unit tests
│   └── fixtures/                     # Test fixtures
├── docker/
│   ├── Dockerfile                    # Production Dockerfile
│   ├── Dockerfile.dev               # Development Dockerfile
│   └── docker-compose.yml           # Local development setup
├── go.mod                            # Go module definition
├── go.sum                            # Go module checksums
├── Makefile                          # Build and development commands
└── README.md                         # Project documentation
```

---

## 🏗️ Layer Architecture

### 1. **Domain Layer** (Core Business Logic)
```go
// internal/domain/user/entity.go
package user

import (
    "time"
    "github.com/google/uuid"
)

type User struct {
    ID               uuid.UUID         `json:"id"`
    Email            string           `json:"email"`
    PasswordHash     string           `json:"-"`
    Role             Role             `json:"role"`
    Profile          Profile          `json:"profile"`
    SubscriptionTier SubscriptionTier `json:"subscription_tier"`
    IsActive         bool             `json:"is_active"`
    IsVerified       bool             `json:"is_verified"`
    CreatedAt        time.Time        `json:"created_at"`
    UpdatedAt        time.Time        `json:"updated_at"`
    LastActiveAt     *time.Time       `json:"last_active_at"`
}

type Role string

const (
    RolePatient   Role = "patient"
    RoleDoctor    Role = "doctor"
    RoleCaregiver Role = "caregiver"
    RoleAdmin     Role = "admin"
)

type Profile struct {
    FirstName    string     `json:"first_name"`
    LastName     string     `json:"last_name"`
    DateOfBirth  *time.Time `json:"date_of_birth"`
    Gender       Gender     `json:"gender"`
    CancerType   string     `json:"cancer_type,omitempty"`
    DiagnosisDate *time.Time `json:"diagnosis_date,omitempty"`
    TreatmentStage TreatmentStage `json:"treatment_stage,omitempty"`
    Location     Location   `json:"location"`
    IsPublic     bool       `json:"is_public"`
}

// Business logic methods
func (u *User) CanAccessMedicalRecord(recordOwnerID uuid.UUID) bool {
    // Domain business rules for medical record access
    return u.ID == recordOwnerID || u.Role == RoleDoctor
}

func (u *User) UpdateLastActive() {
    now := time.Now()
    u.LastActiveAt = &now
}
```

### 2. **Application Layer** (Use Cases)
```go
// internal/application/user/service.go
package user

import (
    "context"
    "github.com/beautiful-life/internal/domain/user"
)

type Service interface {
    RegisterUser(ctx context.Context, req RegisterUserRequest) (*user.User, error)
    AuthenticateUser(ctx context.Context, email, password string) (*user.User, string, error)
    GetUserProfile(ctx context.Context, userID uuid.UUID) (*user.User, error)
    UpdateUserProfile(ctx context.Context, userID uuid.UUID, updates ProfileUpdates) error
    VerifyEmail(ctx context.Context, token string) error
}

type service struct {
    userRepo     user.Repository
    authService  auth.Service
    emailService notification.EmailService
    logger       logger.Logger
}

func NewService(
    userRepo user.Repository,
    authService auth.Service,
    emailService notification.EmailService,
    logger logger.Logger,
) Service {
    return &service{
        userRepo:     userRepo,
        authService:  authService,
        emailService: emailService,
        logger:       logger,
    }
}

func (s *service) RegisterUser(ctx context.Context, req RegisterUserRequest) (*user.User, error) {
    // Validation
    if err := req.Validate(); err != nil {
        return nil, errors.NewValidationError("invalid registration data", err)
    }

    // Check if user already exists
    existing, err := s.userRepo.GetByEmail(ctx, req.Email)
    if err != nil && !errors.IsNotFound(err) {
        return nil, err
    }
    if existing != nil {
        return nil, errors.NewConflictError("user already exists")
    }

    // Hash password
    hashedPassword, err := s.authService.HashPassword(req.Password)
    if err != nil {
        return nil, err
    }

    // Create user entity
    newUser := &user.User{
        ID:           uuid.New(),
        Email:        req.Email,
        PasswordHash: hashedPassword,
        Role:         req.Role,
        Profile:      req.Profile,
        IsActive:     true,
        IsVerified:   false,
        CreatedAt:    time.Now(),
        UpdatedAt:    time.Now(),
    }

    // Save to repository
    if err := s.userRepo.Create(ctx, newUser); err != nil {
        return nil, err
    }

    // Send verification email
    if err := s.sendVerificationEmail(ctx, newUser); err != nil {
        s.logger.Error("failed to send verification email", "user_id", newUser.ID, "error", err)
        // Don't fail registration if email fails
    }

    return newUser, nil
}
```

### 3. **Interface Layer** (Controllers/Handlers)
```go
// internal/interfaces/http/handlers/users.go
package handlers

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/beautiful-life/internal/application/user"
    "github.com/beautiful-life/internal/interfaces/http/dto"
)

type UserHandler struct {
    userService user.Service
    logger      logger.Logger
}

func NewUserHandler(userService user.Service, logger logger.Logger) *UserHandler {
    return &UserHandler{
        userService: userService,
        logger:      logger,
    }
}

// POST /api/v1/users/register
func (h *UserHandler) RegisterUser(c *gin.Context) {
    var req dto.RegisterUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, dto.ErrorResponse{
            Message: "Invalid request body",
            Details: err.Error(),
        })
        return
    }

    user, err := h.userService.RegisterUser(c.Request.Context(), user.RegisterUserRequest{
        Email:    req.Email,
        Password: req.Password,
        Role:     req.Role,
        Profile:  req.Profile.ToDomain(),
    })
    
    if err != nil {
        h.handleError(c, err)
        return
    }

    response := dto.UserResponse{
        ID:               user.ID,
        Email:            user.Email,
        Role:             string(user.Role),
        Profile:          dto.ProfileFromDomain(user.Profile),
        SubscriptionTier: string(user.SubscriptionTier),
        IsVerified:       user.IsVerified,
        CreatedAt:        user.CreatedAt,
    }

    c.JSON(http.StatusCreated, dto.SuccessResponse{
        Data:    response,
        Message: "User registered successfully",
    })
}

func (h *UserHandler) handleError(c *gin.Context, err error) {
    switch {
    case errors.IsValidationError(err):
        c.JSON(http.StatusBadRequest, dto.ErrorResponse{
            Message: "Validation failed",
            Details: err.Error(),
        })
    case errors.IsConflictError(err):
        c.JSON(http.StatusConflict, dto.ErrorResponse{
            Message: err.Error(),
        })
    default:
        h.logger.Error("internal server error", "error", err)
        c.JSON(http.StatusInternalServerError, dto.ErrorResponse{
            Message: "Internal server error",
        })
    }
}
```

### 4. **Infrastructure Layer** (External Dependencies)
```go
// internal/infrastructure/database/postgres.go
package database

import (
    "fmt"
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type PostgresConfig struct {
    Host         string
    Port         int
    Username     string
    Password     string
    DatabaseName string
    SSLMode      string
    MaxOpenConns int
    MaxIdleConns int
}

func NewPostgresConnection(config PostgresConfig) (*gorm.DB, error) {
    dsn := fmt.Sprintf(
        "host=%s user=%s password=%s dbname=%s port=%d sslmode=%s TimeZone=UTC",
        config.Host,
        config.Username,
        config.Password,
        config.DatabaseName,
        config.Port,
        config.SSLMode,
    )

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, fmt.Errorf("failed to get underlying sql.DB: %w", err)
    }

    sqlDB.SetMaxOpenConns(config.MaxOpenConns)
    sqlDB.SetMaxIdleConns(config.MaxIdleConns)

    return db, nil
}
```

---

## 🗄️ Database Architecture

### Entity Relationship Diagram
```mermaid
erDiagram
    USERS {
        uuid id PK
        string email UK
        string password_hash
        enum role
        jsonb profile
        enum subscription_tier
        boolean is_active
        boolean is_verified
        timestamp created_at
        timestamp updated_at
        timestamp last_active_at
    }

    COMMUNITIES {
        uuid id PK
        string name
        text description
        enum type
        enum privacy_level
        uuid creator_id FK
        jsonb settings
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    COMMUNITY_MEMBERS {
        uuid id PK
        uuid community_id FK
        uuid user_id FK
        enum role
        timestamp joined_at
        boolean is_active
    }

    COMMUNITY_POSTS {
        uuid id PK
        uuid community_id FK
        uuid author_id FK
        string title
        text content
        enum post_type
        jsonb metadata
        enum moderation_status
        int likes_count
        int comments_count
        timestamp created_at
        timestamp updated_at
    }

    POST_COMMENTS {
        uuid id PK
        uuid post_id FK
        uuid author_id FK
        text content
        uuid parent_comment_id FK
        int likes_count
        timestamp created_at
        timestamp updated_at
    }

    MEDICAL_RECORDS {
        uuid id PK
        uuid patient_id FK
        string title
        text description
        enum record_type
        string file_path
        string file_type
        int file_size
        jsonb metadata
        text encryption_key
        timestamp record_date
        timestamp created_at
        timestamp updated_at
    }

    MEDICAL_RECORD_SHARES {
        uuid id PK
        uuid record_id FK
        uuid shared_with_id FK
        uuid shared_by_id FK
        enum access_level
        timestamp expires_at
        timestamp created_at
    }

    REMINDERS {
        uuid id PK
        uuid user_id FK
        string title
        text description
        enum reminder_type
        jsonb schedule_config
        jsonb ai_suggestions
        boolean is_active
        timestamp next_due_at
        timestamp created_at
        timestamp updated_at
    }

    REMINDER_LOGS {
        uuid id PK
        uuid reminder_id FK
        enum status
        timestamp due_at
        timestamp completed_at
        text notes
        timestamp created_at
    }

    VIDEO_SESSIONS {
        uuid id PK
        uuid community_id FK
        uuid host_id FK
        string title
        text description
        string google_meet_link
        timestamp scheduled_at
        int duration_minutes
        enum status
        jsonb attendees
        timestamp created_at
    }

    CHAT_CONVERSATIONS {
        uuid id PK
        uuid community_id FK
        enum conversation_type
        jsonb participants
        timestamp last_message_at
        timestamp created_at
    }

    CHAT_MESSAGES {
        uuid id PK
        uuid conversation_id FK
        uuid sender_id FK
        text content
        enum message_type
        jsonb metadata
        boolean is_encrypted
        boolean is_deleted
        timestamp created_at
        timestamp updated_at
    }

    PAYMENTS {
        uuid id PK
        uuid user_id FK
        string external_payment_id
        decimal amount
        string currency
        enum status
        enum payment_method
        jsonb gateway_response
        timestamp paid_at
        timestamp created_at
    }

    SUBSCRIPTIONS {
        uuid id PK
        uuid user_id FK
        enum plan_type
        enum status
        timestamp starts_at
        timestamp ends_at
        timestamp canceled_at
        jsonb billing_info
        timestamp created_at
        timestamp updated_at
    }

    AI_INTERACTIONS {
        uuid id PK
        uuid user_id FK
        text query
        text response
        enum interaction_type
        jsonb context
        decimal cost
        timestamp created_at
    }

    NOTIFICATIONS {
        uuid id PK
        uuid user_id FK
        string title
        text message
        enum notification_type
        enum delivery_method
        enum status
        jsonb metadata
        timestamp sent_at
        timestamp read_at
        timestamp created_at
    }

    AUDIT_LOGS {
        uuid id PK
        uuid user_id FK
        string action
        string resource_type
        uuid resource_id
        jsonb old_values
        jsonb new_values
        string ip_address
        string user_agent
        timestamp created_at
    }

    %% Relationships
    USERS ||--o{ COMMUNITIES : creates
    USERS ||--o{ COMMUNITY_MEMBERS : joins
    USERS ||--o{ COMMUNITY_POSTS : authors
    USERS ||--o{ POST_COMMENTS : writes
    USERS ||--o{ MEDICAL_RECORDS : owns
    USERS ||--o{ REMINDERS : sets
    USERS ||--o{ VIDEO_SESSIONS : hosts
    USERS ||--o{ CHAT_MESSAGES : sends
    USERS ||--o{ PAYMENTS : makes
    USERS ||--o{ SUBSCRIPTIONS : has
    USERS ||--o{ AI_INTERACTIONS : interacts
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--o{ AUDIT_LOGS : generates

    COMMUNITIES ||--o{ COMMUNITY_MEMBERS : contains
    COMMUNITIES ||--o{ COMMUNITY_POSTS : contains
    COMMUNITIES ||--o{ VIDEO_SESSIONS : hosts
    COMMUNITIES ||--o{ CHAT_CONVERSATIONS : has

    COMMUNITY_POSTS ||--o{ POST_COMMENTS : receives
    POST_COMMENTS ||--o{ POST_COMMENTS : replies

    MEDICAL_RECORDS ||--o{ MEDICAL_RECORD_SHARES : shared
    
    REMINDERS ||--o{ REMINDER_LOGS : tracks

    CHAT_CONVERSATIONS ||--o{ CHAT_MESSAGES : contains
```

### Database Optimization Strategy

#### 1. **Indexing Strategy**
```sql
-- User lookups
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_last_active ON users(last_active_at);

-- Community queries
CREATE INDEX idx_community_members_community_user ON community_members(community_id, user_id);
CREATE INDEX idx_community_posts_community_created ON community_posts(community_id, created_at DESC);
CREATE INDEX idx_community_posts_author ON community_posts(author_id);

-- Medical records
CREATE INDEX idx_medical_records_patient ON medical_records(patient_id);
CREATE INDEX idx_medical_records_type ON medical_records(record_type);
CREATE INDEX idx_medical_record_shares_record ON medical_record_shares(record_id);

-- Reminders
CREATE INDEX idx_reminders_user_active ON reminders(user_id, is_active);
CREATE INDEX idx_reminders_next_due ON reminders(next_due_at) WHERE is_active = true;

-- Chat and messaging
CREATE INDEX idx_chat_messages_conversation ON chat_messages(conversation_id, created_at);
CREATE INDEX idx_chat_conversations_community ON chat_conversations(community_id);

-- Audit and analytics
CREATE INDEX idx_audit_logs_user_created ON audit_logs(user_id, created_at);
CREATE INDEX idx_notifications_user_status ON notifications(user_id, status);
```

#### 2. **Partitioning Strategy**
```sql
-- Partition large tables by date for better performance
CREATE TABLE audit_logs_y2024m01 PARTITION OF audit_logs
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE chat_messages_y2024m01 PARTITION OF chat_messages
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

## 🔌 API Design

### RESTful API Structure
```
/api/v1/
├── auth/
│   ├── POST /register              # User registration
│   ├── POST /login                 # User authentication
│   ├── POST /logout                # User logout
│   ├── POST /refresh               # Token refresh
│   ├── POST /forgot-password       # Password reset request
│   ├── POST /reset-password        # Password reset
│   └── GET  /verify-email/:token   # Email verification
├── users/
│   ├── GET    /profile             # Get current user profile
│   ├── PUT    /profile             # Update user profile
│   ├── GET    /profile/:id         # Get user profile by ID
│   ├── POST   /upload-avatar       # Upload profile picture
│   └── DELETE /account             # Delete user account
├── communities/
│   ├── GET    /                    # List communities
│   ├── POST   /                    # Create community
│   ├── GET    /:id                 # Get community details
│   ├── PUT    /:id                 # Update community
│   ├── DELETE /:id                 # Delete community
│   ├── POST   /:id/join            # Join community
│   ├── DELETE /:id/leave           # Leave community
│   ├── GET    /:id/members         # List community members
│   ├── GET    /:id/posts           # List community posts
│   ├── POST   /:id/posts           # Create community post
│   └── GET    /:id/video-sessions  # List video sessions
├── posts/
│   ├── GET    /:id                 # Get post details
│   ├── PUT    /:id                 # Update post
│   ├── DELETE /:id                 # Delete post
│   ├── POST   /:id/like            # Like/unlike post
│   ├── GET    /:id/comments        # List post comments
│   └── POST   /:id/comments        # Create comment
├── medical-records/
│   ├── GET    /                    # List medical records
│   ├── POST   /                    # Upload medical record
│   ├── GET    /:id                 # Get medical record
│   ├── PUT    /:id                 # Update medical record
│   ├── DELETE /:id                 # Delete medical record
│   ├── POST   /:id/share           # Share medical record
│   └── GET    /:id/download        # Download medical record
├── reminders/
│   ├── GET    /                    # List reminders
│   ├── POST   /                    # Create reminder
│   ├── GET    /:id                 # Get reminder details
│   ├── PUT    /:id                 # Update reminder
│   ├── DELETE /:id                 # Delete reminder
│   ├── POST   /:id/complete        # Mark reminder as completed
│   └── GET    /upcoming            # Get upcoming reminders
├── video-sessions/
│   ├── GET    /                    # List video sessions
│   ├── POST   /                    # Create video session
│   ├── GET    /:id                 # Get session details
│   ├── PUT    /:id                 # Update session
│   ├── DELETE /:id                 # Delete session
│   └── POST   /:id/join            # Join video session
├── chat/
│   ├── GET    /conversations       # List conversations
│   ├── POST   /conversations       # Create conversation
│   ├── GET    /conversations/:id/messages # Get messages
│   ├── POST   /conversations/:id/messages # Send message
│   └── PUT    /messages/:id        # Update message
├── ai/
│   ├── POST   /chat                # Chat with AI assistant
│   ├── POST   /analyze-symptoms    # Symptom analysis
│   ├── POST   /medication-check    # Medication interaction check
│   └── GET    /insights            # Personalized insights
├── payments/
│   ├── POST   /create-order        # Create payment order
│   ├── POST   /verify-payment      # Verify payment
│   ├── GET    /history             # Payment history
│   └── POST   /webhook             # Payment webhook
├── notifications/
│   ├── GET    /                    # List notifications
│   ├── PUT    /:id/read            # Mark as read
│   ├── DELETE /:id                 # Delete notification
│   └── PUT    /preferences         # Update notification preferences
└── admin/
    ├── GET    /users               # List all users
    ├── GET    /communities         # List all communities
    ├── GET    /reports             # System reports
    └── GET    /analytics           # System analytics
```

### API Response Format
```go
// Standard API Response Structure
type APIResponse struct {
    Success   bool        `json:"success"`
    Message   string      `json:"message,omitempty"`
    Data      interface{} `json:"data,omitempty"`
    Error     *APIError   `json:"error,omitempty"`
    Meta      *Meta       `json:"meta,omitempty"`
    Timestamp time.Time   `json:"timestamp"`
}

type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

type Meta struct {
    Page       int `json:"page,omitempty"`
    PerPage    int `json:"per_page,omitempty"`
    Total      int `json:"total,omitempty"`
    TotalPages int `json:"total_pages,omitempty"`
}
```

---

## 🔒 Security Architecture

### Authentication & Authorization Flow
```mermaid
sequenceDiagram
    participant Client as Next.js Frontend
    participant API as Golang API
    participant DB as PostgreSQL
    participant Redis as Redis Cache

    Note over Client,Redis: User Login Flow
    Client->>API: POST /api/v1/auth/login
    API->>DB: Validate credentials
    DB-->>API: User data
    API->>Redis: Store session
    API-->>Client: JWT token (HTTP-only cookie)

    Note over Client,Redis: Authenticated Request Flow
    Client->>API: API request with cookie
    API->>Redis: Validate session
    Redis-->>API: Session data
    API->>DB: Process request
    DB-->>API: Response data
    API-->>Client: API response

    Note over Client,Redis: Token Refresh Flow
    Client->>API: POST /api/v1/auth/refresh
    API->>Redis: Validate refresh token
    Redis-->>API: Valid session
    API->>Redis: Generate new tokens
    API-->>Client: New JWT token
```

### Security Middleware Stack
```go
// internal/interfaces/http/middleware/security.go
package middleware

func SecurityMiddleware() gin.HandlerFunc {
    return gin.HandlerFunc(func(c *gin.Context) {
        // Security headers
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Header("Content-Security-Policy", "default-src 'self'")
        
        c.Next()
    })
}

func RateLimitMiddleware(requests int, window time.Duration) gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Every(window/time.Duration(requests)), requests)
    
    return gin.HandlerFunc(func(c *gin.Context) {
        clientIP := c.ClientIP()
        
        if !limiter.Allow() {
            c.JSON(http.StatusTooManyRequests, gin.H{
                "error": "Rate limit exceeded",
            })
            c.Abort()
            return
        }
        
        c.Next()
    })
}
```

---

## 🔗 Integration Architecture

### External Service Integration
```go
// internal/infrastructure/external/openai.go
package external

type OpenAIClient struct {
    apiKey     string
    baseURL    string
    httpClient *http.Client
    logger     logger.Logger
}

func NewOpenAIClient(apiKey string, logger logger.Logger) *OpenAIClient {
    return &OpenAIClient{
        apiKey:  apiKey,
        baseURL: "https://api.openai.com/v1",
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
        logger: logger,
    }
}

func (c *OpenAIClient) ChatCompletion(ctx context.Context, messages []Message) (*ChatResponse, error) {
    // Implementation for OpenAI API integration
    // Include retry logic, error handling, and rate limiting
}

// internal/infrastructure/external/razorpay.go
package external

type RazorpayClient struct {
    keyID     string
    keySecret string
    baseURL   string
    logger    logger.Logger
}

func (c *RazorpayClient) CreateOrder(ctx context.Context, amount int, currency string) (*Order, error) {
    // Implementation for Razorpay payment integration
}
```

---

## 🚀 Deployment Architecture

### Docker Configuration
```dockerfile
# docker/Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main cmd/server/main.go

FROM alpine:latest
RUN apk --no-cache add ca-certificates tzdata
WORKDIR /root/

COPY --from=builder /app/main .
COPY --from=builder /app/cmd/server/config.yaml .

EXPOSE 8080
CMD ["./main"]
```

### Kubernetes Deployment
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beautiful-life-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: beautiful-life-backend
  template:
    metadata:
      labels:
        app: beautiful-life-backend
    spec:
      containers:
      - name: backend
        image: beautiful-life/backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## 📊 Monitoring & Observability

### Metrics Collection
```go
// internal/pkg/metrics/prometheus.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    HTTPRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    HTTPRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "Duration of HTTP requests",
        },
        []string{"method", "endpoint"},
    )

    ActiveUsers = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_users_total",
            Help: "Number of active users",
        },
    )
)
```

### Health Check Endpoints
```go
// internal/interfaces/http/handlers/health.go
func (h *HealthHandler) HealthCheck(c *gin.Context) {
    status := "healthy"
    checks := make(map[string]interface{})

    // Database check
    if err := h.db.Ping(); err != nil {
        status = "unhealthy"
        checks["database"] = "failed"
    } else {
        checks["database"] = "ok"
    }

    // Redis check
    if err := h.redis.Ping().Err(); err != nil {
        status = "unhealthy"
        checks["redis"] = "failed"
    } else {
        checks["redis"] = "ok"
    }

    c.JSON(http.StatusOK, gin.H{
        "status": status,
        "checks": checks,
        "timestamp": time.Now(),
    })
}
```

---

## 🔧 Development Workflow

### Makefile Commands
```makefile
# Makefile
.PHONY: build run test docker-build docker-run migrate seed lint

build:
	go build -o bin/server cmd/server/main.go

run:
	go run cmd/server/main.go

test:
	go test -v ./...

test-coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out

docker-build:
	docker build -f docker/Dockerfile -t beautiful-life-backend .

docker-run:
	docker-compose up -d

migrate:
	migrate -path internal/infrastructure/database/migrations -database "postgres://user:pass@localhost/dbname?sslmode=disable" up

seed:
	go run scripts/seed.go

lint:
	golangci-lint run

deps:
	go mod tidy
	go mod vendor

clean:
	rm -rf bin/
	docker-compose down
```

This comprehensive backend architecture provides a scalable, maintainable, and secure foundation for the Beautiful Life cancer support platform. The clean architecture pattern ensures separation of concerns, while the detailed project structure and implementation examples provide clear guidance for development.
