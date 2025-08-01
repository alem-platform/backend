# ride-hail 🚗

## Learning Objectives

- Advanced Message Queue Patterns
- Real-Time Communication (WebSockets)
- Geospatial Data Processing
- Complex Microservices Orchestration
- High-Concurrency Programming
- Distributed State Management

## Abstract

In this project, you will build a real-time distributed ride-hailing platform. Using Go, you will create multiple microservices that communicate through RabbitMQ message queues and WebSocket connections to handle ride requests, driver matching, live location tracking, and ride coordination. This system simulates the complex backend infrastructure of modern transportation platforms like Uber or Lyft, teaching you how to design systems that handle real-time data streams, coordinate moving entities, and maintain consistency across distributed components.

This project represents the culmination of distributed systems knowledge: you'll implement sophisticated message routing patterns, handle geospatial calculations, manage real-time bidirectional communication, and orchestrate complex business workflows across multiple independent services.

## Context

When you tap "Request Ride" in a ride-hailing app, what seems like a simple button press triggers an intricate choreography of distributed systems. Within seconds, your request must be broadcast to nearby drivers, optimal matches calculated based on distance and traffic, real-time locations tracked and updated, routes calculated and optimized, and all participants kept informed through live updates.

The challenge isn't just handling one ride request—it's handling thousands simultaneously. Drivers are constantly moving, changing the optimal matching calculations every second. Riders cancel requests, drivers go offline, traffic conditions change routes, and payment systems must process transactions reliably. All of this must happen in real-time, with sub-second response times, while maintaining data consistency across multiple services.

This is where advanced message patterns and real-time communication become essential. Unlike traditional request-response systems, ride-hailing platforms operate on event streams: location updates, status changes, and matching events flow continuously through the system. Services must react to these events in real-time, making decisions based on constantly changing data while ensuring no rides are lost and no money goes missing.

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will automatically receive a score of `0`.
- Your program MUST compile successfully.
- Your program MUST NOT crash unexpectedly (any panics: `nil-pointer dereference`, `index out of range`, etc.). If this happens, you will receive `0` points during defense.
- Only built-in Go packages, `pgx/v5` PostgreSQL driver, the official AMQP client (`github.com/rabbitmq/amqp091-go`), and Gorilla WebSocket (`github.com/gorilla/websocket`) are allowed. If other packages are used, you will receive a score of `0`.
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
  max_connections: 1000
  read_buffer_size: 1024
  write_buffer_size: 1024

# Service Ports
services:
  ride_service: ${RIDE_SERVICE_PORT:-3000}
  driver_service: ${DRIVER_SERVICE_PORT:-3001}
  location_service: ${LOCATION_SERVICE_PORT:-3002}
  admin_service: ${ADMIN_SERVICE_PORT:-3004}
```

## System Architecture

Four microservices communicate through PostgreSQL and RabbitMQ to handle the complete ride-hailing workflow:

```
                            +--------------------------------------------------+
                            |                PostgreSQL Database               |
                            |           (Rides, Drivers, Locations,            |
                            |               Events, Sessions)                  |
                            +--------------------------------------------------+
                                        ^                          ^
                               (2,7,11) |                          |
                                        v                          v
    +-------------+        +-------------------+              +-----------+
    |    Rider    |   (1)  |    Ride Service   |              |   Admin   |
    | (WebSocket) |------->|   (Orchestrator)  |              | Dashboard |
    |             |<------ |                   |              |           |
    +-------------+   (8)  +-------------------+              +-----------+     
                                      ^                             ^
                                   (3)|                             |
                                  (14)|                             | (15)
                                      v                             v
                         +-----------------------------------------------------+
                         |                                                     |
                         |              RabbitMQ Message Broker                |
                         |                                                     |
                         |  Exchange: ride_topic                               |
                         |    - Queue: ride_requests      (3) New rides        |
                         |    - Queue: ride_status        (14) Status updates  |
                         |                                                     |
                         |  Exchange: driver_topic                             |
                         |    - Queue: driver_matching    (4) Match requests   |
                         |    - Queue: driver_responses   (5) Driver accepts   |
                         |                                                     |
                         |  Exchange: location_fanout                          |
                         |    - Queue: location_updates   (10) Location data   |
                         |                                                     |
                         +-----------------------------------------------------+
                                   ^                              ^
                               (4) |                          (6) |
                                   v                              v
  +-------------+             +----------+                   +----------+
  |   Driver    |    (9)      |  Driver  |       (4,5)       | Location |
  | (WebSocket) |<----------> |  Service |<----------------->| Service  |
  |             |             |          |                   |          |
  +-------------+             +----------+                   +----------+
