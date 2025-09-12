# ride-hail 🚗

## Learning Objectives

- Advanced Message Queue Patterns
- Real-Time Communication (WebSockets)
- Geospatial Data Processing
- Complex Microservices Orchestration
- High-Concurrency Programming
- Distributed State Management
- Service-Oriented Architecture (SOA) Design Patterns

## Abstract

In this project, you will build a real-time distributed ride-hailing platform using Service-Oriented Architecture (SOA) principles. Using Go, you will create multiple microservices that communicate through RabbitMQ message queues and WebSocket connections to handle ride requests, driver matching, live location tracking, and ride coordination. This system simulates the complex backend infrastructure of modern transportation platforms like Uber, teaching you how to design systems that handle real-time data streams, coordinate moving entities, and maintain consistency across distributed components.

This project represents the culmination of distributed systems knowledge: you'll implement sophisticated message routing patterns, handle geospatial calculations, manage real-time bidirectional communication, and orchestrate complex business workflows across multiple independent services following SOA best practices.

## Context

When you tap "Request Ride" in a ride-hailing app, what seems like a simple button press triggers an intricate choreography of distributed systems. Within seconds, your request must be broadcast to nearby drivers, optimal matches calculated based on distance and traffic, real-time locations tracked and updated, routes calculated and optimized, and all participants kept informed through live updates.

The challenge isn't just handling one ride request—it's handling thousands simultaneously. Drivers are constantly moving, changing the optimal matching calculations every second. Passengers cancel requests, drivers go offline, traffic conditions change routes, and payment systems must process transactions reliably. All of this must happen in real-time, with sub-second response times, while maintaining data consistency across multiple services.

This is where advanced message patterns and real-time communication become essential. Unlike traditional request-response systems, ride-hailing platforms operate on event streams: location updates, status changes, and matching events flow continuously through the system. Services must react to these events in real-time, making decisions based on constantly changing data while ensuring no rides are lost and no money goes missing.

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will automatically receive a score of `0`.
- Your program MUST compile successfully.
- Your program MUST NOT crash unexpectedly (any panics: `nil-pointer dereference`, `index out of range`, etc.). If this happens, you will receive `0` points during defense.
- Only built-in Go packages, `pgx/v5` PostgreSQL driver, the official AMQP client (`github.com/rabbitmq/amqp091-go`), Gorilla WebSocket (`github.com/gorilla/websocket`), and JSON Web Tokens (`github.com/golang-jwt/jwt/v5`) are allowed. If other packages are used, you will receive a score of `0`.
- RabbitMQ server MUST be running and available for connection.
- PostgreSQL database MUST be running and accessible for all services
- All RabbitMQ connections must handle reconnection scenarios
- Implement proper graceful shutdown for all services
- All database operations must be transactional where appropriate
- The project MUST compile with the following command in the project root directory:

```sh
$ go build -o ride-hail-system .
```

### Logging Format

All services must implement structured JSON logging to `stdout` with these mandatory fields:

| Field        | Type   | Description                         |
| ------------ | ------ | ----------------------------------- |
| `timestamp`  | string | ISO 8601 format timestamp           |
| `level`      | string | `INFO`, `DEBUG`, `ERROR`            |
| `service`    | string | Service name (e.g., `ride-service`) |
| `action`     | string | Event name (e.g., `ride_requested`) |
| `message`    | string | Human-readable description          |
| `hostname`   | string | Service hostname                    |
| `request_id` | string | Correlation ID for tracing          |
| `ride_id`    | string | Ride identifier (when applicable)   |

For ERROR logs, include an `error` object with `msg` and `stack` fields.

### Configuration

```yaml
# Database Configuration
database:
  host: ${DB_HOST:-localhost}
  port: ${DB_PORT:-5432}
  user: ${DB_USER:-ridehail_user}
  password: ${DB_PASSWORD:-ridehail_pass}
  database: ${DB_NAME:-ridehail_db}

# RabbitMQ Configuration
rabbitmq:
  host: ${RABBITMQ_HOST:-localhost}
  port: ${RABBITMQ_PORT:-5672}
  user: ${RABBITMQ_USER:-guest}
  password: ${RABBITMQ_PASSWORD:-guest}

# WebSocket Configuration
websocket:
  port: ${WS_PORT:-8080}

# Service Ports
services:
  ride_service: ${RIDE_SERVICE_PORT:-3000}
  driver_location_service: ${DRIVER_LOCATION_SERVICE_PORT:-3001}
  admin_service: ${ADMIN_SERVICE_PORT:-3004}
```

## System Architecture (SOA)

This system follows Service-Oriented Architecture (SOA) principles, where each service is:

