# Design Document: AI-Based Emergency Vehicle Green Corridor System

## Overview

The AI-Based Emergency Vehicle Green Corridor System is a distributed, cloud-edge hybrid platform that coordinates emergency vehicle routing with intelligent traffic signal control. The system combines real-time GPS tracking, AI-powered prediction, computer vision-based traffic analysis, and LLM-driven explanation capabilities to create dynamic green corridors that minimize emergency response times.

### Core Design Principles

1. **Real-time Responsiveness**: All critical path operations complete within 3 seconds
2. **Fault Tolerance**: System degrades gracefully with component failures, never leaving emergency vehicles unsupported
3. **Edge-First Architecture**: Critical signal control decisions can be made locally without cloud connectivity
4. **Explainability**: All AI decisions are traceable and explainable through LLM-powered reasoning
5. **Security by Design**: Multi-layered authentication, encryption, and tamper detection throughout
6. **Scalability**: Horizontally scalable architecture supporting city-wide deployment

### Technology Stack

- **Backend**: Python with FastAPI for REST APIs, asyncio for concurrent processing
- **Database**: PostgreSQL for transactional data, TimescaleDB for time-series GPS streams, Redis for caching
- **AI/ML**: PyTorch for prediction models, OpenCV for computer vision, Hugging Face Transformers for LLM
- **Message Queue**: Apache Kafka for event streaming and inter-component communication
- **Edge Computing**: Lightweight Python runtime on Raspberry Pi or similar edge devices
- **Mobile App**: React Native for cross-platform iOS/Android support
- **Dashboard**: React with WebSocket for real-time updates
- **Infrastructure**: Kubernetes for orchestration, Prometheus/Grafana for monitoring

## Architecture

### High-Level Architecture


The system follows a three-tier architecture:

**Tier 1: Mobile and Edge Devices**
- Emergency vehicle mobile apps (GPS streaming, route input, driver interface)
- Edge controllers at traffic junctions (local signal control, offline operation capability)
- Traffic cameras with embedded AI processing (vehicle counting, density estimation)

**Tier 2: Cloud Backend Services**
- API Gateway (authentication, rate limiting, request routing)
- Incident Management Service (emergency activation, lifecycle management)
- AI Prediction Engine (route forecasting, traffic analysis, severity classification)
- Signal Coordination Service (timing calculation, multi-vehicle priority resolution)
- LLM Reasoning Engine with RAG (explanation generation, incident report creation)
- Analytics and Audit Service (metrics calculation, log management, compliance)

**Tier 3: External Integrations**
- Hospital notification systems (pre-alerts, arrival updates)
- Traffic management system APIs (bidirectional data exchange)
- Authority dashboards (real-time monitoring, manual override)
- Citizen alert platforms (optional push notifications)

