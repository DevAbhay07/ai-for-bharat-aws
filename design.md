# Design Document: Agentic AI Smart Parking System

## Overview

The Agentic AI Smart Parking System is a serverless, event-driven architecture built on AWS services that automates parking facility operations. The system uses computer vision (AWS Rekognition) for vehicle detection and license plate recognition, IoT sensors (AWS IoT Core) for real-time slot monitoring, AI agents (AWS Lambda) for intelligent decision-making, and DynamoDB for data persistence.

The architecture follows a microservices pattern where each AI agent is an independent Lambda function that responds to events and makes autonomous decisions. The system is designed for high availability, scalability, and real-time performance with sub-3-second response times for critical operations.

### Key Design Principles

1. **Event-Driven Architecture**: All components communicate through events (IoT messages, DynamoDB streams, EventBridge)
2. **Serverless-First**: Leverage AWS Lambda for compute to achieve automatic scaling and cost optimization
3. **Separation of Concerns**: Each AI agent has a single responsibility (allocation, violation detection, payment)
4. **Idempotency**: All operations are designed to be safely retried without side effects
5. **Observability**: Comprehensive logging, metrics, and tracing for all operations

## Architecture

### High-Level Architecture

```
┌─────────────────┐         ┌──────────────────┐         ┌─────────────────┐
│  Entry Gate     │         │   Exit Gate      │         │  IoT Sensors    │
│  - FASTag       │         │   - FASTag       │         │  (Parking Slots)│
│  - Camera       │         │   - Camera       │         │                 │
└────────┬────────┘         └────────┬─────────┘         └────────┬────────┘
         │                           │                            │
         │                           │                            │
         ▼                           ▼                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          AWS IoT Core                                    │
│  - Device Gateway                                                        │
│  - Message Broker (MQTT)                                                 │
│  - Rules Engine                                                          │
└────────┬────────────────────────────────────────────────┬────────────────┘
         │                                                 │
         │ (Entry/Exit Events)                            │ (Sensor Events)
         ▼                                                 ▼
┌─────────────────────┐                         ┌──────────────────────┐
│  Vehicle Detector   │                         │  Slot Monitor        │
│  (Lambda + Rekogn.) │                         │  (Lambda)            │
│  - Detect vehicle   │                         │  - Update slot status│
│  - Extract plate    │                         │  - Publish events    │
│  - Classify type    │                         └──────────┬───────────┘
└─────────┬───────────┘                                    │
          │                                                │
          │ (Vehicle Identified)                           │
          ▼                                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Amazon EventBridge                              │
│  - Event routing and orchestration                                       │
└────────┬────────────────────────────────────────────────┬────────────────┘
         │                                                 │
         │ (Entry Event)                                   │ (Slot Update)
         ▼                                                 ▼
┌─────────────────────┐                         ┌──────────────────────┐
│  Slot Allocator     │◄────────────────────────│  DynamoDB            │
│  (Lambda - AI Agent)│                         │  - Vehicles          │
│  - Find best slot   │                         │  - Slots             │
│  - Reserve slot     │                         │  - Transactions      │
│  - Update vehicle   │                         │  - Violations        │
└─────────┬───────────┘                         │  - Configuration     │
          │                                     └──────────┬───────────┘
          │ (Slot Allocated)                               │
          ▼                                                │
┌─────────────────────┐                                    │
│  Gate Controller    │                                    │
│  (Lambda)           │                                    │
│  - Open entry gate  │                                    │
│  - Open exit gate   │                                    │
└─────────────────────┘                                    │
                                                           │
┌─────────────────────┐                                    │
│  Violation Detector │◄───────────────────────────────────┘
│  (Lambda - AI Agent)│    (DynamoDB Streams / Scheduled)
│  - Scan sessions    │
│  - Detect overstay  │
│  - Detect unauth.   │
└─────────┬───────────┘
          │
          │ (Violation Detected)
          ▼
┌─────────────────────┐
│  Payment Processor  │
│  (Lambda)           │
│  - Calculate charges│
│  - Process payment  │
│  - Create transaction│
└─────────────────────┘
```

### Component Interaction Flow

**Entry Flow:**
1. Vehicle approaches → FASTag detected → Camera captures image
2. IoT Core receives entry event → Routes to Vehicle Detector Lambda
3. Vehicle Detector calls Rekognition → Extracts plate & classifies vehicle
4. EventBridge routes vehicle identified event → Slot Allocator Lambda
5. Slot Allocator queries DynamoDB → Finds optimal slot → Reserves slot
6. Gate Controller opens entry gate → Vehicle enters
7. IoT sensor detects occupancy → Updates slot status in DynamoDB

**Exit Flow:**
1. Vehicle approaches exit → FASTag detected
2. IoT Core receives exit event → Routes to Payment Processor Lambda
3. Payment Processor retrieves vehicle record → Calculates charges
4. Payment processed via FASTag → Transaction created in DynamoDB
5. Gate Controller opens exit gate → Vehicle exits
6. IoT sensor detects vacancy → Updates slot status in DynamoDB