- **Loosely coupled** - Services communicate through well-defined interfaces (APIs and message queues)
- **Reusable** - Each service can be scaled and deployed independently
- **Composable** - Services can be combined to create complex business workflows
- **Standards-based** - Uses standard protocols (HTTP/REST, WebSocket, AMQP)
- **Self-contained** - Each service owns its domain logic and data
- **Shared Database** - Multiple services can share a common database for consistency

Three microservices communicate through PostgreSQL and RabbitMQ to handle the complete ride-hailing workflow:

- **Ride Service** - Orchestrates the complete ride lifecycle and manages passenger interactions
- **Driver & Location Service** - Handles driver operations, matching algorithms, and real-time location tracking
- **Admin Service** - Provides monitoring, analytics, and system oversight capabilities

```
    +-------------+        +-------------------+              +-----------+
    |  Passenger  |        |    Ride Service   |              |   Admin   |
    | (WebSocket) |<------>|   (Orchestrator)  |              | Dashboard |
    +-------------+        +-------------------+              +-----------+
                                      ^                             ^
                                      |                             |
                                      v                             v
                         +-----------------------------------------------------+
                         |              RabbitMQ Message Broker                |
                         |                                                     |
                         |  Exchange: ride_topic                               |
                         |    - Queue: ride_requests      - New rides          |
                         |    - Queue: ride_status        - Status updates     |
                         |                                                     |
                         |  Exchange: driver_topic                             |
                         |    - Queue: driver_matching    - Match requests     |
                         |    - Queue: driver_responses   - Driver accepts     |
                         |                                                     |
                         |  Exchange: location_fanout                          |
                         |    - Queue: location_updates   - Location data      |
                         +-----------------------------------------------------+
                                         ^
                                         |
                                         v
  +-------------+             +-----------------------+
  |   Driver    |             |   Driver & Location   |
  | (WebSocket) |<----------> |      Service          |
  +-------------+             +-----------------------+
```

## Request Flow - Step by Step

**PHASE 1: RIDE REQUEST INITIATION**

![phase1.png](phase1.png)

**PHASE 2: DRIVER MATCHING PROCESS**

![phase2.png](phase2.png)

**PHASE 3: RIDE CONFIRMATION AND SETUP**

![phase3.png](phase3.png)


**PHASE 4: REAL-TIME TRACKING AND UPDATES**


![phase4.png](phase4.png)


**PHASE 5: RIDE EXECUTION AND COMPLETION**

![phase5.png](phase5.png)


## Services API and Events Overview

### Service Endpoints

| Service                       | Method | Endpoint                        | Description                 |
| ----------------------------- | ------ | ------------------------------- | --------------------------- |
| **Ride Service**              | POST   | `/rides`                        | Create a new ride request   |
| **Ride Service**              | POST   | `/rides/{ride_id}/cancel`       | Cancel a ride               |
| **Driver & Location Service** | POST   | `/drivers/{driver_id}/online`   | Driver goes online          |
| **Driver & Location Service** | POST   | `/drivers/{driver_id}/offline`  | Driver goes offline         |
| **Driver & Location Service** | POST   | `/drivers/{driver_id}/location` | Update driver location      |
| **Driver & Location Service** | POST   | `/drivers/{driver_id}/start`    | Start a ride                |
| **Driver & Location Service** | POST   | `/drivers/{driver_id}/complete` | Complete a ride             |
| **Admin Service**             | GET    | `/admin/overview`               | Get system metrics overview |
| **Admin Service**             | GET    | `/admin/rides/active`           | Get list of active rides    |

### WebSocket Connections

| Service                       | WebSocket URL                              | Purpose                             |
| ----------------------------- | ------------------------------------------ | ----------------------------------- |
| **Ride Service**              | `ws://{host}/ws/passengers/{passenger_id}` | WebSocket connection for passengers |
| **Driver & Location Service** | `ws://{host}/ws/drivers/{driver_id}`       | WebSocket connection for drivers    |

### WebSocket Connection Lifecycle

**Connection Flow:**

1. Client connects to WebSocket endpoint
2. Client sends authentication message within 5 seconds
3. Server validates token and establishes authenticated connection
4. Client and server exchange messages
5. Either party can close connection

**Authentication Message:**

```json
{
  "type": "auth",
  "token": "Bearer {jwt_token}"
}
```

**Keep-Alive:**

- Server sends ping every 30 seconds
- Client must respond with pong
- Connection closed if no pong received within 60 seconds

### Events Published (Outgoing)

