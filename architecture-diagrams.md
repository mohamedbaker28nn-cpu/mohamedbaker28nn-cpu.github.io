# System Architecture Diagrams

## High-Level System Architecture

```mermaid
graph TB
    subgraph "Global Edge Layer"
        CF[Cloudflare CDN]
        DNS[Cloudflare DNS]
        SSL[SSL/TLS Termination]
    end
    
    subgraph "AWS Cloud Infrastructure"
        subgraph "Application Tier"
            ALB[Application Load Balancer]
            
            subgraph "ECS Fargate Cluster"
                FE[Next.js Frontend<br/>Port 3000]
                BE[NestJS Backend<br/>Port 4000]
            end
        end
        
        subgraph "Data Tier"
            Aurora[(Aurora Serverless v2<br/>PostgreSQL)]
            Redis[(ElastiCache Redis<br/>Session & Cache)]
        end
        
        subgraph "Video Processing Pipeline"
            S3[S3 Temporary Storage]
            SQS[SQS Queue]
            Lambda[Lambda Functions<br/>FFmpeg Encoding]
            R2[Cloudflare R2<br/>Video Storage]
        end
        
        subgraph "Authentication & Security"
            ST[SuperTokens<br/>Auth Service]
            KMS[AWS KMS<br/>Key Management]
        end
        
        subgraph "DNS & Networking"
            R53[Route53<br/>DNS Management]
            ACM[ACM<br/>SSL Certificates]
            VPC[VPC with<br/>Private Subnets]
        end
    end
    
    subgraph "External Services"
        NC[Namecheap API]
        GD[GoDaddy API]
        GAUTH[Google OAuth]
        GRAPE[GrapesJS CDN]
    end
    
    subgraph "Client Applications"
        INST[Instructor Dashboard]
        STUD[Student Portal]
        LAND[Custom Landing Pages]
    end
    
    %% Client to Edge
    INST --> CF
    STUD --> CF
    LAND --> CF
    
    %% Edge to Application
    CF --> ALB
    DNS --> R53
    SSL --> ACM
    
    %% Application Tier
    ALB --> FE
    ALB --> BE
    FE <--> BE
    
    %% Data Connections
    BE --> Aurora
    BE --> Redis
    BE --> ST
    
    %% Video Pipeline
    BE --> S3
    S3 --> SQS
    SQS --> Lambda
    Lambda --> R2
    CF --> R2
    
    %% External Integrations
    BE --> NC
    BE --> GD
    ST --> GAUTH
    FE --> GRAPE
    
    %% Security
    BE --> KMS
    Aurora --> KMS
    
    %% Styling
    classDef aws fill:#ff9900,stroke:#333,stroke-width:2px,color:#fff
    classDef cf fill:#f38020,stroke:#333,stroke-width:2px,color:#fff
    classDef app fill:#2196f3,stroke:#333,stroke-width:2px,color:#fff
    classDef data fill:#4caf50,stroke:#333,stroke-width:2px,color:#fff
    classDef ext fill:#9c27b0,stroke:#333,stroke-width:2px,color:#fff
    
    class ALB,Aurora,Redis,S3,SQS,Lambda,KMS,R53,ACM,VPC aws
    class CF,DNS,SSL,R2 cf
    class FE,BE,ST app
    class Aurora,Redis data
    class NC,GD,GAUTH,GRAPE ext
```

## Detailed Component Architecture