**Violation Detection Flow:**
1. EventBridge scheduled rule triggers Violation Detector every 5 minutes
2. Violation Detector scans active sessions in DynamoDB
3. Identifies overstays and unauthorized parking
4. Creates violation records → Updates charges
5. Sends notifications to operators

## Components and Interfaces

### 1. Vehicle Detector (Lambda Function)

**Purpose**: Process vehicle images using AWS Rekognition to detect vehicles, extract license plates, and classify vehicle types.

**Inputs**:
- Event from IoT Core containing:
  - `gateId`: string (entry or exit gate identifier)
  - `imageS3Key`: string (S3 path to captured image)
  - `fasTagId`: string (FASTag identifier)
  - `timestamp`: ISO 8601 timestamp

**Outputs**:
- EventBridge event containing:
  - `vehicleId`: string (generated UUID)
  - `licensePlate`: string (extracted plate number)
  - `vehicleType`: enum (COMPACT, SEDAN, SUV, TRUCK)
  - `fasTagId`: string
  - `confidence`: float (0.0-1.0)
  - `imageS3Key`: string
  - `timestamp`: ISO 8601 timestamp

**Processing Logic**:
```
function detectVehicle(event):
    image = S3.getObject(event.imageS3Key)
    
    // Detect vehicle presence
    detectionResult = Rekognition.detectLabels(image, minConfidence=98)
    if not detectionResult.contains("Car", "Vehicle", "Truck"):
        return error("No vehicle detected")
    
    // Extract license plate
    textResult = Rekognition.detectText(image)
    licensePlate = extractLicensePlate(textResult)
    
    // Classify vehicle type
    vehicleType = classifyVehicleType(detectionResult)
    
    // Validate against FASTag
    if not validatePlateWithFASTag(licensePlate, event.fasTagId):
        logSuspiciousEntry(licensePlate, event.fasTagId)
    
    // Publish event
    EventBridge.putEvent({
        source: "parking.vehicle-detector",
        detailType: "VehicleIdentified",
        detail: {
            vehicleId: generateUUID(),
            licensePlate: licensePlate,
            vehicleType: vehicleType,
            fasTagId: event.fasTagId,
            confidence: detectionResult.confidence,
            imageS3Key: event.imageS3Key,
            timestamp: now()
        }
    })
```

**Error Handling**:
- Retry image capture up to 3 times if detection fails
- Request manual verification if plate extraction fails after retries
- Log all failures with image reference for debugging

### 2. Slot Monitor (Lambda Function)

**Purpose**: Process IoT sensor events to maintain real-time parking slot status.

**Inputs**:
- IoT Core message containing:
  - `slotId`: string (unique slot identifier)
  - `status`: enum (OCCUPIED, VACANT)
  - `sensorId`: string (sensor device identifier)
  - `timestamp`: ISO 8601 timestamp

**Outputs**:
- DynamoDB update to Slots table
- EventBridge event for slot status change

**Processing Logic**:
```
function updateSlotStatus(event):
    // Validate sensor authentication
    if not validateSensor(event.sensorId):
        return error("Unauthorized sensor")
    
    // Update slot status atomically
    DynamoDB.updateItem({
        tableName: "Slots",
        key: { slotId: event.slotId },
        updateExpression: "SET #status = :status, lastUpdated = :timestamp",
        conditionExpression: "attribute_exists(slotId)",
        expressionAttributeNames: { "#status": "status" },
        expressionAttributeValues: {
            ":status": event.status,
            ":timestamp": event.timestamp
        }
    })
    
    // Publish status change event
    EventBridge.putEvent({
        source: "parking.slot-monitor",
        detailType: "SlotStatusChanged",
        detail: {
            slotId: event.slotId,
            status: event.status,
            timestamp: event.timestamp
        }
    })
```

### 3. Slot Allocator (Lambda Function - AI Agent)

**Purpose**: Intelligently allocate parking slots based on vehicle type, proximity, and availability.

**Inputs**:
- EventBridge event from Vehicle Detector containing vehicle details

**Outputs**:
- DynamoDB updates to Vehicles and Slots tables
- EventBridge event for gate control