| Publishing Service            | Destination                | Event Type                  | Description                                                                    |
| ----------------------------- | -------------------------- | --------------------------- | ------------------------------------------------------------------------------ |
| **Ride Service**              | `ride_topic` exchange      | `ride.request.{ride_type}`  | Driver match request message                                                   |
| **Ride Service**              | `ride_topic` exchange      | `ride.status.{status}`      | Ride status updates                                                            |
| **Ride Service**              | WebSocket (passengers)     | `ride_status_update`        | Status updates (MATCHED, EN_ROUTE, ARRIVED, IN_PROGRESS, COMPLETED, CANCELLED) |
| **Driver & Location Service** | `driver_topic` exchange    | `driver.response.{ride_id}` | Driver acceptance/rejection responses                                          |
| **Driver & Location Service** | `driver_topic` exchange    | `driver.status.{driver_id}` | Driver status changes                                                          |
| **Driver & Location Service** | `location_fanout` exchange | N/A (fanout)                | Location updates broadcast to all interested services                          |
| **Driver & Location Service** | WebSocket (drivers)        | `ride_offer`                | Ride offer notifications with timeout mechanism                                |
| **Driver & Location Service** | WebSocket (drivers)        | `ride_details`              | Ride confirmations with pickup details                                         |

### Events Consumed (Incoming)

| Consuming Service             | Source                     | Event Type                  | Description                          |
| ----------------------------- | -------------------------- | --------------------------- | ------------------------------------ |
| **Ride Service**              | `driver_topic` exchange    | `driver.response.{ride_id}` | Driver match responses               |
| **Ride Service**              | `driver_topic` exchange    | `driver.status.*`           | Driver status updates                |
| **Ride Service**              | `location_fanout` exchange | N/A (fanout)                | Location updates from drivers        |
| **Driver & Location Service** | `ride_topic` exchange      | `ride.request.*`            | Ride requests for driver matching    |
| **Driver & Location Service** | `ride_topic` exchange      | `ride.status.*`             | Ride status updates                  |
| **Driver & Location Service** | WebSocket (drivers)        | `ride_response`             | Driver acceptance/rejection messages |

### Message Queue Architecture

#### Exchanges

| Exchange Name     | Type            | Purpose                         |
| ----------------- | --------------- | ------------------------------- |
| `ride_topic`      | Topic Exchange  | Handles ride-related messages   |
| `driver_topic`    | Topic Exchange  | Handles driver-related messages |
| `location_fanout` | Fanout Exchange | Broadcasts location updates     |

#### Queues

| Queue Name              | Binding Exchange  | Routing Pattern     | Description                       |
| ----------------------- | ----------------- | ------------------- | --------------------------------- |
| `ride_requests`         | `ride_topic`      | `ride.request.*`    | New ride requests                 |
| `ride_status`           | `ride_topic`      | `ride.status.*`     | Ride status updates               |
| `driver_matching`       | `ride_topic`      | `ride.request.*`    | Driver matching requests          |
| `driver_responses`      | `driver_topic`    | `driver.response.*` | Driver acceptance responses       |
| `driver_status`         | `driver_topic`    | `driver.status.*`   | Driver status updates             |
| `location_updates_ride` | `location_fanout` | N/A (fanout)        | Location updates for ride service |

## Services

### Ride Service

**Core orchestrator managing ride lifecycle.**

#### Migrations