### Component Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Cloud Backend Layer                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │   API        │  │   Incident   │  │   Signal     │              │
│  │   Gateway    │──│   Manager    │──│   Coordinator│              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│         │                  │                  │                      │
│         │                  │                  │                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │   AI         │  │   Camera AI  │  │   LLM + RAG  │              │
│  │   Prediction │  │   Module     │  │   Engine     │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│         │                  │                  │                      │
│         └──────────────────┴──────────────────┘                      │
│                            │                                         │
│                   ┌────────┴────────┐                                │
│                   │  Kafka Event    │                                │
│                   │  Streaming      │                                │
│                   └────────┬────────┘                                │
│                            │                                         │
│                   ┌────────┴────────┐                                │
│                   │  PostgreSQL +   │                                │
│                   │  TimescaleDB    │                                │
│                   └─────────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────┴────────┐  ┌───────┴────────┐  ┌──────┴───────┐
│  Mobile Apps   │  │  Edge          │  │  Authority   │
│  (Emergency    │  │  Controllers   │  │  Dashboard   │
│   Vehicles)    │  │  (Junctions)   │  │  (Web UI)    │
└────────────────┘  └────────────────┘  └──────────────┘
```

## Components and Interfaces

### 1. Mobile Application (Emergency Vehicle)

**Purpose**: Provides emergency vehicle drivers with activation interface, route guidance, and real-time corridor status.

**Key Functions**:
- Emergency mode activation with severity selection
- Route declaration (destination input, waypoint selection)
- GPS streaming at 1 Hz minimum frequency
- Lane recommendation display
- Alternate route suggestions with accept/reject interface
- Real-time ETA updates
- Offline mode with cached predictions

**Technology**: React Native, GPS libraries, WebSocket for real-time communication

**Interfaces**:
- REST API: POST /api/v1/incidents/activate
- REST API: POST /api/v1/incidents/{id}/route
- WebSocket: /ws/incidents/{id}/gps-stream
- WebSocket: /ws/incidents/{id}/guidance (receives lane recommendations)

### 2. API Gateway

**Purpose**: Single entry point for all external requests, handles authentication, rate limiting, and request routing.

**Key Functions**:
- JWT-based authentication with 15-minute token expiration
- Multi-factor authentication for dashboard users
- Rate limiting (100 requests/minute per vehicle, 1000 requests/minute per dashboard user)
- Request validation and sanitization
- TLS 1.3 termination
- API versioning support

**Technology**: Python FastAPI with middleware for auth and rate limiting

**Interfaces**:
- Exposes all REST APIs to external clients
- Routes requests to appropriate backend services
- Publishes authentication events to Kafka

### 3. Incident Management Service

**Purpose**: Manages the complete lifecycle of emergency incidents from activation to completion.

**Key Functions**:
- Emergency mode activation and validation
- Incident state management (Active, Paused, Completed, Cancelled)
- Vehicle authentication (credential verification, device identity validation)
- Route declaration processing and validation
- GPS stream ingestion and storage
- Automatic incident completion detection
- Fake emergency flagging and alerting

**Technology**: Python with asyncio for concurrent incident handling

**Data Models**:
```python
class Incident:
    id: UUID
    vehicle_id: str
    driver_id: str
    severity: Enum['CRITICAL', 'HIGH', 'MEDIUM', 'LOW']
    status: Enum['ACTIVE', 'PAUSED', 'COMPLETED', 'CANCELLED']
    origin: GeoPoint
    destination: GeoPoint
    waypoints: List[GeoPoint]
    activated_at: datetime
    completed_at: Optional[datetime]
    priority_score: float
    estimated_arrival: datetime
```

**Interfaces**:
- REST API: POST /api/v1/incidents/activate
- REST API: PUT /api/v1/incidents/{id}/status
- REST API: GET /api/v1/incidents/{id}
- REST API: GET /api/v1/incidents/active
- Kafka Producer: incident.activated, incident.completed, incident.flagged
- Kafka Consumer: gps.stream, signal.acknowledgment

### 4. AI Prediction Engine

**Purpose**: Forecasts emergency vehicle route progression and predicts next junction arrivals.

**Key Functions**:
- Next junction prediction using LSTM neural network
- Arrival time estimation based on speed, distance, and traffic
- Route deviation detection and recalculation triggering
- Prediction confidence scoring
- Model retraining pipeline with historical data
- A/B testing framework for model improvements

**ML Models**:

1. **Route Prediction LSTM**: Predicts next junction based on current GPS position, velocity vector, historical routes
   - Input: [lat, lon, speed, heading, time_of_day, day_of_week]
   - Output: [junction_id, confidence_score, estimated_arrival_seconds]
   - Training: Historical GPS traces from completed incidents

2. **Severity Classifier**: Classifies emergency severity from driver input and contextual signals
   - Input: [driver_severity_input, time_of_day, destination_type, historical_incident_data]
   - Output: [CRITICAL, HIGH, MEDIUM, LOW] with confidence
   - Training: Historical incidents labeled by outcome urgency

**Technology**: PyTorch for model training and inference, MLflow for experiment tracking

**Interfaces**:
- Kafka Consumer: gps.stream
- Kafka Producer: prediction.next_junction, prediction.route_deviation
- REST API: GET /api/v1/predictions/{incident_id}/next-junction
- gRPC: Internal high-performance prediction service

### 5. Camera AI Module

**Purpose**: Analyzes traffic camera feeds to estimate vehicle density and congestion levels.

**Key Functions**:
- Real-time vehicle detection and counting using YOLO object detection
- Lane-level traffic density estimation
- Congestion classification (Free-flow, Moderate, Congested, Gridlock)
- Camera health monitoring and quality assessment
- Historical traffic pattern learning
- Multi-camera fusion for junction-level density

**ML Models**:

1. **Vehicle Detection Model**: YOLOv8 fine-tuned on traffic camera footage
   - Input: Video frame (1920x1080 or lower resolution)
   - Output: Bounding boxes with vehicle class and confidence
   - Inference: 10 FPS on edge GPU

2. **Density Estimator**: Converts vehicle counts to density percentage
   - Input: Vehicle count per lane, lane capacity, historical baseline
   - Output: Density percentage (0-100%), congestion level
   - Logic: density = (current_count / lane_capacity) * 100

**Technology**: OpenCV for video processing, PyTorch with YOLO for detection, edge deployment on NVIDIA Jetson

**Interfaces**:
- RTSP/MJPEG: Camera feed ingestion
- Kafka Producer: traffic.density, camera.health
- REST API: GET /api/v1/traffic/density/{junction_id}

### 6. Signal Coordination Service

**Purpose**: Calculates optimal signal timing and coordinates multiple emergency vehicles.

**Key Functions**:
- Signal timing calculation based on predicted arrival and traffic density
- Multi-vehicle priority resolution using priority scoring algorithm
- Signal controller command generation and acknowledgment tracking
- Conflict detection when multiple vehicles approach same junction
- Normal timing restoration after vehicle passage
- Fallback to advisory mode when controllers are offline

**Algorithms**:

1. **Priority Scoring Algorithm**:
```
priority_score = (severity_weight * severity_value) + 
                 (urgency_weight * (1 / time_to_destination)) + 
                 (vehicle_type_weight * vehicle_type_value)