**Processing Logic**:
```
function allocateSlot(vehicleEvent):
    vehicleType = vehicleEvent.vehicleType
    
    // Query available slots
    availableSlots = DynamoDB.query({
        tableName: "Slots",
        indexName: "StatusIndex",
        keyConditionExpression: "#status = :vacant",
        expressionAttributeNames: { "#status": "status" },
        expressionAttributeValues: { ":vacant": "VACANT" }
    })
    
    if availableSlots.isEmpty():
        return rejectEntry("No slots available")
    
    // Rank slots by suitability
    rankedSlots = rankSlots(availableSlots, vehicleType)
    bestSlot = rankedSlots[0]
    
    // Reserve slot atomically
    try:
        DynamoDB.transactWriteItems([
            {
                Update: {
                    tableName: "Slots",
                    key: { slotId: bestSlot.slotId },
                    updateExpression: "SET #status = :occupied",
                    conditionExpression: "#status = :vacant"
                }
            },
            {
                Put: {
                    tableName: "Vehicles",
                    item: {
                        vehicleId: vehicleEvent.vehicleId,
                        licensePlate: vehicleEvent.licensePlate,
                        fasTagId: vehicleEvent.fasTagId,
                        vehicleType: vehicleEvent.vehicleType,
                        slotId: bestSlot.slotId,
                        entryTime: now(),
                        status: "PARKED"
                    }
                }
            }
        ])
    catch ConditionalCheckFailedException:
        // Slot was taken by another vehicle, retry
        return allocateSlot(vehicleEvent)
    
    // Notify gate controller
    EventBridge.putEvent({
        source: "parking.slot-allocator",
        detailType: "SlotAllocated",
        detail: {
            vehicleId: vehicleEvent.vehicleId,
            slotId: bestSlot.slotId,
            gateAction: "OPEN_ENTRY"
        }
    })

function rankSlots(slots, vehicleType):
    scored = []
    for slot in slots:
        score = 0
        
        // Prefer exact type match
        if slot.type == vehicleType:
            score += 100
        // Allow larger slots for smaller vehicles
        else if slot.type > vehicleType:
            score += 50
        else:
            continue  // Skip smaller slots
        
        // Prefer slots closer to entrance
        score += (1000 - slot.distanceFromEntrance)
        
        // Consider historical usage patterns
        if slot.averageOccupancyDuration < 2 hours:
            score += 20  // Prefer quick-turnover slots for short stays
        
        scored.append((slot, score))
    
    return sorted(scored, key=lambda x: x[1], reverse=True)
```

### 4. Violation Detector (Lambda Function - AI Agent)

**Purpose**: Continuously monitor parking sessions to detect violations (overstay, unauthorized parking).

**Inputs**:
- Scheduled EventBridge trigger (every 5 minutes)
- DynamoDB stream events for real-time detection

**Outputs**:
- Violation records in DynamoDB
- Notifications to operators

**Processing Logic**:
```
function detectViolations():
    currentTime = now()
    
    // Scan active parking sessions
    activeSessions = DynamoDB.scan({
        tableName: "Vehicles",
        filterExpression: "#status = :parked",
        expressionAttributeNames: { "#status": "status" },
        expressionAttributeValues: { ":parked": "PARKED" }
    })
    
    violations = []
    
    for session in activeSessions:
        // Check for overstay
        parkingDuration = currentTime - session.entryTime
        maxDuration = getMaxDuration(session.vehicleType)
        
        if parkingDuration > (maxDuration + 15 minutes):
            violation = {
                violationId: generateUUID(),
                vehicleId: session.vehicleId,
                type: "OVERSTAY",
                detectedAt: currentTime,
                duration: parkingDuration,
                penaltyAmount: calculateOverstayPenalty(parkingDuration, maxDuration)
            }
            violations.append(violation)
    
    // Check for unauthorized parking (vehicle in slot without record)
    occupiedSlots = DynamoDB.query({
        tableName: "Slots",
        indexName: "StatusIndex",
        keyConditionExpression: "#status = :occupied",
        expressionAttributeNames: { "#status": "status" },
        expressionAttributeValues: { ":occupied": "OCCUPIED" }
    })
    
    for slot in occupiedSlots:
        vehicleInSlot = DynamoDB.query({
            tableName: "Vehicles",
            indexName: "SlotIndex",
            keyConditionExpression: "slotId = :slotId AND #status = :parked",
            expressionAttributeValues: { ":slotId": slot.slotId, ":parked": "PARKED" }
        })
        
        if vehicleInSlot.isEmpty():
            violation = {
                violationId: generateUUID(),
                slotId: slot.slotId,
                type: "UNAUTHORIZED",
                detectedAt: currentTime,
                penaltyAmount: getUnauthorizedPenalty()
            }
            violations.append(violation)
    
    // Persist violations
    for violation in violations:
        DynamoDB.putItem({
            tableName: "Violations",
            item: violation
        })
        
        // Send notification
        SNS.publish({
            topic: "parking-violations",
            message: formatViolationAlert(violation)
        })
```

### 5. Payment Processor (Lambda Function)

**Purpose**: Calculate parking charges, process payments via FASTag, and manage exit flow.

**Inputs**:
- EventBridge event from exit gate containing FASTag ID

**Outputs**:
- Transaction record in DynamoDB
- Gate control event
- Payment confirmation