```sql
begin;

-- User roles enumeration
create table "roles"("value" text not null primary key);
insert into
    "roles" ("value")
values
    ('PASSENGER'),     -- Passenger/Customer
    ('DRIVER'),    -- Driver
    ('ADMIN')      -- Administrator
;

-- User status enumeration
create table "user_status"("value" text not null primary key);
insert into
    "user_status" ("value")
values
    ('ACTIVE'),    -- Active user
    ('INACTIVE'),  -- Inactive/Suspended
    ('BANNED')     -- Banned user
;

-- Main users table
create table users (
    id uuid primary key default gen_random_uuid(),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    email varchar(100) unique not null,
    role text references "roles"(value) not null,
    status text references "user_status"(value) not null default 'ACTIVE',
    password_hash text not null,
    attrs jsonb default '{}'::jsonb
);

-- Ride status enumeration
create table "ride_status"("value" text not null primary key);
insert into
    "ride_status" ("value")
values
    ('REQUESTED'),   -- Ride has been requested by customer
    ('MATCHED'),     -- Driver has been matched to the ride
    ('EN_ROUTE'),    -- Driver is on the way to pickup location
    ('ARRIVED'),     -- Driver has arrived at pickup location
    ('IN_PROGRESS'), -- Ride is currently in progress
    ('COMPLETED'),   -- Ride has been successfully completed
    ('CANCELLED')    -- Ride was cancelled
;

-- Ride type enumeration
create table "vehicle_type"("value" text not null primary key);
insert into
    "vehicle_type" ("value")
values
    ('ECONOMY'),     -- Standard economy ride
    ('PREMIUM'),     -- Premium comfort ride
    ('XL')           -- Extra large vehicle for groups
;

-- Coordinates table for real-time location tracking
create table coordinates (
    id uuid primary key default gen_random_uuid(),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    entity_id uuid not null, -- driver_id or passenger_id
    entity_type varchar(20) not null check (entity_type in ('driver', 'passenger')),
    address text not null,
    latitude decimal(10,8) not null check (latitude between -90 and 90),
    longitude decimal(11,8) not null check (longitude between -180 and 180),
    fare_amount decimal(10,2) check (fare_amount >= 0),
    distance_km decimal(8,2) check (distance_km >= 0),
    duration_minutes integer check (duration_minutes >= 0),
    is_current boolean default true
);

-- Create indexes for performance
create index idx_coordinates_entity on coordinates(entity_id, entity_type);
create index idx_coordinates_current on coordinates(entity_id, entity_type) where is_current = true;

-- Main rides table
create table rides (
    id uuid primary key default gen_random_uuid(),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    ride_number varchar(50) unique not null,
    passenger_id uuid not null references users(id),
    driver_id uuid references users(id),
    vehicle_type text references "vehicle_type"(value),
    status text references "ride_status"(value),
    priority integer default 1 check (priority between 1 and 10),
    requested_at timestamptz default now(),
    matched_at timestamptz,
    arrived_at timestamptz,
    started_at timestamptz,
    completed_at timestamptz,
    cancelled_at timestamptz,
    cancellation_reason text,
    estimated_fare decimal(10,2),
    final_fare decimal(10,2),
    pickup_coordinate_id uuid references coordinates(id),
    destination_coordinate_id uuid references coordinates(id)
);

-- Create index for status queries
create index idx_rides_status on rides(status);

-- Event type enumeration for audit trail
create table "ride_event_type"("value" text not null primary key);
insert into
    "ride_event_type" ("value")
values
    ('RIDE_REQUESTED'),    -- Initial ride request
    ('DRIVER_MATCHED'),    -- Driver assigned to ride
    ('DRIVER_ARRIVED'),    -- Driver arrived at pickup
    ('RIDE_STARTED'),      -- Ride began
    ('RIDE_COMPLETED'),    -- Ride finished
    ('RIDE_CANCELLED'),    -- Ride was cancelled
    ('STATUS_CHANGED'),    -- General status change
    ('LOCATION_UPDATED'),  -- Location update during ride
    ('FARE_ADJUSTED')      -- Fare was adjusted
;

-- Event sourcing table for complete ride audit trail
create table ride_events (
    id uuid primary key default gen_random_uuid(),
    created_at timestamptz not null default now(),
    ride_id uuid references rides(id) not null,
    event_type text references "ride_event_type"(value),
    event_data jsonb not null
);

/*
Example event_data field:
{
  "old_status": "requested",
  "new_status": "matched",
  "driver_id": "550e8400-e29b-41d4-a716-446655440001",
  "location": {"lat": 43.238949, "lng": 76.889709},
  "estimated_arrival": "2024-12-16T10:35:00Z"
}
*/

commit;
```

#### API

**Create Ride:**

```http
POST /rides
Content-Type: application/json
Authorization: Bearer {passenger_token}

{
  "passenger_id": "550e8400-e29b-41d4-a716-446655440001",
  "pickup_latitude": 43.238949,
  "pickup_longitude": 76.889709,
  "pickup_address": "Almaty Central Park",
  "destination_latitude": 43.222015,
  "destination_longitude": 76.851511,
  "destination_address": "Kok-Tobe Hill",
  "ride_type": "ECONOMY"
}
```

**Response (201 Created):**

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "ride_number": "RIDE_20241216_001",
  "status": "REQUESTED",
  "estimated_fare": 1450.0,
  "estimated_duration_minutes": 15,
  "estimated_distance_km": 5.2
}
```

**Cancel Ride:**

```http
POST /rides/{ride_id}/cancel
Content-Type: application/json
Authorization: Bearer {passenger_token}

{
  "reason": "Changed my mind"
}
```

**Response (200 OK):**

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "CANCELLED",
  "cancelled_at": "2024-12-16T10:33:00Z",
  "message": "Ride cancelled successfully"
}
```

#### Logic