```mermaid
graph LR
    subgraph "Frontend Architecture"
        subgraph "Next.js Application"
            APP[App Router]
            PAGES[Page Components]
            COMP[Shared Components]
            HOOKS[Custom Hooks]
            STORE[Zustand Store]
        end
        
        subgraph "UI Libraries"
            TAIL[Tailwind CSS]
            HEADLESS[Headless UI]
            GRAPE[GrapesJS Builder]
            FORMS[React Hook Form]
        end
    end
    
    subgraph "Backend Architecture"
        subgraph "NestJS Application"
            GUARD[Auth Guards]
            CONTROLLER[Controllers]
            SERVICE[Services]
            REPOSITORY[Repositories]
            MIDDLEWARE[Middleware]
        end
        
        subgraph "Core Modules"
            TENANT[Tenant Module]
            COURSE[Course Module]
            USER[User Module]
            VIDEO[Video Module]
            DOMAIN[Domain Module]
        end
    end
    
    subgraph "Data Layer"
        PRISMA[Prisma ORM]
        CACHE[Redis Cache]
        DB[(PostgreSQL)]
    end
    
    APP --> PAGES
    PAGES --> COMP
    COMP --> HOOKS
    HOOKS --> STORE
    
    CONTROLLER --> GUARD
    CONTROLLER --> SERVICE
    SERVICE --> REPOSITORY
    REPOSITORY --> PRISMA
    PRISMA --> DB
    
    SERVICE --> CACHE
    MIDDLEWARE --> GUARD
```

## Multi-Tenant Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant CF as Cloudflare
    participant LB as Load Balancer
    participant FE as Frontend
    participant BE as Backend
    participant C as Cache
    participant DB as Database
    
    U->>CF: Request (tenant.domain.com)
    CF->>LB: Forward with headers
    LB->>FE: Route request
    FE->>BE: API call with tenant context
    
    BE->>C: Check tenant cache
    alt Cache Hit
        C->>BE: Return tenant data
    else Cache Miss
        BE->>DB: Query tenant by domain
        DB->>BE: Return tenant info
        BE->>C: Cache tenant data (1h TTL)
    end
    
    BE->>BE: Set tenant context
    BE->>DB: Query tenant-scoped data
    DB->>BE: Return filtered results
    BE->>FE: Tenant-specific response
    FE->>CF: Rendered page
    CF->>U: Cached response
```

## Video Processing Pipeline

```mermaid
graph TD
    subgraph "Upload Process"
        A[Instructor Upload] --> B[Presigned S3 URL]
        B --> C[Direct S3 Upload]
        C --> D[Upload Complete Webhook]
    end
    
    subgraph "Processing Queue"
        D --> E[SQS Message]
        E --> F[Lambda Trigger]
        F --> G[Download from S3]
    end
    
    subgraph "Video Encoding"
        G --> H[FFmpeg Processing]
        H --> I[1080p Stream]
        H --> J[720p Stream]
        H --> K[480p Stream]
        H --> L[360p Stream]
    end
    
    subgraph "Storage & Cleanup"
        I --> M[Upload to R2]
        J --> M
        K --> M
        L --> M
        M --> N[Update Database]
        N --> O[Delete S3 Original]
    end
    
    subgraph "CDN Distribution"
        M --> P[Cloudflare CDN]
        P --> Q[Global Edge Caching]
    end
```

## Authentication & Authorization Flow

```mermaid
graph TB
    subgraph "Authentication Flow"
        A[User Login] --> B{Auth Method}
        B -->|Email/Password| C[SuperTokens Validation]
        B -->|Google OAuth| D[Google Verification]
        C --> E[Generate Session]
        D --> E
        E --> F[Set HTTP-Only Cookies]
    end
    
    subgraph "Authorization Flow"
        G[API Request] --> H[Extract Session]
        H --> I[Validate with SuperTokens]
        I --> J{Valid Session?}
        J -->|Yes| K[Extract User Context]
        J -->|No| L[Return 401]
        K --> M[Check Tenant Access]
        M --> N{Authorized?}
        N -->|Yes| O[Process Request]
        N -->|No| P[Return 403]
    end
    
    F --> G