**Processing Logic**:
```
function processExit(exitEvent):
    fasTagId = exitEvent.fasTagId
    
    // Retrieve vehicle record
    vehicleRecord = DynamoDB.query({
        tableName: "Vehicles",
        indexName: "FASTagIndex",
        keyConditionExpression: "fasTagId = :fasTagId AND #status = :parked",
        expressionAttributeNames: { "#status": "status" },
        expressionAttributeValues: { ":fasTagId": fasTagId, ":parked": "PARKED" }
    })
    
    if vehicleRecord.isEmpty():
        return error("No active parking session found")
    
    vehicle = vehicleRecord[0]
    exitTime = now()
    duration = exitTime - vehicle.entryTime
    
    // Calculate base parking charges
    baseCharge = calculateParkingCharge(duration, vehicle.vehicleType)
    
    // Add violation penalties
    violations = DynamoDB.query({
        tableName: "Violations",
        indexName: "VehicleIndex",
        keyConditionExpression: "vehicleId = :vehicleId AND #status = :unpaid",
        expressionAttributeNames: { "#status": "status" },
        expressionAttributeValues: { ":vehicleId": vehicle.vehicleId, ":unpaid": "UNPAID" }
    })
    
    penaltyTotal = sum(v.penaltyAmount for v in violations)
    totalCharge = baseCharge + penaltyTotal
    
    // Process payment
    paymentResult = processFASTagPayment(fasTagId, totalCharge)
    
    if not paymentResult.success:
        return error("Payment failed: " + paymentResult.reason)
    
    // Create transaction and update records
    DynamoDB.transactWriteItems([
        {
            Put: {
                tableName: "Transactions",
                item: {
                    transactionId: generateUUID(),
                    vehicleId: vehicle.vehicleId,
                    fasTagId: fasTagId,
                    baseCharge: baseCharge,
                    penalties: penaltyTotal,
                    totalAmount: totalCharge,
                    paymentMethod: "FASTAG",
                    paymentStatus: "COMPLETED",
                    timestamp: exitTime
                }
            }
        },
        {
            Update: {
                tableName: "Vehicles",
                key: { vehicleId: vehicle.vehicleId },
                updateExpression: "SET #status = :exited, exitTime = :exitTime",
                expressionAttributeNames: { "#status": "status" },
                expressionAttributeValues: { ":exited": "EXITED", ":exitTime": exitTime }
            }
        },
        {
            Update: {
                tableName: "Slots",
                key: { slotId: vehicle.slotId },
                updateExpression: "SET #status = :vacant",
                expressionAttributeNames: { "#status": "status" },
                expressionAttributeValues: { ":vacant": "VACANT" }
            }
        }
    ])
    
    // Mark violations as paid
    for violation in violations:
        DynamoDB.updateItem({
            tableName: "Violations",
            key: { violationId: violation.violationId },
            updateExpression: "SET #status = :paid",
            expressionAttributeNames: { "#status": "status" },
            expressionAttributeValues: { ":paid": "PAID" }
        })
    
    // Open exit gate
    EventBridge.putEvent({
        source: "parking.payment-processor",
        detailType: "PaymentCompleted",
        detail: {
            vehicleId: vehicle.vehicleId,
            gateAction: "OPEN_EXIT"
        }
    })

function calculateParkingCharge(duration, vehicleType):
    config = getConfiguration()
    rates = config.rates[vehicleType]
    
    hours = ceil(duration / 1 hour)
    charge = hours * rates.hourlyRate
    
    // Apply time-based pricing
    if isPeakHour(now()):
        charge *= config.peakMultiplier
    
    return charge
```

### 6. Gate Controller (Lambda Function)

**Purpose**: Control physical entry and exit gates based on system events.

**Inputs**:
- EventBridge events for gate actions

**Outputs**:
- IoT Core commands to gate actuators

**Processing Logic**:
```
function controlGate(event):
    gateAction = event.detail.gateAction
    
    if gateAction == "OPEN_ENTRY":
        IoTCore.publish({
            topic: "parking/gates/entry/command",
            payload: { action: "OPEN", duration: 10 }
        })
    else if gateAction == "OPEN_EXIT":
        IoTCore.publish({
            topic: "parking/gates/exit/command",
            payload: { action: "OPEN", duration: 10 }
        })
    
    // Log gate operation
    CloudWatch.putMetricData({
        namespace: "SmartParking",
        metricName: "GateOperations",
        value: 1,
        dimensions: { GateType: gateAction }
    })
```

## Data Models

### DynamoDB Tables

#### Vehicles Table
```
{
    vehicleId: string (PK),
    licensePlate: string (GSI),
    fasTagId: string (GSI),
    vehicleType: string,  // COMPACT, SEDAN, SUV, TRUCK
    slotId: string (GSI),
    entryTime: string (ISO 8601),
    exitTime: string (ISO 8601, optional),
    status: string,  // PARKED, EXITED
    imageS3Key: string,
    confidence: number
}
```

**Indexes**:
- Primary Key: `vehicleId`
- GSI: `licensePlate-index`
- GSI: `fasTagId-status-index` (composite)
- GSI: `slotId-status-index` (composite)

#### Slots Table
```
{
    slotId: string (PK),
    type: string,  // COMPACT, SEDAN, SUV, TRUCK
    status: string (GSI),  // VACANT, OCCUPIED, UNKNOWN
    location: {
        zone: string,
        level: number,
        position: string
    },
    distanceFromEntrance: number,  // meters
    sensorId: string,
    lastUpdated: string (ISO 8601),
    averageOccupancyDuration: number  // seconds
}
```

**Indexes**:
- Primary Key: `slotId`
- GSI: `status-type-index` (composite)

#### Transactions Table
```
{
    transactionId: string (PK),
    vehicleId: string (GSI),
    fasTagId: string (GSI),
    baseCharge: number,
    penalties: number,
    totalAmount: number,
    paymentMethod: string,  // FASTAG, MANUAL
    paymentStatus: string,  // COMPLETED, FAILED, PENDING
    timestamp: string (ISO 8601, GSI)
}
```