1. **Validate request** including coordinate ranges and address verification
2. **Calculate fare** using dynamic pricing:
   - Base fare calculation: `base_fare + (distance_km * rate_per_km) + (duration_min * rate_per_min)`
   - Rates:
     - ECONOMY: 500₸ base, 100₸/km, 50₸/min
     - PREMIUM: 800₸ base, 120₸/km, 60₸/min
     - XL: 1000₸ base, 150₸/km, 75₸/min
3. **Store ride** with status 'REQUESTED' in transaction
4. **Publish** to `ride_topic` exchange with routing key `ride.request.{ride_type}`
5. **Start timeout timer** for driver matching (2 minutes)
6. **Handle driver responses** and update status to 'MATCHED'
7. **Track ride progress** through status transitions (ARRIVED, IN_PROGRESS, COMPLETED)
8. **Handle cancellations** with appropriate refund logic

#### Message Patterns

##### Outgoing Messages

**Driver Match Request** → `ride_topic` exchange → `ride.request.{ride_type}`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "ride_number": "RIDE_20241216_001",
  "pickup_location": {
    "lat": 43.238949,
    "lng": 76.889709,
    "address": "Almaty Central Park"
  },
  "destination_location": {
    "lat": 43.222015,
    "lng": 76.851511,
    "address": "Kok-Tobe Hill"
  },
  "ride_type": "ECONOMY",
  "estimated_fare": 1450.0,
  "max_distance_km": 5.0,
  "timeout_seconds": 30,
  "correlation_id": "req_123456"
}
```

**Location Update** ← `location_fanout` exchange

```json
{
  "driver_id": "660e8400-e29b-41d4-a716-446655440001",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "location": {
    "lat": 43.236,
    "lng": 76.886
  },
  "speed_kmh": 45.0,
  "heading_degrees": 180.0,
  "timestamp": "2024-12-16T10:35:30Z"
}
```

#### WebSocket Events

```
ws://{host}/ws/passengers/{passenger_id}
```

**Authentication:**

```json
{
  "type": "auth",
  "token": "Bearer {passenger_token}"
}
```

**To Passenger - Match Notification:**

```json
{
  "type": "ride_status_update",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "ride_number": "RIDE_20241216_001",
  "status": "MATCHED",
  "driver_info": {
    "driver_id": "660e8400-e29b-41d4-a716-446655440001",
    "name": "Aidar Nurlan",
    "rating": 4.8,
    "vehicle": {
      "make": "Toyota",
      "model": "Camry",
      "color": "White",
      "plate": "KZ 123 ABC"
    }
  },
  "correlation_id": "req_123456"
}
```

**Status Update** → `ride_topic` exchange → `ride.status.{status}`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "IN_PROGRESS",
  "timestamp": "2024-12-16T10:35:00Z",
  "driver_id": "660e8400-e29b-41d4-a716-446655440001",
  "correlation_id": "req_123456"
}
```

##### Incoming Messages

**Driver Match Response** ← `driver_topic` exchange ← `driver.response.{ride_id}`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "driver_id": "660e8400-e29b-41d4-a716-446655440001",
  "accepted": true,
  "estimated_arrival_minutes": 3,
  "driver_location": {
    "lat": 43.235,
    "lng": 76.885
  },
  "driver_info": {
    "name": "Aidar Nurlan",
    "rating": 4.8,
    "vehicle": {
      "make": "Toyota",
      "model": "Camry",
      "color": "White",
      "plate": "KZ 123 ABC"
    }
  },
  "driver_location": {
    "lat": 43.235,
    "lng": 76.885
  },
  "estimated_arrival": "2024-12-16T10:35:00Z"
}
```

**To Passenger - Location Update:**

```json
{
  "type": "driver_location_update",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "driver_location": {
    "lat": 43.236,
    "lng": 76.886
  },
  "estimated_arrival": "2024-12-16T10:34:00Z",
  "distance_to_pickup_km": 1.2
}
```

**To Passenger - Status Updates:**

```json
{
  "type": "ride_status_update",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "ARRIVED",
  "message": "Your driver has arrived at the pickup location"
}
```

### Driver & Location Service

**Combined service managing driver registration, availability, matching algorithm, and real-time location tracking.**

#### Migrations

```sql
begin;

-- Driver status enumeration
create table "driver_status"("value" text not null primary key);
insert into
    "driver_status" ("value")
values
    ('OFFLINE'),      -- Driver is not accepting rides
    ('AVAILABLE'),    -- Driver is available to accept rides
    ('BUSY'),         -- Driver is currently occupied
    ('EN_ROUTE')      -- Driver is on the way to pickup
;