```

## Database Schema Relationships

```mermaid
erDiagram
    tenants ||--o{ courses : has
    tenants ||--o{ students : belongs_to
    tenants ||--|| landing_pages : has
    
    courses ||--o{ lectures : contains
    courses ||--o{ access_codes : has
    
    students ||--o{ enrollments : has
    courses ||--o{ enrollments : includes
    
    access_codes ||--o| students : used_by
    
    tenants {
        uuid id PK
        string email UK
        string subdomain UK
        string custom_domain UK
        string subscription_tier
        jsonb branding_config
        timestamp created_at
        timestamp updated_at
    }
    
    courses {
        uuid id PK
        uuid tenant_id FK
        string title
        text description
        string thumbnail_url
        boolean published
        timestamp created_at
        timestamp updated_at
    }
    
    lectures {
        uuid id PK
        uuid course_id FK
        string title
        string video_url
        integer duration
        integer order_index
        timestamp created_at
    }
    
    students {
        uuid id PK
        uuid tenant_id FK
        string email
        string name
        uuid[] enrolled_courses
        timestamp created_at
    }
    
    access_codes {
        uuid id PK
        uuid course_id FK
        string code UK
        uuid used_by FK
        timestamp used_at
        timestamp expires_at
        timestamp created_at
    }
    
    landing_pages {
        uuid id PK
        uuid tenant_id FK
        jsonb grapesjs_data
        text css_styles
        integer published_version
        timestamp created_at
        timestamp updated_at
    }
    
    enrollments {
        uuid id PK
        uuid student_id FK
        uuid course_id FK
        timestamp enrolled_at
        timestamp last_accessed
        jsonb progress
    }
```

## Infrastructure as Code Structure

```mermaid
graph TD
    subgraph "Terraform Modules"
        A[Root Module] --> B[VPC Module]
        A --> C[ECS Module]
        A --> D[Database Module]
        A --> E[Storage Module]
        A --> F[Security Module]
    end
    
    subgraph "VPC Module"
        B --> B1[Public Subnets]
        B --> B2[Private Subnets]
        B --> B3[NAT Gateway]
        B --> B4[Internet Gateway]
    end
    
    subgraph "ECS Module"
        C --> C1[Cluster]
        C --> C2[Task Definitions]
        C --> C3[Services]
        C --> C4[Load Balancer]
    end
    
    subgraph "Database Module"
        D --> D1[Aurora Cluster]
        D --> D2[Parameter Groups]
        D --> D3[Subnet Groups]
        D --> D4[Security Groups]
    end
    
    subgraph "Storage Module"
        E --> E1[S3 Buckets]
        E --> E2[R2 Configuration]
        E --> E3[Lambda Functions]
        E --> E4[SQS Queues]
    end
    
    subgraph "Security Module"
        F --> F1[KMS Keys]
        F --> F2[IAM Roles]
        F --> F3[Security Groups]
        F --> F4[SSL Certificates]
    end
```

## Deployment Pipeline

```mermaid
graph LR
    subgraph "Source Control"
        A[GitHub Repository]
        B[Feature Branch]
        C[Main Branch]
    end
    
    subgraph "CI/CD Pipeline"
        D[GitHub Actions]
        E[Build & Test]
        F[Security Scan]
        G[Deploy Staging]
        H[Integration Tests]
        I[Deploy Production]
    end
    
    subgraph "Infrastructure"
        J[Staging Environment]
        K[Production Environment]
        L[Monitoring & Alerts]
    end
    
    B --> D
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    G --> J
    I --> K
    K --> L
```

## Cost Optimization Strategy

```mermaid
graph TD
    subgraph "Cost Drivers"
        A[Compute Costs] --> A1[ECS Fargate]
        A --> A2[Lambda Executions]
        
        B[Storage Costs] --> B1[Aurora Storage]
        B --> B2[S3 Temporary]
        B --> B3[R2 Video Storage]
        
        C[Network Costs] --> C1[Data Transfer]
        C --> C2[Load Balancer]
        
        D[Service Costs] --> D1[Route53 Zones]
        D --> D2[SSL Certificates]
    end
    
    subgraph "Optimization Strategies"
        E[Shared Infrastructure] --> E1[Multi-tenant Architecture]
        E --> E2[Resource Pooling]
        
        F[Zero Egress] --> F1[Cloudflare R2]
        F --> F2[CDN Caching]
        
        G[Serverless Scaling] --> G1[Aurora Serverless]
        G --> G2[Lambda Functions]
        
        H[Intelligent Caching] --> H1[Redis Cache]
        H --> H2[Cloudflare Cache]
    end
    
    A1 --> E1
    B3 --> F1
    A2 --> G2
    C1 --> F2
```