**Indexes**:
- Primary Key: `transactionId`
- GSI: `vehicleId-index`
- GSI: `fasTagId-index`
- GSI: `timestamp-index`

#### Violations Table
```
{
    violationId: string (PK),
    vehicleId: string (GSI, optional),
    slotId: string (optional),
    type: string,  // OVERSTAY, UNAUTHORIZED
    detectedAt: string (ISO 8601),
    duration: number (optional),  // seconds
    penaltyAmount: number,
    status: string,  // UNPAID, PAID
    evidenceS3Keys: [string]  // image references
}
```

**Indexes**:
- Primary Key: `violationId`
- GSI: `vehicleId-status-index` (composite)
- GSI: `detectedAt-index`

#### Configuration Table
```
{
    configId: string (PK),  // "RATES", "PENALTIES", "SYSTEM"
    version: number,
    rates: {
        COMPACT: { hourlyRate: number },
        SEDAN: { hourlyRate: number },
        SUV: { hourlyRate: number },
        TRUCK: { hourlyRate: number }
    },
    penalties: {
        OVERSTAY: { baseAmount: number, perHourAmount: number },
        UNAUTHORIZED: { amount: number }
    },
    peakMultiplier: number,
    peakHours: [{ start: string, end: string }],
    maxDurations: {
        COMPACT: number,  // seconds
        SEDAN: number,
        SUV: number,
        TRUCK: number
    },
    updatedAt: string (ISO 8601),
    updatedBy: string
}
```

**Indexes**:
- Primary Key: `configId`

### Event Schemas

#### VehicleIdentified Event
```json
{
    "source": "parking.vehicle-detector",
    "detail-type": "VehicleIdentified",
    "detail": {
        "vehicleId": "uuid",
        "licensePlate": "string",
        "vehicleType": "COMPACT|SEDAN|SUV|TRUCK",
        "fasTagId": "string",
        "confidence": 0.98,
        "imageS3Key": "s3://bucket/path",
        "timestamp": "2024-01-15T10:30:00Z"
    }
}
```

#### SlotStatusChanged Event
```json
{
    "source": "parking.slot-monitor",
    "detail-type": "SlotStatusChanged",
    "detail": {
        "slotId": "string",
        "status": "OCCUPIED|VACANT",
        "timestamp": "2024-01-15T10:30:00Z"
    }
}
```

#### SlotAllocated Event
```json
{
    "source": "parking.slot-allocator",
    "detail-type": "SlotAllocated",
    "detail": {
        "vehicleId": "uuid",
        "slotId": "string",
        "gateAction": "OPEN_ENTRY",
        "timestamp": "2024-01-15T10:30:00Z"
    }
}
```

#### PaymentCompleted Event
```json
{
    "source": "parking.payment-processor",
    "detail-type": "PaymentCompleted",
    "detail": {
        "vehicleId": "uuid",
        "transactionId": "uuid",
        "totalAmount": 150.00,
        "gateAction": "OPEN_EXIT",
        "timestamp": "2024-01-15T12:30:00Z"
    }
}
```


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: License Plate Extraction Consistency

*For any* vehicle image containing a visible license plate, when processed by the Vehicle_Detector, the extracted license plate text should be consistent across multiple extractions of the same image.

**Validates: Requirements 1.2**

### Property 2: License Plate and FASTag Validation

*For any* license plate and FASTag pair, the validation function should correctly identify whether they match according to the registered vehicle database, and mismatches should be flagged as suspicious.

**Validates: Requirements 1.3, 1.4**

### Property 3: Vehicle Record Completeness

*For any* successful vehicle identification, the created Vehicle_Record should contain all required fields: vehicleId, licensePlate, fasTagId, vehicleType, entryTime, and status.

**Validates: Requirements 1.5**

### Property 4: Slot Status Synchronization

*For any* parking slot status change event (occupied or vacant), the Database slot record should reflect the new status, and the system's occupancy count for that vehicle type should be updated accordingly.

**Validates: Requirements 2.2, 2.3, 2.4, 2.5**

### Property 5: Occupancy Query Accuracy

*For any* point in time, querying the system for current occupancy should return counts that exactly match the number of slots marked as occupied in the database for each vehicle type.

**Validates: Requirements 2.6**

### Property 6: Vehicle Type Classification Determinism

*For any* vehicle image, the vehicle type classification should be deterministic—processing the same image multiple times should yield the same vehicle type classification.

**Validates: Requirements 3.1, 8.4**

### Property 7: Slot Type Matching Priority

*For any* slot allocation request where slots matching the vehicle type are available, the allocated slot's type should match the vehicle type.

**Validates: Requirements 3.2**

### Property 8: Proximity Optimization

*For any* slot allocation where multiple slots of the same suitability exist, the selected slot should have the minimum distanceFromEntrance value among all suitable slots.

**Validates: Requirements 3.3**

### Property 9: Slot Type Fallback

*For any* vehicle requiring a slot where no exact type match exists but larger slot types are available, the allocated slot type should be larger than the vehicle type.

**Validates: Requirements 3.4**

### Property 10: Entry Rejection When Full

*For any* vehicle entry request when all parking slots are occupied, the system should reject the entry and not create a Vehicle_Record with status "PARKED".