```

## Request Flow - Step by Step

| Step | Actor | Action | Data | Description |
|------|-------|--------|------|-------------|
| | **PHASE 1: RIDE REQUEST INITIATION** | | | |
| **1** | Rider | HTTP POST to `/rides` endpoint | Pickup location, destination, ride type, rider preferences | User fills out ride request form with pickup address, destination, and ride type. Request includes GPS coordinates, payment method, and special requirements. |
| **2** | Ride Service | PostgreSQL INSERT (Database transaction) | Complete ride record with status 'requested' and calculated fare estimate | Service validates request data, generates unique ride ID (RIDE_YYYYMMDD_HHMMSS_XXX), calculates initial fare estimate using Fare Service, and stores ride in database with status 'requested'. |
| **3** | Ride Service | Publish message to `ride_requests` queue via `ride_topic` exchange | Ride details, pickup/destination coordinates, estimated fare, ride preferences | After successful database storage, service publishes ride request message to RabbitMQ with routing key `ride.request.{ride_type}`, making it available for driver matching process. |
| | **PHASE 2: DRIVER MATCHING PROCESS** | | | |
| **4** | Driver Service | Consume from `driver_matching` queue & execute matching | Geospatial matching algorithm execution with Location Service | Driver service receives ride request, calls Location Service to query available drivers within configurable radius using PostGIS functions, filters by vehicle type, driver rating, and status, then ranks candidates by distance and rating. |
| **5** | Driver Service | Send ride offers via WebSocket to selected drivers | Multiple ride offers with timeout mechanism | Service sends ride offer notifications to top 3-5 drivers through WebSocket connections, including pickup location, estimated fare, distance to pickup, and 30-second acceptance timeout. |
| **6** | Driver | Accept ride offer via WebSocket | Driver acceptance with current location and ETA(Estimated Time of Arrival) | Driver receives offer notification, reviews ride details (distance, fare, pickup location), and sends acceptance message. System implements first-come-first-served matching with automatic offer expiration and fallback to next available driver. |
| | **PHASE 3: RIDE CONFIRMATION AND SETUP** | | | |
| **7** | Ride Service | PostgreSQL UPDATE (Database transaction) | Ride status changed to 'matched', driver assigned, timestamps | Service updates ride record with matched driver using database transaction, changes status to 'matched', records matching timestamp, updates driver status to 'busy', and publishes match event to ride_status queue. |
| **8** | Ride Service | WebSocket message to rider | Driver details, vehicle info, current location, ETA | Rider receives confirmation with driver profile (photo, name, rating), vehicle details (make, model, color, license plate), real-time location, and calculated ETA. UI updates to show "Driver En Route" state. |
| **9** | Driver Service | WebSocket message to driver | Ride details, pickup location, rider info, navigation data | Driver receives ride confirmation with pickup address, rider contact info, special instructions, and turn-by-turn navigation data. Driver app switches to "En Route to Pickup" mode. |
| | **PHASE 4: REAL-TIME TRACKING AND UPDATES** | | | |
| **10** | Driver App | Send location updates to `location_updates` queue | GPS coordinates, accuracy, speed, heading, timestamp, ride_id | Driver app automatically broadcasts location updates every 3-5 seconds while active, including GPS coordinates, speed, heading, and accuracy. Location Service processes these for ETA calculations and geofencing events. |
| **11** | Location Service | Process location updates and broadcast | Processed location data with ETA calculations | Location Service receives raw GPS data, calculates ETAs, detects geofencing events (arrival at pickup, completion), and broadcasts processed updates to all interested services via fanout exchange. |
| | **PHASE 5: RIDE EXECUTION AND COMPLETION** | | | |
| **12** | Driver | Status transitions via `ride_status` queue | Ride status changes (arrived, picked_up, in_progress, completed) | Driver manually updates status (arrived at pickup, passenger picked up).|
| **13** | Ride Service | Send completion notifications | Ride summary, receipt, rating prompts | Service sends ride completion notifications to both rider and driver with trip summary, final fare breakdown, receipt, and prompts for mutual rating/feedback. Updates driver status back to 'available' for new ride requests. |

## Services

### Ride Service

**Core orchestrator managing ride lifecycle.**

#### Migrations

```sql
begin;