where:
  severity_weight = 0.5
  urgency_weight = 0.3
  vehicle_type_weight = 0.2
  severity_value: CRITICAL=4, HIGH=3, MEDIUM=2, LOW=1
  vehicle_type_value: AMBULANCE=1.2, FIRE_TRUCK=1.0
```

2. **Signal Timing Calculation**:
```
green_duration = base_green_time + density_adjustment + clearance_time

where:
  base_green_time = 20 seconds (minimum safe crossing time)
  density_adjustment = (traffic_density / 100) * 15 seconds
  clearance_time = max(7 seconds, pedestrian_crossing_time)
  
red_duration_other_directions = green_duration + 3 seconds (yellow transition)
```

3. **Multi-Vehicle Conflict Resolution**:
```
if two vehicles approach same junction:
  if priority_score_diff > 0.5:
    grant to higher priority vehicle
    delay lower priority vehicle by (green_duration + 10 seconds)
  else:
    grant to vehicle with shorter distance to junction
    recalculate alternate route for other vehicle
```

**Technology**: Python with NumPy for calculations, Redis for distributed locking

**Interfaces**:
- Kafka Consumer: prediction.next_junction, traffic.density
- Kafka Producer: signal.command, signal.conflict_detected
- REST API: POST /api/v1/signals/command
- REST API: GET /api/v1/signals/{junction_id}/status

### 7. Edge Controller (Junction)

**Purpose**: Local signal control at traffic junctions with offline operation capability.

**Key Functions**:
- Receives and executes signal timing commands from cloud
- Maintains local cache of recent predictions for offline operation
- Communicates with physical signal controller hardware via NTCIP protocol
- Monitors signal controller health and acknowledgment
- Falls back to historical patterns during network outage
- Synchronizes audit logs when connectivity restored

**Technology**: Python on Raspberry Pi 4 or industrial edge computer, local SQLite for caching

**Interfaces**:
- Kafka Consumer: signal.command (when online)
- Kafka Producer: signal.acknowledgment, signal.health (when online)
- NTCIP 1202: Communication with physical signal controller
- Local REST API: GET /health, GET /status

### 8. LLM Reasoning Engine with RAG

**Purpose**: Generates natural language explanations of system decisions and creates post-incident reports.

**Key Functions**:
- Retrieval-Augmented Generation for context-aware explanations
- Query understanding and intent classification
- Document retrieval from incident data, audit logs, and policy documents
- Explanation generation with citations and timestamps
- Post-incident report generation with structured sections
- Factual accuracy validation against source data

**RAG Architecture**:

1. **Data Sources**:
   - Incident database (PostgreSQL): All incident records with routes, timings, outcomes
   - Audit log database (TimescaleDB): All system actions with timestamps and parameters
   - Traffic pattern database: Historical density data and signal timing history
   - Policy documents: System configuration, priority rules, timing formulas
   - Model metadata: AI model versions, accuracy metrics, training data ranges

2. **Retrieval Layer**:
   - Vector database (Pinecone or Weaviate) for semantic search
   - Document embeddings using sentence-transformers
   - Hybrid search combining semantic similarity and keyword matching
   - Relevance ranking with recency weighting
   - Top-k retrieval (k=5 for explanations, k=10 for reports)

3. **LLM Layer**:
   - Base model: GPT-4 or Llama-2-70B for high-quality generation
   - Prompt engineering with role definition and output format specification
   - Few-shot examples for consistent report structure
   - Temperature=0.3 for factual accuracy
   - Max tokens=2000 for comprehensive explanations

4. **Validation Layer**:
   - Fact-checking against retrieved source documents
   - Timestamp validation (all cited times must exist in audit logs)
   - Numerical accuracy verification (recalculate formulas mentioned)
   - Hallucination detection using entailment models

**Technology**: Hugging Face Transformers, LangChain for RAG orchestration, Pinecone for vector storage

**Interfaces**:
- REST API: POST /api/v1/llm/explain (query explanation)
- REST API: POST /api/v1/llm/report/{incident_id} (generate incident report)
- REST API: GET /api/v1/llm/reports/{incident_id} (retrieve generated report)

### 9. Authority Dashboard

**Purpose**: Real-time monitoring interface for traffic police and command center operators.

**Key Functions**:
- Live map with all active emergency vehicles
- Incident list with priority scores and ETAs
- Signal controller status visualization
- Manual override controls for incidents
- Fake emergency flagging and review
- Historical incident playback
- Analytics and metrics display
- Drill mode activation and management

**Technology**: React with Mapbox GL for mapping, WebSocket for real-time updates, Chart.js for analytics

**Interfaces**:
- WebSocket: /ws/dashboard/incidents (real-time incident updates)
- WebSocket: /ws/dashboard/signals (signal status updates)
- REST API: GET /api/v1/dashboard/incidents/active
- REST API: POST /api/v1/dashboard/incidents/{id}/override
- REST API: POST /api/v1/dashboard/incidents/{id}/flag

### 10. Hospital Interface

**Purpose**: Notification system for hospital control rooms to prepare for incoming emergencies.

**Key Functions**:
- Pre-alert delivery with incident details
- ETA updates as route progresses
- Arrival confirmation notifications
- Incident completion notifications
- Integration with hospital resource management systems

**Technology**: REST API client, webhook delivery, SMS/email fallback

**Interfaces**:
- Webhook: POST {hospital_webhook_url}/emergency-alert
- REST API: POST /api/v1/hospitals/{id}/alerts
- SMS Gateway: Twilio for critical alerts

### 11. Analytics and Audit Service

**Purpose**: Calculates performance metrics, maintains audit logs, and ensures compliance.

**Key Functions**:
- Time saved calculation (actual vs baseline travel time)
- Corridor success rate computation
- False emergency detection rate tracking
- Audit log ingestion and storage
- Tamper-evident log signing with cryptographic hashes
- Data retention and anonymization
- Compliance reporting for city governance

**Technology**: Python with Pandas for analytics, TimescaleDB for time-series audit logs

**Interfaces**:
- Kafka Consumer: All event topics for audit logging
- REST API: GET /api/v1/analytics/metrics
- REST API: GET /api/v1/audit/logs (with filtering and pagination)
- REST API: POST /api/v1/analytics/reports/generate

## Data Models

### Database Schema

**Vehicles Table**:
```sql
CREATE TABLE vehicles (
    id VARCHAR(50) PRIMARY KEY,
    vehicle_type VARCHAR(20) NOT NULL CHECK (vehicle_type IN ('AMBULANCE', 'FIRE_TRUCK')),
    registration_number VARCHAR(50) UNIQUE NOT NULL,
    organization VARCHAR(100) NOT NULL,
    siren_signature BYTEA,
    device_certificate TEXT NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**Drivers Table**:
```sql
CREATE TABLE drivers (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    license_number VARCHAR(50) UNIQUE NOT NULL,
    organization VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Incidents Table**:
```sql
CREATE TABLE incidents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vehicle_id VARCHAR(50) REFERENCES vehicles(id),
    driver_id VARCHAR(50) REFERENCES drivers(id),
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('CRITICAL', 'HIGH', 'MEDIUM', 'LOW')),
    status VARCHAR(20) NOT NULL CHECK (status IN ('ACTIVE', 'PAUSED', 'COMPLETED', 'CANCELLED')),
    origin_lat DOUBLE PRECISION NOT NULL,
    origin_lon DOUBLE PRECISION NOT NULL,
    destination_lat DOUBLE PRECISION NOT NULL,
    destination_lon DOUBLE PRECISION NOT NULL,
    waypoints JSONB,
    priority_score DOUBLE PRECISION,
    estimated_arrival TIMESTAMP,
    activated_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    time_saved_seconds INTEGER,
    is_flagged BOOLEAN DEFAULT false,
    flag_reason TEXT,
    is_drill BOOLEAN DEFAULT false
);