-- Main drivers table
create table drivers (
    id uuid primary key references users(id),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    license_number varchar(50) unique not null,
    vehicle_type text references "vehicle_type"(value),
    vehicle_attrs jsonb,
    rating decimal(3,2) default 5.0 check (rating between 1.0 and 5.0),
    total_rides integer default 0 check (total_rides >= 0),
    total_earnings decimal(10,2) default 0 check (total_earnings >= 0),
    status text references "driver_status"(value),
    is_verified boolean default false
);

-- Create index for status queries
create index idx_drivers_status on drivers(status);

/* vehicle_attrs example:
{
  "vehicle_make": "Toyota",
  "vehicle_model": "Camry",
  "vehicle_color": "White",
  "vehicle_plate": "KZ 123 ABC",
  "vehicle_year": 2020
}
*/

-- Driver sessions for tracking online/offline times
create table driver_sessions (
    id uuid primary key default gen_random_uuid(),
    driver_id uuid references drivers(id) not null,
    started_at timestamptz not null default now(),
    ended_at timestamptz,
    total_rides integer default 0,
    total_earnings decimal(10,2) default 0
);

-- Location history for analytics and dispute resolution
create table location_history (
    id uuid primary key default gen_random_uuid(),
    coordinate_id uuid references coordinates(id),
    driver_id uuid references drivers(id),
    latitude decimal(10,8) not null check (latitude between -90 and 90),
    longitude decimal(11,8) not null check (longitude between -180 and 180),
    accuracy_meters decimal(6,2),
    speed_kmh decimal(5,2),
    heading_degrees decimal(5,2) check (heading_degrees between 0 and 360),
    recorded_at timestamptz not null default now(),
    ride_id uuid references rides(id)
);

commit;
```

#### API

**Go Online:**

```http
POST /drivers/{driver_id}/online
Content-Type: application/json
Authorization: Bearer {driver_token}

{
  "latitude": 43.238949,
  "longitude": 76.889709
}
```

**Response (200 OK):**

```json
{
  "status": "AVAILABLE",
  "session_id": "660e8400-e29b-41d4-a716-446655440001",
  "message": "You are now online and ready to accept rides"
}
```

**Go Offline:**

```http
POST /drivers/{driver_id}/offline
Content-Type: application/json
Authorization: Bearer {driver_token}
```

**Response (200 OK):**

```json
{
  "status": "OFFLINE",
  "session_id": "660e8400-e29b-41d4-a716-446655440001",
  "session_summary": {
    "duration_hours": 5.5,
    "rides_completed": 12,
    "earnings": 18500.0
  },
  "message": "You are now offline"
}
```

**Update Location:**

```http
POST /drivers/{driver_id}/location
Content-Type: application/json
Authorization: Bearer {driver_token}

{
  "latitude": 43.238949,
  "longitude": 76.889709,
  "accuracy_meters": 5.0,
  "speed_kmh": 45.0,
  "heading_degrees": 180.0
}
```

**Response (200 OK):**

```json
{
  "coordinate_id": "770e8400-e29b-41d4-a716-446655440002",
  "updated_at": "2024-12-16T10:30:00Z"
}
```

**Start Ride:**

```http
POST /drivers/{driver_id}/start
Content-Type: application/json
Authorization: Bearer {driver_token}

{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "driver_location": {
    "latitude": 43.238949,
    "longitude": 76.889709
  }
}
```

**Response (200 OK):**

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "BUSY",
  "started_at": "2024-12-16T10:35:00Z",
  "message": "Ride started successfully"
}
```

**Complete Ride:**

```http
POST /drivers/{driver_id}/complete
Content-Type: application/json
Authorization: Bearer {driver_token}

{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "final_location": {
    "latitude": 43.222015,
    "longitude": 76.851511
  },
  "actual_distance_km": 5.5,
  "actual_duration_minutes": 16
}
```

