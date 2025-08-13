# URL Shortening Service - High Level Design

## Table of Contents
- [Overview](#overview)
- [Requirements](#requirements)
- [System Design](#system-design)
- [Database Design](#database-design)
- [API Design](#api-design)
- [Architecture Components](#architecture-components)
- [Analytics System](#analytics-system)
- [Scalability Considerations](#scalability-considerations)
- [Trade-offs](#trade-offs)

## Overview

This document outlines the high-level design for a URL shortening service similar to TinyURL. The system converts long URLs into shorter, more manageable URLs and provides redirection services.

## Requirements

### Functional Requirements (FRs)
1. **URL Shortening**: Given a long URL, generate and return a shorter URL
2. **URL Redirection**: When a short URL is accessed, redirect users to the original long URL

### Non-Functional Requirements (NFRs)
1. **High Availability**: System must be available 24/7
2. **Low Latency**: Fast response times for both shortening and redirection
3. **Scalability**: Handle millions of requests per day
4. **Durability**: Store URLs for at least 10 years

## System Design

### URL Length Calculation

To determine the appropriate length for short URLs:

```
Total URLs needed = X requests/second × 60 × 60 × 24 × 365 × 10 years = Y

Character set: [a-z, A-Z, 0-9] = 62 characters
URL length needed: n where 62^n ≥ Y
```

**Standard Configuration:**
- **Character Set**: 62 characters (a-z, A-Z, 0-9)
- **URL Length**: 7 characters
- **Capacity**: 62^7 ≈ 3.5 trillion unique URLs

### High-Level Architecture

```mermaid
graph TD
    A[Client] --> B[Load Balancer]
    B --> C[Short URL Service<br/>Instance 1]
    B --> D[Short URL Service<br/>Instance 2]
    B --> E[Short URL Service<br/>Instance N]
    
    C --> F[Token Service]
    D --> F
    E --> F
    
    C --> G[Cassandra<br/>URL Storage]
    D --> G
    E --> G
    
    F --> H[MySQL<br/>Token Ranges]
    
    C --> I[Kafka<br/>Analytics]
    D --> I
    E --> I
    
    I --> J[Analytics Pipeline]
    J --> K[Analytics Dashboard]
```

## Database Design

### URL Storage (Cassandra)
```sql
Table: url_mappings
- short_url (TEXT, Primary Key)
- long_url (TEXT)
- created_at (TIMESTAMP)
- expires_at (TIMESTAMP)
```

### Token Management (MySQL)
```sql
Table: token_ranges
- range_id (INT, Primary Key, Auto Increment)
- start_range (BIGINT)
- end_range (BIGINT)
- assigned (BOOLEAN)
- assigned_to (VARCHAR)
- assigned_at (TIMESTAMP)
```

### URL Shortening Flow

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant SUS as Short URL Service
    participant TS as Token Service
    participant DB as Cassandra
    participant K as Kafka

    Client->>LB: POST /shorten {long_url}
    LB->>SUS: Route request
    
    alt Token needed
        SUS->>TS: Request token range
        TS->>SUS: Return range (e.g., 1001-2000)
    end
    
    SUS->>SUS: Generate short URL from token
    SUS->>DB: Store mapping (short_url, long_url)
    SUS-->>K: Log analytics (async)
    SUS->>LB: Return short URL
    LB->>Client: Response with short URL
```

### URL Redirection Flow

```mermaid
sequenceDiagram
    participant Client
    participant LB as Load Balancer
    participant SUS as Short URL Service
    participant DB as Cassandra
    participant K as Kafka

    Client->>LB: GET /abc123X
    LB->>SUS: Route request
    SUS->>DB: Query long URL by short URL
    DB->>SUS: Return long URL
    SUS-->>K: Log analytics (async)
    SUS->>LB: 302 Redirect to long URL
    LB->>Client: Redirect response
    Client->>Client: Navigate to long URL
```

### 1. Shorten URL
```http
POST /api/v1/shorten
Content-Type: application/json

{
  "long_url": "https://example.com/very/long/url"
}

Response:
{
  "short_url": "https://short.ly/aBc123X",
  "long_url": "https://example.com/very/long/url",
  "created_at": "2024-01-01T00:00:00Z"
}
```

### 2. Redirect URL
```http
GET /{short_code}

Response: 302 Redirect
Location: https://example.com/very/long/url
```

## Architecture Components

### 1. Short URL Service
- **Responsibilities**:
    - Generate short URLs using token ranges
    - Store URL mappings
    - Handle redirections
- **Scaling**: Multiple instances behind load balancer
- **Technology**: Java/Python with REST API

### 2. Token Service
- **Responsibilities**:
    - Distribute unique number ranges to service instances
    - Ensure no collisions in short URL generation
- **Implementation**: Single-threaded service with MySQL backend
- **Range Distribution**: Allocates ranges (e.g., 1-1000, 1001-2000)

### 3. Load Balancer
- **Function**: Distribute requests across service instances
- **Health Checks**: Monitor service instance availability
- **Technology**: NGINX/HAProxy

### 4. Database Layer
- **Cassandra**:
    - Primary storage for URL mappings
    - Handles high read/write throughput
    - Distributed and fault-tolerant
- **MySQL**:
    - Token range management
    - ACID compliance for range allocation

## Analytics System

### Analytics Data Collection Flow

```mermaid
graph LR
    A[Client Request] --> B[Short URL Service]
    B --> C[Generate Response]
    
    B --> D[Analytics Data<br/>- Timestamp<br/>- User Agent<br/>- IP Address<br/>- Referrer]
    
    subgraph "Analytics Processing Options"
        D --> E[Option 1: Async Kafka Write]
        D --> F[Option 2: Local Aggregation]
        
        F --> G[Batch Write to Kafka<br/>Every 10 seconds or<br/>Threshold reached]
        E --> H[Kafka Topic]
        G --> H
    end
    
    H --> I[Stream Processing<br/>Spark/Storm]
    H --> J[Batch Processing<br/>Hadoop/Hive]
    
    I --> K[Real-time Analytics DB]
    J --> L[Historical Analytics DB]
    
    K --> M[Analytics Dashboard]
    L --> M
```

### Analytics Architecture Options

```mermaid
graph TD
    subgraph "Option 1: Synchronous (High Latency)"
        A1[Request] --> B1[Process URL]
        B1 --> C1[Write to Kafka]
        C1 --> D1[Return Response]
    end
    
    subgraph "Option 2: Asynchronous (Low Latency)"
        A2[Request] --> B2[Process URL]
        B2 --> C2[Return Response]
        B2 --> D2[Async Write to Kafka]
    end
    
    subgraph "Option 3: Batch Processing (Best Performance)"
        A3[Request] --> B3[Process URL]
        B3 --> C3[Return Response]
        B3 --> D3[Add to Local Queue]
        D3 --> E3[Batch Write<br/>Every 10s or Threshold]
    end
```

### Implementation Options

#### Option 1: Asynchronous Kafka Writes
- Write analytics data to Kafka in parallel thread
- **Pros**: Low latency impact
- **Cons**: Potential data loss on failures

#### Option 2: Batch Processing
- Aggregate requests locally in memory
- Flush to Kafka periodically or on threshold
- **Pros**: Better performance, reduced I/O
- **Cons**: Higher potential data loss

### Analytics Data Points
- Request timestamp
- Short URL accessed
- User agent information
- Source IP address (for geographic analysis)
- Referrer information
- Device type

### Processing Pipeline
1. **Stream Processing**: Apache Spark/Storm for real-time analytics
2. **Batch Processing**: Hadoop/Hive for historical analysis
3. **Storage**: Time-series database for metrics
4. **Visualization**: Dashboard for insights

## Scalability Considerations

### System Scaling Strategy

```mermaid
graph TB
    subgraph "Geographic Distribution"
        US[US Data Center<br/>Primary]
        EU[EU Data Center<br/>Primary]
        ASIA[Asia Data Center<br/>Secondary]
        SA[S.America Data Center<br/>Secondary]
    end
    
    subgraph "Service Layer Scaling"
        LB1[Load Balancer US] --> SUS1[Service Instances<br/>Auto-scaling Group]
        LB2[Load Balancer EU] --> SUS2[Service Instances<br/>Auto-scaling Group]
    end
    
    subgraph "Data Layer Scaling"
        CASS1[(Cassandra Cluster<br/>Multi-region)]
        REDIS1[Redis Cache Layer]
        CDN[CDN for Static Assets]
    end
    
    US --> SUS1
    EU --> SUS2
    SUS1 --> CASS1
    SUS2 --> CASS1
    SUS1 --> REDIS1
    SUS2 --> REDIS1
    
    CDN --> US
    CDN --> EU
```

### Database Sharding Strategy

```mermaid
graph TD
    subgraph "Cassandra Sharding (Natural)"
        C1[Cassandra Node 1<br/>Partition Range: A-G]
        C2[Cassandra Node 2<br/>Partition Range: H-N]
        C3[Cassandra Node 3<br/>Partition Range: O-U]
        C4[Cassandra Node 4<br/>Partition Range: V-Z]
    end
    
    subgraph "MySQL Sharding (If Needed)"
        M1[MySQL Shard 1<br/>Token Ranges: 1-1M]
        M2[MySQL Shard 2<br/>Token Ranges: 1M-2M]
        M3[MySQL Shard 3<br/>Token Ranges: 2M-3M]
    end
    
    APP[Application Layer] --> C1
    APP --> C2
    APP --> C3
    APP --> C4
    
    TS[Token Service] --> M1
    TS --> M2
    TS --> M3
```

## Trade-offs

### Token Range Approach
**Pros:**
- Eliminates collisions
- Predictable URL generation
- No single point of failure (with multiple token services)

**Cons:**
- Wasted tokens on service restarts
- Additional complexity

### Analytics Implementation
**Trade-off**: Latency vs Data Accuracy
- **Synchronous**: High accuracy, higher latency
- **Asynchronous**: Low latency, potential data loss
- **Batch**: Best performance, highest potential data loss

### Database Choice
**Cassandra vs MySQL:**
- **Cassandra**: Better for high throughput, eventual consistency
- **MySQL**: Better for ACID requirements, complex queries

## Monitoring and Alerting

### Monitoring and System Health

```mermaid
graph TD
    subgraph "Application Metrics"
        A1[Request Throughput]
        A2[Response Latency P95/P99]
        A3[Error Rates]
        A4[URL Generation Rate]
    end
    
    subgraph "Infrastructure Metrics"
        I1[CPU/Memory Usage]
        I2[Database Performance]
        I3[Network I/O]
        I4[Token Service Availability]
    end
    
    subgraph "Business Metrics"
        B1[Popular URLs]
        B2[Geographic Distribution]
        B3[User Agent Analysis]
        B4[Traffic Patterns]
    end
    
    subgraph "Alerting System"
        AL1[PagerDuty/Slack Alerts]
        AL2[Health Check Dashboard]
        AL3[Automated Recovery]
    end
    
    A1 --> AL1
    A2 --> AL1
    I2 --> AL1
    I4 --> AL1
    
    A1 --> AL2
    A2 --> AL2
    A3 --> AL2
    I1 --> AL2
    B1 --> AL2
    B2 --> AL2
```

## Security Considerations

### URL Validation
- Validate input URLs for malicious content
- Implement rate limiting per user/IP
- Prevent abuse through monitoring

### Access Control
- API authentication for URL creation
- Admin interface for system management
- Audit logging for compliance

---

*This HLD provides a comprehensive foundation for building a scalable URL shortening service. Implementation details may vary based on specific requirements and constraints.*