-- Ride status enumeration
create type ride_status as enum (
    'REQUESTED',   -- Ride has been requested by customer
    'MATCHED',     -- Driver has been matched to the ride
    'EN_ROUTE',    -- Driver is on the way to pickup location
    'ARRIVED',     -- Driver has arrived at pickup location
    'IN_PROGRESS', -- Ride is currently in progress
    'COMPLETED',   -- Ride has been successfully completed
    'CANCELLED'    -- Ride has been cancelled
);

-- Ride type enumeration
create type ride_type as enum (
    'ECONOMY',     -- Standard economy ride
    'PREMIUM',     -- Premium comfort ride
    'XL'           -- Extra large vehicle for groups
);

-- Main rides table
create table rides (
    id uuid primary key default gen_random_uuid(),
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    ride_number varchar(50) unique not null,
    rider_id varchar(50) not null,
    driver_id varchar(50) references drivers(id),
    pickup_latitude decimal(10,8) not null check (pickup_latitude between -90 and 90),
    pickup_longitude decimal(11,8) not null check (pickup_longitude between -180 and 180),
    pickup_address text not null,
    destination_latitude decimal(10,8) check (destination_latitude between -90 and 90),
    destination_longitude decimal(11,8) check (destination_longitude between -180 and 180),
    destination_address text,
    ride_type ride_type not null default 'ECONOMY',
    status ride_status not null default 'REQUESTED',
    fare_amount decimal(10,2) check (fare_amount >= 0),
    distance_km decimal(8,2) check (distance_km >= 0),
    duration_minutes integer check (duration_minutes >= 0),
    priority integer default 1 check (priority between 1 and 10),
    requested_at timestamptz default now(),
    matched_at timestamptz,
    started_at timestamptz,
    completed_at timestamptz,
    cancelled_at timestamptz,
    cancellation_reason text
);

-- Event type enumeration for audit trail
create type ride_event_type as enum (
    'RIDE_REQUESTED',    -- Initial ride request
    'DRIVER_MATCHED',    -- Driver assigned to ride
    'DRIVER_ARRIVED',    -- Driver arrived at pickup
    'RIDE_STARTED',      -- Ride began
    'RIDE_COMPLETED',    -- Ride finished
    'RIDE_CANCELLED',    -- Ride was cancelled
    'STATUS_CHANGED',    -- General status change
    'LOCATION_UPDATED',  -- Location update during ride
    'FARE_ADJUSTED'      -- Fare was adjusted
);

-- Event sourcing table for complete ride audit trail
create table ride_events (
    id uuid primary key default gen_random_uuid(),
    created_at timestamptz not null default now(),
    ride_id uuid references rides(id) not null,
    event_type ride_event_type not null,
    event_data jsonb not null
);