**Response (200 OK):**

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "AVAILABLE",
  "completed_at": "2024-12-16T10:51:00Z",
  "driver_earnings": 1216.0,
  "message": "Ride completed successfully"
}
```

#### Logic

1. **Driver Management:**

   - **Consume ride requests** from `driver_matching` queue
   - **Find nearby drivers** using PostGIS and real-time coordinates:

   ```sql
   SELECT d.id, u.email, d.rating, c.latitude, c.longitude,
          ST_Distance(
            ST_MakePoint(c.longitude, c.latitude)::geography,
            ST_MakePoint($1, $2)::geography
          ) / 1000 as distance_km
   FROM drivers d
   JOIN users u ON d.id = u.id
   JOIN coordinates c ON c.entity_id = d.id
     AND c.entity_type = 'driver'
     AND c.is_current = true
   WHERE d.status = 'AVAILABLE'
     AND d.vehicle_type = $3
     AND ST_DWithin(
           ST_MakePoint(c.longitude, c.latitude)::geography,
           ST_MakePoint($1, $2)::geography,
           5000  -- 5km radius
         )
   ORDER BY distance_km, d.rating DESC
   LIMIT 10;
   ```

   - **Send ride offers** via WebSocket to selected drivers
   - **Handle timeouts** for driver responses (30 seconds per offer)
   - **Update driver status** based on ride lifecycle

2. **Location Tracking:**
   - **Process real-time location updates** from drivers
   - **Update coordinates table** with current position (set previous to is_current=false)
   - **Archive previous coordinates** to location_history
   - **Calculate ETAs** based on distance and current speed
   - **Broadcast location updates** via fanout exchange
   - **Rate limit** location updates to prevent abuse (max 1 update per 3 seconds)

#### Message Patterns

##### Incoming Messages

**Driver Match Request** ← `ride_topic` exchange ← `ride.request.{ride_type}`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "ride_number": "RIDE_20241216_001",
  "pickup_location": {
    "lat": 43.238949,
    "lng": 76.889709,
    "address": "Almaty Central Park"
  },
  "destination_location": {
    "lat": 43.222015,
    "lng": 76.851511,
    "address": "Kok-Tobe Hill"
  },
  "ride_type": "ECONOMY",
  "estimated_fare": 1450.0,
  "max_distance_km": 5.0,
  "timeout_seconds": 30,
  "correlation_id": "req_123456"
}
```

**Ride Status Update** ← `ride_topic` exchange ← `ride.status.*`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "timestamp": "2024-12-16T10:51:00Z",
  "final_fare": 1520.0,
  "correlation_id": "req_123456"
}
```

##### Outgoing Messages

**Driver Match Response** → `driver_topic` exchange → `driver.response.{ride_id}`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "driver_id": "660e8400-e29b-41d4-a716-446655440001",
  "accepted": true,
  "estimated_arrival_minutes": 3,
  "driver_location": {
    "lat": 43.235,
    "lng": 76.885
  },
  "driver_info": {
    "name": "Aidar Nurlan",
    "rating": 4.8,
    "vehicle": {
      "make": "Toyota",
      "model": "Camry",
      "color": "White",
      "plate": "KZ 123 ABC"
    }
  },
  "correlation_id": "req_123456"
}
```

**Driver Status Update** → `driver_topic` exchange → `driver.status.{driver_id}`

```json
{
  "driver_id": "660e8400-e29b-41d4-a716-446655440001",
  "status": "BUSY",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2024-12-16T10:35:00Z"
}
```

**Location Update** → `location_fanout` exchange

```json
{
  "driver_id": "660e8400-e29b-41d4-a716-446655440001",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "location": {
    "lat": 43.236,
    "lng": 76.886
  },
  "speed_kmh": 45.0,
  "heading_degrees": 180.0,
  "timestamp": "2024-12-16T10:35:30Z"
}
```

#### WebSocket Events

```
ws://{host}/ws/drivers/{driver_id}
```

**Authentication:**

```json
{
  "type": "auth",
  "token": "Bearer {driver_token}"
}
```

**To Driver - Ride Offer:**

```json
{
  "type": "ride_offer",
  "offer_id": "offer_123456",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "ride_number": "RIDE_20241216_001",
  "pickup_location": {
    "latitude": 43.238949,
    "longitude": 76.889709,
    "address": "Almaty Central Park"
  },
  "destination_location": {
    "latitude": 43.222015,
    "longitude": 76.851511,
    "address": "Kok-Tobe Hill"
  },
  "estimated_fare": 1500.0,
  "driver_earnings": 1200.0,
  "distance_to_pickup_km": 2.1,
  "estimated_ride_duration_minutes": 15,
  "expires_at": "2024-12-16T10:32:00Z"
}
```

**From Driver - Ride Response:**

```json
{
  "type": "ride_response",
  "offer_id": "offer_123456",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "accepted": true,
  "current_location": {
    "latitude": 43.235,
    "longitude": 76.885
  }
}
```

**To Driver - Ride Details (After Acceptance):**

```json
{
  "type": "ride_details",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "passenger_name": "Saule Karimova",
  "passenger_phone": "+7-XXX-XXX-XX-XX",
  "pickup_location": {
    "latitude": 43.238949,
    "longitude": 76.889709,
    "address": "Almaty Central Park",
    "notes": "Near the main entrance"
  }
}
```

**From Driver - Location Update:**

```json
{
  "type": "location_update",
  "latitude": 43.236,
  "longitude": 76.886,
  "accuracy_meters": 5.0,
  "speed_kmh": 45.0,
  "heading_degrees": 180.0
}
```

### Admin Dashboard