**Validates: Requirements 3.5**

### Property 11: Slot Exclusivity Invariant

*For any* point in time, no parking slot should be allocated to more than one vehicle with status "PARKED" simultaneously—the slotId in the Vehicles table should be unique among all PARKED vehicles.

**Validates: Requirements 3.7**

### Property 12: Overstay Violation Detection

*For any* vehicle with parking duration exceeding (maxDuration + 15 minutes), when the Violation_Detector scans, a Violation record of type "OVERSTAY" should be created.

**Validates: Requirements 4.2**

### Property 13: Unauthorized Parking Detection

*For any* parking slot with status "OCCUPIED" that has no corresponding Vehicle_Record with status "PARKED" and matching slotId, a Violation record of type "UNAUTHORIZED" should be created.

**Validates: Requirements 4.3**

### Property 14: Penalty Calculation Correctness

*For any* violation, the calculated penaltyAmount should match the formula defined in the Configuration table based on violation type and duration.

**Validates: Requirements 4.4**

### Property 15: Audit Trail Completeness

*For any* parking session (entry to exit), transaction, violation, or configuration change, there should be a corresponding record in the database with timestamp and relevant identifiers.

**Validates: Requirements 4.6, 10.6, 12.4, 15.6**

### Property 16: Penalty Inclusion in Transactions

*For any* vehicle with unpaid violations at exit time, the Transaction totalAmount should equal baseCharge plus the sum of all penaltyAmounts from associated violations.

**Validates: Requirements 5.1, 6.4**

### Property 17: Evidence Capture for Violations

*For any* violation of type "UNAUTHORIZED", the Violation record should contain at least one evidenceS3Key referencing a captured image.

**Validates: Requirements 5.2**

### Property 18: Progressive Penalty Escalation

*For any* vehicle with multiple violations, the total penalty amount should be greater than or equal to the sum of individual violation penalties (allowing for escalation multipliers).

**Validates: Requirements 5.3**

### Property 19: Exit Prevention for Unpaid Violations

*For any* vehicle with violations where status is "UNPAID", the Payment_Processor should not open the exit gate and should return a payment failure error.

**Validates: Requirements 5.4**

### Property 20: Violation Resolution State Transition

*For any* violation that is paid, the violation status should transition from "UNPAID" to "PAID", and this transition should be irreversible.

**Validates: Requirements 5.5**

### Property 21: Parking Charge Calculation Formula

*For any* parking session, the baseCharge should equal: ceil(duration / 1 hour) × hourlyRate[vehicleType] × peakMultiplier (if during peak hours, otherwise 1.0).

**Validates: Requirements 6.3**

### Property 22: Payment Failure Handling

*For any* payment processing that fails, the Exit_Gate should not receive an open command, and the vehicle status should remain "PARKED".

**Validates: Requirements 6.6**

### Property 23: Transaction Record Completeness

*For any* successful payment, the created Transaction record should contain all required fields: transactionId, vehicleId, fasTagId, baseCharge, penalties, totalAmount, paymentMethod, paymentStatus, and timestamp.

**Validates: Requirements 6.7**

### Property 24: Exit State Transition Atomicity

*For any* vehicle exit, the following state changes should occur atomically: vehicle status changes to "EXITED", slot status changes to "VACANT", and a Transaction record is created—either all succeed or all fail.

**Validates: Requirements 6.9**

### Property 25: Sensor Disconnect Handling

*For any* IoT sensor disconnect event, the associated parking slot status should be set to "UNKNOWN" until the sensor reconnects and publishes its current status.

**Validates: Requirements 7.2**

### Property 26: Unauthenticated Message Rejection

*For any* IoT message that fails authentication, the IoT_Gateway should discard the message and not update any slot status in the database.

**Validates: Requirements 7.4**

### Property 27: Malformed Data Handling

*For any* sensor message with malformed data (invalid JSON, missing required fields, invalid enum values), the system should log an error and discard the message without updating state.

**Validates: Requirements 7.5, 14.4**

### Property 28: Multiple Vehicle Image Processing

*For any* image containing multiple vehicles, the Vehicle_Detector should process only the vehicle with the smallest distance to the camera (closest vehicle).

**Validates: Requirements 8.5**

### Property 29: Database Record Uniqueness

*For any* Vehicle_Record, Transaction, or Violation created, the primary key (vehicleId, transactionId, violationId) should be unique and not conflict with existing records.

**Validates: Requirements 9.1, 9.3**

### Property 30: Atomic Slot Status Updates

*For any* slot status update operation, the database update should be atomic—no partial updates should be visible, and concurrent reads should see either the old or new state, never an intermediate state.

**Validates: Requirements 9.2**

### Property 31: Optimistic Locking for Concurrent Updates

*For any* two concurrent update operations on the same database record, at most one should succeed, and the other should fail with a conflict error that can be retried.

**Validates: Requirements 9.7**

### Property 32: Slot Ranking Correctness

*For any* set of available slots, the Slot_Allocator's ranking function should order slots such that: exact type matches score higher than larger types, and among equal type matches, closer slots (lower distanceFromEntrance) score higher.

**Validates: Requirements 10.2**