/*
Example event_data field:
{
  "old_status": "requested",
  "new_status": "matched",
  "driver_id": "driver_67890",
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
Authorization: Bearer {rider_token}

{
  "rider_id": "rider_12345",
  "pickup_latitude": 43.238949,
  "pickup_longitude": 76.889709,
  "pickup_address": "Almaty Central Park",
  "destination_latitude": 43.222015,
  "destination_longitude": 76.851511,
  "destination_address": "Kok-Tobe Hill",
  "ride_type": "ECONOMY"
}
```

**Response:**

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

##### Incoming Messages

**Driver Match Response** ← `driver_topic` exchange ← `driver.response.{ride_id}`

```json
{
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "driver_id": "driver_67890",
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

#### WebSocket Events

```
ws://{host}/ws/riders/{rider_id}
```

**Authentication:**
```json
{
  "type": "auth",
  "token": "Bearer {rider_token}"
}
```

**To Rider:**

```json
{
  "type": "ride_status_update",
  "ride_id": "550e8400-e29b-41d4-a716-446655440000",
  "ride_number": "RIDE_20241216_001",
  "status": "MATCHED",
  "driver_info": {
    "driver_id": "driver_67890",
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

### Driver Service

**Manages driver registration, availability, and matching algorithm.**

#### Migrations

```sql
begin;

-- Driver status enumeration
create type driver_status as enum (
    'OFFLINE',      -- Driver is not accepting rides
    'AVAILABLE',    -- Driver is available to accept rides
    'BUSY',         -- Driver is currently occupied
    'EN_ROUTE'      -- Driver is on the way to pickup
);

-- Vehicle type enumeration (must match ride types)
create type vehicle_type as enum (
    'ECONOMY',     -- Standard vehicle
    'PREMIUM',     -- Higher comfort/luxury
    'XL'           -- Large vehicle for groups
);

-- Main drivers table
create table drivers (
    id varchar(50) primary key,
    created_at timestamptz not null default now(),
    updated_at timestamptz not null default now(),
    name varchar(100) not null,
    phone varchar(20) not null,
    email varchar(100) unique not null,
    license_number varchar(50) unique not null,
    vehicle_type vehicle_type not null,
    vehicle_make varchar(50) not null,
    vehicle_model varchar(50) not null,
    vehicle_color varchar(30) not null,
    vehicle_plate varchar(20) unique not null,
    vehicle_year integer not null check (vehicle_year between 2000 and extract(year from now())),
    rating decimal(3,2) default 5.0 check (rating between 1.0 and 5.0),
    total_rides integer default 0 check (total_rides >= 0),
    total_earnings decimal(10,2) default 0 check (total_earnings >= 0),
    status driver_status not null default 'OFFLINE',
    current_latitude decimal(10,8) check (current_latitude between -90 and 90),
    current_longitude decimal(11,8) check (current_longitude between -180 and 180),
    last_location_update timestamptz,
    is_verified boolean default false,
    documents_verified boolean default false
);

-- Driver sessions for tracking online/offline times
create table driver_sessions (
    id uuid primary key default gen_random_uuid(),
    driver_id varchar(50) references drivers(id) not null,
    started_at timestamptz not null default now(),
    ended_at timestamptz,
    total_rides integer default 0,
    total_earnings decimal(10,2) default 0
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

**Response:**
```json
{
  "status": "AVAILABLE",
  "session_id": "660e8400-e29b-41d4-a716-446655440001",
  "message": "You are now online and ready to accept rides"
}
```

#### Logic

1. **Consume ride requests** from `driver_matching` queue
2. **Find nearby drivers** using PostGIS and Haversine formula:
   ```sql
   SELECT id, name, rating, 
          ST_Distance(
            point(current_longitude, current_latitude)::geography,
            point($1, $2)::geography
          ) / 1000 as distance_km
   FROM drivers
   WHERE status = 'AVAILABLE'
     AND vehicle_type = $3
     AND ST_DWithin(
           point(current_longitude, current_latitude)::geography,
           point($1, $2)::geography,
           5000  -- 5km radius
         )
   ORDER BY distance_km, rating DESC
   LIMIT 10;
   ```
3. **Score and rank drivers** based on:
   - Distance to pickup (40% weight)
   - Driver rating (30% weight)
   - Completion rate (20% weight)
   - Response time history (10% weight)
4. **Send ride offers** via WebSocket to top 3-5 drivers

### Message Patterns

#### Incoming Messages

**Driver Match Request** ← `ride_topic` exchange ← `ride.request.{ride_type}`

#### Outgoing Messages

**Driver Match Response** → `driver_topic` exchange → `driver.response.{ride_id}`

### WebSocket Events

```
ws://{host}/ws/drivers/{driver_id}
```

**To Driver:**

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

**From Driver:**

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

### Location Service

**Handles real-time location tracking and geospatial operations.**

#### Migrations

```sql
begin;

-- Location history for analytics and dispute resolution
create table location_history (
    id uuid primary key default gen_random_uuid(),
    entity_id varchar(50) not null, -- driver_id or ride_id
    entity_type varchar(20) not null check (entity_type in ('driver', 'rider')),
    latitude decimal(10,8) not null check (latitude between -90 and 90),
    longitude decimal(11,8) not null check (longitude between -180 and 180),
    accuracy_meters decimal(6,2),
    speed_kmh decimal(5,2),
    heading_degrees decimal(5,2) check (heading_degrees between 0 and 360),
    recorded_at timestamptz not null default now(),
    ride_id uuid references rides(id)
);

-- Geofences for important locations
create table geofences (
    id uuid primary key default gen_random_uuid(),
    name varchar(100) not null,
    type varchar(50) not null, -- 'airport', 'station', 'mall', etc
    center_latitude decimal(10,8) not null,
    center_longitude decimal(11,8) not null,
    radius_meters integer not null,
    surge_multiplier decimal(3,2) default 1.0
);

commit;
```

#### Message Patterns

**Incoming:** Location updates from drivers

**Outgoing:** Processed locations with ETA (Estimated Time of Arrival) calculations via fanout exchange

### Admin Dashboard

**Provides monitoring API for system metrics.**

#### API

##### Get system overview

```http
GET /admin/overview
Authorization: Bearer {admin_token}
```

**Response:**

```json
{
  "timestamp": "2024-12-16T10:30:00Z",
  "metrics": {
    "active_rides": 45,
    "available_drivers": 123,
    "busy_drivers": 45,
    "total_rides_today": 892,
    "total_revenue_today": 1234567.50,
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
    }
  ]
}
```

##### Get active rides

```http
GET /admin/rides/active
Authorization: Bearer {admin_token}
```

**Response:**

```json
{
  "rides": [
    {
      "ride_id": "550e8400-e29b-41d4-a716-446655440000",
      "ride_number": "RIDE_20241216_001",
      "status": "IN_PROGRESS",
      "rider_id": "rider_12345",
      "driver_id": "driver_67890",
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

## Message Queue Architecture

### Exchange Configuration

#### Topic Exchange: `ride_topic`
```yaml
name: ride_topic
bindings:
  - queue: ride_requests
    routing_key: ride.request.*
  - queue: ride_status
    routing_key: ride.status.*
  - queue: ride_events
    routing_key: ride.event.*
```

#### Topic Exchange: `driver_topic`
```yaml
name: driver_topic
bindings:
  - queue: driver_matching
    routing_key: driver.match.*
  - queue: driver_responses
    routing_key: driver.response.*
  - queue: driver_status
    routing_key: driver.status.*
```

#### Fanout Exchange: `location_fanout`
```yaml
name: location_fanout
bindings:
  - queue: location_updates_ride
  - queue: location_updates_admin
  - queue: location_updates_analytics
```

#### Direct Exchange: `events_direct`
```yaml
name: events_direct
bindings:
  - queue: audit_events
    routing_key: audit
  - queue: analytics_events
    routing_key: analytics
```

## Security Considerations

1. **Authentication:**
   - JWT tokens for API authentication
   - Separate tokens for riders/drivers/admin

2. **Authorization:**
   - Role-based access control (RBAC)
   - Resource-level permissions

3. **Data Protection:**
   - Encrypt sensitive data at rest
   - Use TLS for all communications
   - Sanitize logs (no passwords, tokens)

4. **Input Validation:**
   - Validate all coordinates
   - Sanitize text inputs

## Support

If you get stuck, test your system components individually before integrating them. Use the RabbitMQ management interface (http://localhost:15672) to monitor queues and message flow.

Start with a minimal implementation of each service, then gradually add complexity. Test the message flow between services using simple console applications before adding WebSocket and real-time features.

Monitor your logs for correlation IDs to trace requests across services. Use the admin dashboard to verify system state and identify bottlenecks.

If services fail to communicate, verify:
- RabbitMQ exchange and queue bindings
- Database foreign key constraints
- WebSocket authentication headers
- Message serialization formats
- Coordinate validation ranges

Remember to handle edge cases:
- Driver cancellations after acceptance
- Rider cancellations with fees
- Invalid or spoofed location data
- Concurrent ride requests from same rider
- Network partitions between services
- Database connection pool exhaustion

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

### Common Pitfalls to Avoid

- Don't use floating point for money calculations
- Remember timezone handling for international deployments
- Account for GPS accuracy in location matching
- Handle WebSocket reconnection gracefully
- Implement proper connection pooling

### Success Mindset

Think like a system architect. Balance consistency with availability. Design for resilience - systems will fail, but they should recover quickly and gracefully. Always consider the business impact of technical decisions.

## Author

This project has been created by:

Sabrina Bakirova

Contacts:

- Email: [bakirova200024@dgmail.com](mailto:bakirova200024@dgmail.com)
- [GitHub](https://github.com/saboopher/)