CREATE INDEX idx_incidents_status ON incidents(status);
CREATE INDEX idx_incidents_activated ON incidents(activated_at);
```

**GPS Streams Table (TimescaleDB Hypertable)**:
```sql
CREATE TABLE gps_streams (
    incident_id UUID REFERENCES incidents(id),
    timestamp TIMESTAMP NOT NULL,
    lat DOUBLE PRECISION NOT NULL,
    lon DOUBLE PRECISION NOT NULL,
    speed DOUBLE PRECISION,
    heading DOUBLE PRECISION,
    accuracy DOUBLE PRECISION,
    PRIMARY KEY (incident_id, timestamp)
);

SELECT create_hypertable('gps_streams', 'timestamp');
```

**Junctions Table**:
```sql
CREATE TABLE junctions (
    id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(100),
    lat DOUBLE PRECISION NOT NULL,
    lon DOUBLE PRECISION NOT NULL,
    signal_controller_id VARCHAR(50),
    has_camera BOOLEAN DEFAULT false,
    camera_url TEXT,
    edge_controller_id VARCHAR(50),
    is_controllable BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Signal Commands Table**:
```sql
CREATE TABLE signal_commands (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id UUID REFERENCES incidents(id),
    junction_id VARCHAR(50) REFERENCES junctions(id),
    command_type VARCHAR(20) NOT NULL,
    green_duration_seconds INTEGER,
    red_duration_seconds INTEGER,
    sent_at TIMESTAMP DEFAULT NOW(),
    acknowledged_at TIMESTAMP,
    executed_at TIMESTAMP,
    status VARCHAR(20) CHECK (status IN ('SENT', 'ACKNOWLEDGED', 'EXECUTED', 'FAILED'))
);

CREATE INDEX idx_signal_commands_incident ON signal_commands(incident_id);
```

**Traffic Density Table (TimescaleDB Hypertable)**:
```sql
CREATE TABLE traffic_density (
    junction_id VARCHAR(50) REFERENCES junctions(id),
    timestamp TIMESTAMP NOT NULL,
    vehicle_count INTEGER NOT NULL,
    density_percentage DOUBLE PRECISION NOT NULL,
    congestion_level VARCHAR(20) CHECK (congestion_level IN ('FREE_FLOW', 'MODERATE', 'CONGESTED', 'GRIDLOCK')),
    confidence DOUBLE PRECISION,
    PRIMARY KEY (junction_id, timestamp)
);

SELECT create_hypertable('traffic_density', 'timestamp');
```

**Audit Logs Table (TimescaleDB Hypertable)**:
```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    actor_type VARCHAR(20) CHECK (actor_type IN ('DRIVER', 'OPERATOR', 'SYSTEM', 'ADMIN')),
    actor_id VARCHAR(50),
    incident_id UUID,
    resource_type VARCHAR(50),
    resource_id VARCHAR(50),
    action VARCHAR(50) NOT NULL,
    parameters JSONB,
    result VARCHAR(20) CHECK (result IN ('SUCCESS', 'FAILURE', 'PENDING')),
    signature TEXT
);

SELECT create_hypertable('audit_logs', 'timestamp');
CREATE INDEX idx_audit_logs_incident ON audit_logs(incident_id);
CREATE INDEX idx_audit_logs_event_type ON audit_logs(event_type);
```

**LLM Reports Table**:
```sql
CREATE TABLE llm_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    incident_id UUID REFERENCES incidents(id),
    report_type VARCHAR(20) CHECK (report_type IN ('POST_INCIDENT', 'EXPLANATION')),
    query TEXT,
    report_content TEXT NOT NULL,
    sources JSONB,
    generated_at TIMESTAMP DEFAULT NOW(),
    generated_by VARCHAR(50)
);
```

## Data Flow

### Emergency Activation Flow

1. Driver opens mobile app and taps "Activate Emergency Mode"
2. App sends POST /api/v1/incidents/activate with credentials, destination, severity
3. API Gateway validates JWT token and routes to Incident Management Service
4. Incident Management Service:
   - Verifies driver credentials against drivers table
   - Verifies vehicle registration against vehicles table
   - Validates device certificate
   - Creates incident record in database
   - Publishes incident.activated event to Kafka
5. Signal Coordination Service consumes incident.activated event
6. AI Prediction Engine consumes incident.activated event and begins monitoring
7. Hospital Interface consumes incident.activated event and sends pre-alert webhook
8. Authority Dashboard receives WebSocket update and displays new incident
9. Mobile app receives incident ID and begins GPS streaming

### GPS Streaming and Prediction Flow

1. Mobile app streams GPS coordinates via WebSocket /ws/incidents/{id}/gps-stream
2. Incident Management Service receives GPS data and:
   - Stores in gps_streams table
   - Publishes gps.stream event to Kafka
3. AI Prediction Engine consumes gps.stream event:
   - Runs LSTM model to predict next junction
   - Calculates arrival time estimate
   - Publishes prediction.next_junction event
4. Signal Coordination Service consumes prediction.next_junction:
   - Retrieves traffic density for predicted junction
   - Calculates priority score if multiple vehicles present
   - Computes optimal signal timing
   - Publishes signal.command event
5. Edge Controller consumes signal.command:
   - Validates command authenticity
   - Sends timing command to physical signal controller via NTCIP
   - Publishes signal.acknowledgment event
6. Authority Dashboard receives updates via WebSocket

### Multi-Vehicle Coordination Flow

1. Two incidents (A and B) have overlapping routes approaching Junction X
2. Signal Coordination Service detects conflict:
   - Calculates priority_score_A and priority_score_B
   - Determines priority_score_A > priority_score_B
3. For Incident A:
   - Sends signal.command for green corridor at Junction X
4. For Incident B:
   - Calculates delay = green_duration_A + 10 seconds
   - Checks if alternate route exists that avoids Junction X
   - If alternate route saves time, publishes route.suggestion to Incident B
   - If no alternate route, adjusts ETA by delay amount
5. Publishes signal.conflict_detected event with resolution details
6. Authority Dashboard displays conflict resolution explanation
7. Audit log records priority decision with formula inputs

### LLM Explanation Flow

1. Command Center Operator asks: "Why did vehicle A123 get priority over B456 at Junction 5?"
2. Authority Dashboard sends POST /api/v1/llm/explain with query
3. LLM Reasoning Engine:
   - Classifies query intent as "priority_decision_explanation"
   - Retrieves from RAG system:
     * Incident records for A123 and B456
     * Signal command for Junction 5
     * Audit log entry for priority decision
     * Priority scoring formula from policy documents
   - Constructs prompt with retrieved context
   - Generates explanation with GPT-4
   - Validates facts against source documents
   - Returns explanation with citations
4. Dashboard displays explanation with clickable source references

## API Design

### Vehicle Activation API

**Endpoint**: POST /api/v1/incidents/activate

**Request**:
```json
{
  "vehicle_id": "AMB-001",
  "driver_id": "DRV-123",
  "destination": {
    "lat": 40.7589,
    "lon": -73.9851,
    "address": "Mount Sinai Hospital"
  },
  "severity": "CRITICAL",
  "waypoints": []
}
```

**Response** (200 OK):
```json
{
  "incident_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "ACTIVE",
  "priority_score": 3.8,
  "estimated_arrival": "2024-01-15T14:35:00Z",
  "websocket_url": "wss://api.greencorridor.city/ws/incidents/550e8400-e29b-41d4-a716-446655440000/gps-stream"
}
```

**Error Responses**:
- 401 Unauthorized: Invalid credentials
- 403 Forbidden: Vehicle not authorized
- 409 Conflict: Vehicle already has active incident

### GPS Stream API

**Endpoint**: WebSocket /ws/incidents/{incident_id}/gps-stream

**Client Message**:
```json
{
  "lat": 40.7589,
  "lon": -73.9851,
  "speed": 45.5,
  "heading": 270.0,
  "accuracy": 5.0,
  "timestamp": "2024-01-15T14:25:30Z"
}
```

**Server Message** (Guidance):
```json
{
  "type": "lane_recommendation",
  "message": "Move to right lane - less congested",
  "next_junction": "JCT-045",
  "eta_seconds": 120,
  "signal_status": "GREEN_PREPARED"
}
```

### Signal Control API

**Endpoint**: POST /api/v1/signals/command

**Request**:
```json
{
  "junction_id": "JCT-045",
  "incident_id": "550e8400-e29b-41d4-a716-446655440000",
  "command_type": "GREEN_CORRIDOR",
  "green_duration_seconds": 35,
  "direction": "NORTH_SOUTH",
  "execute_at": "2024-01-15T14:27:00Z"
}
```

**Response** (202 Accepted):
```json
{
  "command_id": "cmd-789",
  "status": "SENT",
  "expected_acknowledgment_by": "2024-01-15T14:26:55Z"
}
```

### Dashboard API

**Endpoint**: GET /api/v1/dashboard/incidents/active

**Response** (200 OK):
```json
{
  "incidents": [
    {
      "incident_id": "550e8400-e29b-41d4-a716-446655440000",
      "vehicle_id": "AMB-001",
      "vehicle_type": "AMBULANCE",
      "severity": "CRITICAL",
      "priority_score": 3.8,
      "current_location": {"lat": 40.7589, "lon": -73.9851},
      "destination": {"lat": 40.7589, "lon": -73.9851},
      "eta_seconds": 420,
      "status": "ACTIVE",
      "is_flagged": false
    }
  ],
  "total_count": 1
}
```

### Hospital Alert API

**Endpoint**: Webhook POST {hospital_webhook_url}/emergency-alert

**Payload**:
```json
{
  "incident_id": "550e8400-e29b-41d4-a716-446655440000",
  "alert_type": "PRE_ALERT",
  "vehicle_type": "AMBULANCE",
  "severity": "CRITICAL",
  "estimated_arrival": "2024-01-15T14:35:00Z",
  "patient_count": 1,
  "special_requirements": []
}
```

### LLM Explanation API

**Endpoint**: POST /api/v1/llm/explain

**Request**:
```json
{
  "query": "Why did vehicle AMB-001 get priority over FIRE-002 at Junction 5?",
  "context": {
    "incident_ids": ["550e8400-e29b-41d4-a716-446655440000", "660e8400-e29b-41d4-a716-446655440001"],
    "junction_id": "JCT-005"
  }
}
```

**Response** (200 OK):
```json
{
  "explanation": "Vehicle AMB-001 received priority over FIRE-002 at Junction 5 because it had a higher priority score (3.8 vs 3.2). The priority calculation considered: (1) AMB-001 had CRITICAL severity while FIRE-002 had HIGH severity, (2) AMB-001 was 2 minutes from destination while FIRE-002 was 5 minutes away, and (3) Ambulances receive a 1.2x multiplier in the priority formula. The decision was made at 14:26:45 and logged in audit entry AUD-12345.",
  "sources": [
    {"type": "audit_log", "id": "AUD-12345", "timestamp": "2024-01-15T14:26:45Z"},
    {"type": "incident", "id": "550e8400-e29b-41d4-a716-446655440000"},
    {"type": "policy", "id": "priority_formula_v2"}
  ],
  "confidence": 0.95
}
```