### Property 33: Allocation Decision Factors

*For any* slot allocation decision, the selected slot should be influenced by current occupancy levels—if occupancy is above 80%, prefer slots with historically shorter occupancy durations.

**Validates: Requirements 10.3**

### Property 34: Operation Prioritization Under Load

*For any* high system load scenario (>80% capacity), entry and exit operations should be processed before background tasks like violation scanning.

**Validates: Requirements 10.5**

### Property 35: Unauthenticated API Request Rejection

*For any* API request without valid AWS IAM authentication, the system should return a 401 Unauthorized error and not process the request.

**Validates: Requirements 12.3**

### Property 36: Rate Limiting Enforcement

*For any* client making more than 100 requests per minute, subsequent requests should be rejected with a 429 Too Many Requests error.

**Validates: Requirements 12.5**

### Property 37: Suspicious Activity Response

*For any* detected suspicious activity (multiple failed authentications, unusual access patterns), the system should trigger a security alert and temporarily block the source IP or client.

**Validates: Requirements 12.6**

### Property 38: Metrics Publication for Operations

*For any* key operation (vehicle entry, exit, slot allocation, payment processing), at least one metric should be published to CloudWatch with the operation name and timestamp.

**Validates: Requirements 13.1**

### Property 39: Error Logging Completeness

*For any* error or exception that occurs, a log entry should be created containing: error message, stack trace, timestamp, and contextual information (vehicleId, slotId, etc.).

**Validates: Requirements 13.2**

### Property 40: Webhook Notification for Events

*For any* key event (vehicle entry, exit, violation detected, payment completed), if webhooks are configured, a webhook notification should be sent to the registered endpoint.

**Validates: Requirements 14.6**

### Property 41: Configuration Hot Reload

*For any* configuration change (rates, penalties, slot definitions), the new configuration should be applied to subsequent operations without requiring system restart, and the change should be visible within the specified time window.

**Validates: Requirements 15.4**

### Property 42: Configuration Validation

*For any* configuration update with invalid values (negative rates, invalid vehicle types, malformed data), the system should reject the update and return a validation error without modifying the existing configuration.

**Validates: Requirements 15.5**

## Error Handling

### Error Categories

1. **Transient Errors**: Temporary failures that can be retried
   - AWS service throttling
   - Network timeouts
   - Database connection failures
   
2. **Permanent Errors**: Failures that should not be retried
   - Invalid vehicle image format
   - Malformed FASTag data
   - Authentication failures

3. **Business Logic Errors**: Expected error conditions
   - No available parking slots
   - Payment failure
   - Unauthorized parking

### Error Handling Strategies

**Retry with Exponential Backoff**:
- Apply to all AWS service calls (Rekognition, DynamoDB, IoT Core)
- Maximum 3 retry attempts
- Exponential backoff: 100ms, 200ms, 400ms
- Jitter to prevent thundering herd

**Dead Letter Queues (DLQ)**:
- All Lambda functions should have DLQ configured
- Failed events after max retries go to DLQ
- Separate DLQs for different failure types
- Monitoring and alerting on DLQ depth

**Circuit Breaker Pattern**:
- Implement for external payment gateway integration
- Open circuit after 5 consecutive failures
- Half-open state after 30 seconds
- Close circuit after 2 successful calls

**Graceful Degradation**:
- If Rekognition is unavailable, fall back to manual verification
- If violation detection fails, continue normal operations and retry later
- If metrics publishing fails, log locally and continue

**Error Response Format**:
```json
{
    "error": {
        "code": "SLOT_UNAVAILABLE",
        "message": "No parking slots available for vehicle type SUV",
        "timestamp": "2024-01-15T10:30:00Z",
        "requestId": "uuid",
        "retryable": false
    }
}
```

### Specific Error Scenarios

**Vehicle Detection Failure**:
1. Retry image capture up to 3 times
2. If still failing, request manual verification
3. Log failure with image reference
4. Notify operator for manual intervention

**Slot Allocation Conflict**:
1. Detect optimistic locking failure
2. Retry allocation with fresh slot query
3. Maximum 5 retry attempts
4. If all retries fail, reject entry

**Payment Processing Failure**:
1. Prevent exit gate from opening
2. Display error message to driver
3. Log payment failure details
4. Provide manual payment option
5. Notify operator

**Sensor Communication Failure**:
1. Mark slot status as "UNKNOWN"
2. Exclude from allocation until reconnected
3. Alert operator if offline > 5 minutes
4. Log all communication failures

**Database Write Failure**:
1. Retry with exponential backoff
2. If transaction fails, rollback all changes
3. Log failure with full context
4. Trigger high-priority alert

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests**: Focus on specific examples, edge cases, and integration points
- Specific vehicle images with known license plates
- Specific slot allocation scenarios
- Payment calculation examples
- Error condition handling
- AWS service integration mocking

**Property-Based Tests**: Verify universal properties across randomized inputs
- Generate random vehicle data, slot configurations, and parking sessions
- Test invariants that must hold for all valid inputs
- Verify state transitions and data consistency
- Test concurrent operations and race conditions

### Property-Based Testing Configuration

**Framework**: Use `fast-check` for TypeScript/JavaScript Lambda functions