**Provides monitoring API for system metrics.**

#### API

##### Get system overview

```http
GET /admin/overview
Authorization: Bearer {admin_token}
```

**Response (200 OK):**

```json
{
  "timestamp": "2024-12-16T10:30:00Z",
  "metrics": {
    "active_rides": 45,
    "available_drivers": 123,
    "busy_drivers": 45,
    "total_rides_today": 892,
    "total_revenue_today": 1234567.5,
    "average_wait_time_minutes": 4.2,
    "average_ride_duration_minutes": 18.5,
    "cancellation_rate": 0.05
  },
  "driver_distribution": {
    "ECONOMY": 89,
    "PREMIUM": 28,
    "XL": 6
  },
  "hotspots": [
    {
      "location": "Almaty Airport",
      "active_rides": 12,
      "waiting_drivers": 34
    },
    {
      "location": "Mega Alma-Ata",
      "active_rides": 8,
      "waiting_drivers": 15
    }
  ]
}
```

##### Get active rides

```http
GET /admin/rides/active?page=1&page_size=20
Authorization: Bearer {admin_token}
```

**Response (200 OK):**

```json
{
  "rides": [
    {
      "ride_id": "550e8400-e29b-41d4-a716-446655440000",
      "ride_number": "RIDE_20241216_001",
      "status": "IN_PROGRESS",
      "passenger_id": "880e8400-e29b-41d4-a716-446655440003",
      "driver_id": "660e8400-e29b-41d4-a716-446655440001",
      "pickup_address": "Almaty Central Park",
      "destination_address": "Kok-Tobe Hill",
      "started_at": "2024-12-16T10:30:00Z",
      "estimated_completion": "2024-12-16T10:45:00Z",
      "current_driver_location": {
        "latitude": 43.23,
        "longitude": 76.87
      },
      "distance_completed_km": 2.3,
      "distance_remaining_km": 2.9
    }
  ],
  "total_count": 45,
  "page": 1,
  "page_size": 20
}
```

## Security Considerations

1. **Authentication:**

   - JWT tokens for API authentication
   - Separate tokens for passengers/drivers/admin
   - Service-to-service authentication tokens

2. **Authorization:**

   - Role-based access control (RBAC)
   - Resource-level permissions
   - Validate driver can only update their own location

3. **Data Protection:**

   - Encrypt sensitive data at rest
   - Use TLS for all communications
   - Sanitize logs (no passwords, tokens, phone numbers)

4. **Input Validation:**

   - Validate coordinate ranges (-90 to 90 lat, -180 to 180 lng)
   - Sanitize text inputs for SQL injection

5. **WebSocket Security:**
   - Enforce authentication timeout (5 seconds)

## Support

If you get stuck, test your system components individually before integrating them. Use the RabbitMQ management interface (http://localhost:15672) to monitor queues and message flow.

Start with a minimal implementation of each service, then gradually add complexity. Test the message flow between services using simple console applications before adding WebSocket and real-time features.

Monitor your logs for correlation IDs to trace requests across services. Use the admin dashboard to verify system state and identify bottlenecks.

If services fail to communicate, verify:

- RabbitMQ exchange and queue bindings match the architecture table
- WebSocket authentication is implemented correctly
- Message serialization formats are consistent
- Database migrations have run successfully
- Service authentication tokens are configured

## Guidelines from Author

### Core Approach

Start by mapping your data flow. Trace how a ride request moves through your system before writing code. This mental model drives all implementation decisions.

### Key Priorities

- **Data Consistency:** Use database transactions and message acknowledgments properly
- **Failure Handling:** Design for graceful degradation with circuit breakers and retries
- **Real-time Performance:** Optimize hot paths and use appropriate data structures
- **User Experience:** Every technical decision impacts real users waiting for rides

### Implementation Strategy

1. **Phase 1:** Build core ride creation and database schema
2. **Phase 2:** Implement message queue infrastructure
3. **Phase 3:** Add driver matching algorithm
4. **Phase 4:** Integrate WebSocket real-time updates
5. **Phase 5:** Add monitoring and resilience patterns

### Success Mindset

Think like a system architect. Balance consistency with availability. Design for resilience - systems will fail, but they should recover quickly and gracefully. Always consider the business impact of technical decisions.

## Author

This project has been created by:

Sabrina Bakirova

Contacts:

- Email: [bakirova200024@dgmail.com](mailto:bakirova200024@dgmail.com)
- [GitHub](https://github.com/saboopher/)
- [Discord](https://discordapp.com/users/1098895545524822036/)
- [LinkedIn](https://kz.linkedin.com/in/sabrina-bakirova-651b821b1)