**Test Configuration**:
- Minimum 100 iterations per property test
- Seed-based reproducibility for failed tests
- Shrinking to find minimal failing examples
- Parallel execution where possible

**Property Test Structure**:
```typescript
// Example property test structure
describe('Feature: agentic-smart-parking, Property 11: Slot Exclusivity Invariant', () => {
    it('no slot should be allocated to multiple vehicles simultaneously', async () => {
        await fc.assert(
            fc.asyncProperty(
                fc.array(vehicleGenerator(), { minLength: 1, maxLength: 50 }),
                fc.array(slotGenerator(), { minLength: 1, maxLength: 100 }),
                async (vehicles, slots) => {
                    // Setup: Initialize database with slots
                    await setupSlots(slots);
                    
                    // Execute: Allocate slots for all vehicles concurrently
                    const allocations = await Promise.allSettled(
                        vehicles.map(v => allocateSlot(v))
                    );
                    
                    // Verify: No slot is allocated to multiple vehicles
                    const parkedVehicles = await getParkedVehicles();
                    const slotIds = parkedVehicles.map(v => v.slotId);
                    const uniqueSlotIds = new Set(slotIds);
                    
                    expect(slotIds.length).toBe(uniqueSlotIds.size);
                }
            ),
            { numRuns: 100 }
        );
    });
});
```

### Test Data Generators

**Vehicle Generator**:
```typescript
const vehicleGenerator = () => fc.record({
    licensePlate: fc.stringMatching(/^[A-Z]{2}[0-9]{2}[A-Z]{2}[0-9]{4}$/),
    fasTagId: fc.uuid(),
    vehicleType: fc.constantFrom('COMPACT', 'SEDAN', 'SUV', 'TRUCK'),
    imageS3Key: fc.string().map(s => `s3://bucket/images/${s}.jpg`)
});
```

**Slot Generator**:
```typescript
const slotGenerator = () => fc.record({
    slotId: fc.uuid(),
    type: fc.constantFrom('COMPACT', 'SEDAN', 'SUV', 'TRUCK'),
    status: fc.constantFrom('VACANT', 'OCCUPIED'),
    distanceFromEntrance: fc.integer({ min: 10, max: 500 }),
    location: fc.record({
        zone: fc.constantFrom('A', 'B', 'C', 'D'),
        level: fc.integer({ min: 1, max: 5 }),
        position: fc.string({ minLength: 1, maxLength: 3 })
    })
});
```

**Parking Session Generator**:
```typescript
const parkingSessionGenerator = () => fc.record({
    vehicleId: fc.uuid(),
    entryTime: fc.date({ min: new Date('2024-01-01'), max: new Date() }),
    duration: fc.integer({ min: 300, max: 86400 }), // 5 min to 24 hours
    vehicleType: fc.constantFrom('COMPACT', 'SEDAN', 'SUV', 'TRUCK')
});
```

### Unit Test Coverage

**Component-Level Tests**:
1. Vehicle Detector
   - Test with sample images
   - Mock AWS Rekognition responses
   - Test error handling for failed detection
   - Test retry logic

2. Slot Allocator
   - Test allocation with various slot configurations
   - Test fallback to larger slots
   - Test rejection when full
   - Test concurrent allocation conflicts

3. Violation Detector
   - Test overstay detection with specific durations
   - Test unauthorized parking detection
   - Test penalty calculation examples

4. Payment Processor
   - Test charge calculation with specific rates
   - Test penalty inclusion
   - Test payment failure handling
   - Test transaction creation

5. Slot Monitor
   - Test status update processing
   - Test malformed message handling
   - Test authentication validation

### Integration Tests

**End-to-End Flows**:
1. Complete entry flow: FASTag → Detection → Allocation → Gate Open
2. Complete exit flow: FASTag → Payment → Gate Open → Slot Vacant
3. Violation detection and penalty application
4. Configuration update and application

**AWS Service Integration**:
- Test with actual AWS services in test environment
- Use LocalStack for local AWS service emulation
- Test IAM permissions and authentication
- Test DynamoDB streams and EventBridge routing

### Performance Testing

**Load Testing**:
- Simulate 50 concurrent entries per minute
- Simulate 50 concurrent exits per minute
- Measure response times under load
- Verify auto-scaling behavior

**Stress Testing**:
- Test system behavior at 2x expected load
- Identify breaking points
- Verify graceful degradation

### Monitoring and Observability in Tests

**Test Instrumentation**:
- Capture CloudWatch metrics during tests
- Verify log entries are created
- Test alarm triggering conditions
- Verify trace data completeness

**Chaos Engineering**:
- Inject failures (network, database, AWS services)
- Verify error handling and recovery
- Test circuit breaker behavior
- Verify DLQ processing

### Continuous Testing

**CI/CD Pipeline**:
1. Unit tests run on every commit
2. Property-based tests run on every PR
3. Integration tests run on merge to main
4. Performance tests run nightly
5. Chaos tests run weekly

**Test Environments**:
- Local: LocalStack + DynamoDB Local
- Dev: Isolated AWS account with test data
- Staging: Production-like environment
- Production: Canary deployments with monitoring
