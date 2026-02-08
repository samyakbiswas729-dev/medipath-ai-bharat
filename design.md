# Design Document: Medical Record Management with Blockchain Audit Trail

## Overview

This design implements a secure medical record management system with blockchain-based audit trails for the Medipath platform. The system leverages the existing blockchain infrastructure (`SimpleBlockchain`, `Block`, `AuditService`) and extends it with comprehensive medical record management capabilities.

### Key Design Principles

1. **Separation of Concerns**: Medical record business logic is separate from blockchain audit logic
2. **Immutability**: All medical actions are logged as immutable blockchain events
3. **Cryptographic Security**: SHA-256 hashing ensures tamper-proof audit trails
4. **Role-Based Security**: Clear separation between doctor (write) and patient (read) permissions
5. **Fail-Safe Operations**: Blockchain verification before critical operations
6. **Persistence**: Dual storage (in-memory blockchain + database) for reliability

### Architecture Philosophy

The design follows a layered architecture:
- **Presentation Layer**: REST API controllers exposing medical record and audit endpoints
- **Business Logic Layer**: Services handling medical record operations and blockchain integration
- **Data Access Layer**: JPA repositories for database operations
- **Blockchain Layer**: Existing `SimpleBlockchain` and `AuditService` for immutable audit logging

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────┐
│                     REST API Layer                          │
│  ┌──────────────────────┐  ┌─────────────────────────────┐ │
│  │ MedicalRecordController│  │   AuditController (exists)  │ │
│  └──────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Service Layer                             │
│  ┌──────────────────────┐  ┌─────────────────────────────┐ │
│  │ MedicalRecordService │  │  AuditService (exists)      │ │
│  │  - CRUD operations   │  │  - Blockchain operations    │ │
│  │  - Validation        │  │  - Chain verification       │ │
│  │  - Audit integration │  │  - Event logging            │ │
│  └──────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                 Repository Layer                            │
│  ┌──────────────────────┐  ┌─────────────────────────────┐ │
│  │MedicalRecordRepository│  │ BlockRepository (exists)    │ │
│  │PatientRepository      │  │                             │ │
│  └──────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Layer                               │
│  ┌──────────────────────┐  ┌─────────────────────────────┐ │
│  │  MedicalRecord       │  │  BlockEntity (exists)       │ │
│  │  Patient (exists)    │  │  SimpleBlockchain (exists)  │ │
│  └──────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

**Medical Record Creation Flow:**
```
Doctor → POST /api/medical-records
  ↓
MedicalRecordController.createRecord()
  ↓
MedicalRecordService.createRecord()
  ├─→ Validate input data
  ├─→ Verify patient exists
  ├─→ Save to database (MedicalRecordRepository)
  └─→ AuditService.addAudit() → Blockchain event
      ↓
Return MedicalRecord + confirmation
```

**Chain Verification Flow:**
```
Admin → GET /api/audit/verify
  ↓
AuditController.verifyChain()
  ↓
AuditService.verifyChain()
  ↓
SimpleBlockchain.validateChain()
  ├─→ Check each block's prevHash
  ├─→ Recompute each block's hash
  └─→ Verify proof-of-work difficulty
      ↓
Return validation result
```

## Components and Interfaces

### MedicalRecord Entity

```java
@Entity
@Table(name = "medical_records")
public class MedicalRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String patientId;
    
    @Column(nullable = false)
    private String doctorId;
    
    @Column(nullable = false, length = 1000)
    private String diagnosis;
    
    @Column(nullable = false, length = 2000)
    private String treatmentPlan;
    
    @Column(length = 2000)
    private String medications;
    
    @Lob
    @Column(columnDefinition = "TEXT")
    private String clinicalNotes;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @Column(nullable = false)
    private LocalDateTime updatedAt;
    
    @Column(nullable = false)
    private String createdBy;
    
    @Column(nullable = false)
    private String lastModifiedBy;
    
    // Getters and setters
}
```

### MedicalRecordService Interface

```java
public interface MedicalRecordService {
    /**
     * Creates a new medical record and logs to blockchain
     * @param request DTO containing medical record data
     * @param doctorId ID of the doctor creating the record
     * @return Created medical record with ID
     * @throws ValidationException if data is invalid
     * @throws PatientNotFoundException if patient doesn't exist
     */
    MedicalRecordDTO createRecord(CreateMedicalRecordRequest request, String doctorId);
    
    /**
     * Updates an existing medical record and logs to blockchain
     * @param recordId ID of the record to update
     * @param request DTO containing updated data
     * @param doctorId ID of the doctor updating the record
     * @return Updated medical record
     * @throws RecordNotFoundException if record doesn't exist
     * @throws ValidationException if data is invalid
     */
    MedicalRecordDTO updateRecord(Long recordId, UpdateMedicalRecordRequest request, String doctorId);
    
    /**
     * Retrieves all medical records for a patient
     * @param patientId ID of the patient
     * @param requesterId ID of the user requesting records
     * @param requesterRole Role of the requester (DOCTOR/PATIENT)
     * @return List of medical records with verification status
     * @throws UnauthorizedException if requester lacks permission
     */
    List<MedicalRecordDTO> getRecordsByPatient(String patientId, String requesterId, UserRole requesterRole);
    
    /**
     * Retrieves a specific medical record by ID
     * @param recordId ID of the record
     * @param requesterId ID of the user requesting the record
     * @param requesterRole Role of the requester
     * @return Medical record with verification status
     * @throws RecordNotFoundException if record doesn't exist
     * @throws UnauthorizedException if requester lacks permission
     */
    MedicalRecordDTO getRecordById(Long recordId, String requesterId, UserRole requesterRole);
    
    /**
     * Retrieves audit trail for a patient's medical records
     * @param patientId ID of the patient
     * @param requesterId ID of the user requesting audit trail
     * @param requesterRole Role of the requester
     * @return List of audit events from blockchain
     * @throws UnauthorizedException if requester lacks permission
     */
    List<AuditEventDTO> getAuditTrail(String patientId, String requesterId, UserRole requesterRole);
}
```

### MedicalRecordController Endpoints

```java
@RestController
@RequestMapping("/api/medical-records")
@CrossOrigin(origins = "http://localhost:3000")
public class MedicalRecordController {
    
    /**
     * POST /api/medical-records
     * Creates a new medical record
     * Request body: CreateMedicalRecordRequest
     * Headers: X-User-Id (doctor ID), X-User-Role (DOCTOR)
     * Response: 201 Created with MedicalRecordDTO
     */
    @PostMapping
    public ResponseEntity<MedicalRecordDTO> createRecord(
        @RequestBody CreateMedicalRecordRequest request,
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader("X-User-Role") String userRole
    );
    
    /**
     * PUT /api/medical-records/{id}
     * Updates an existing medical record
     * Path variable: id (record ID)
     * Request body: UpdateMedicalRecordRequest
     * Headers: X-User-Id (doctor ID), X-User-Role (DOCTOR)
     * Response: 200 OK with updated MedicalRecordDTO
     */
    @PutMapping("/{id}")
    public ResponseEntity<MedicalRecordDTO> updateRecord(
        @PathVariable Long id,
        @RequestBody UpdateMedicalRecordRequest request,
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader("X-User-Role") String userRole
    );
    
    /**
     * GET /api/medical-records/patient/{patientId}
     * Retrieves all medical records for a patient
     * Path variable: patientId
     * Headers: X-User-Id, X-User-Role (DOCTOR or PATIENT)
     * Response: 200 OK with List<MedicalRecordDTO>
     */
    @GetMapping("/patient/{patientId}")
    public ResponseEntity<List<MedicalRecordDTO>> getRecordsByPatient(
        @PathVariable String patientId,
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader("X-User-Role") String userRole
    );
    
    /**
     * GET /api/medical-records/{id}
     * Retrieves a specific medical record
     * Path variable: id (record ID)
     * Headers: X-User-Id, X-User-Role
     * Response: 200 OK with MedicalRecordDTO
     */
    @GetMapping("/{id}")
    public ResponseEntity<MedicalRecordDTO> getRecordById(
        @PathVariable Long id,
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader("X-User-Role") String userRole
    );
    
    /**
     * GET /api/medical-records/patient/{patientId}/audit
     * Retrieves audit trail for a patient
     * Path variable: patientId
     * Headers: X-User-Id, X-User-Role
     * Response: 200 OK with List<AuditEventDTO>
     */
    @GetMapping("/patient/{patientId}/audit")
    public ResponseEntity<List<AuditEventDTO>> getAuditTrail(
        @PathVariable String patientId,
        @RequestHeader("X-User-Id") String userId,
        @RequestHeader("X-User-Role") String userRole
    );
}
```

### Audit Event Structure

Audit events stored in blockchain blocks follow this structure:

```java
Map<String, Object> auditData = Map.of(
    "action", "CREATE" | "UPDATE" | "VIEW" | "DELETE",
    "userId", "doctor123",
    "userRole", "DOCTOR" | "PATIENT",
    "patientId", "patient456",
    "recordId", 789L,
    "timestamp", System.currentTimeMillis(),
    "metadata", Map.of(
        "changedFields", List.of("diagnosis", "medications"),
        "previousValues", Map.of("diagnosis", "old value")
    )
);
```

### Role-Based Access Control

```java
public enum UserRole {
    DOCTOR,  // Can CREATE, READ, UPDATE medical records
    PATIENT  // Can READ only their own medical records
}

public class AccessControlService {
    /**
     * Validates if a user has permission to perform an action
     * @param userId ID of the user
     * @param role Role of the user
     * @param action Action being performed
     * @param patientId Patient ID affected by the action
     * @return true if authorized, false otherwise
     */
    public boolean hasPermission(String userId, UserRole role, String action, String patientId) {
        switch (role) {
            case DOCTOR:
                return true; // Doctors can perform all actions
            case PATIENT:
                // Patients can only read their own records
                return action.equals("READ") && userId.equals(patientId);
            default:
                return false;
        }
    }
}
```

## Data Models

### Database Schema

**medical_records table:**
```sql
CREATE TABLE medical_records (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    patient_id VARCHAR(255) NOT NULL,
    doctor_id VARCHAR(255) NOT NULL,
    diagnosis VARCHAR(1000) NOT NULL,
    treatment_plan VARCHAR(2000) NOT NULL,
    medications VARCHAR(2000),
    clinical_notes TEXT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    created_by VARCHAR(255) NOT NULL,
    last_modified_by VARCHAR(255) NOT NULL,
    INDEX idx_patient_id (patient_id),
    INDEX idx_doctor_id (doctor_id),
    INDEX idx_created_at (created_at),
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id)
);
```

**audit_blocks table (existing):**
```sql
CREATE TABLE audit_blocks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idx INTEGER UNIQUE NOT NULL,
    timestamp DOUBLE NOT NULL,
    prev_hash VARCHAR(256) NOT NULL,
    data_json TEXT NOT NULL,
    nonce INTEGER NOT NULL,
    hash VARCHAR(256) NOT NULL,
    INDEX idx_timestamp (timestamp),
    INDEX idx_hash (hash)
);
```

### DTOs (Data Transfer Objects)

**CreateMedicalRecordRequest:**
```java
public class CreateMedicalRecordRequest {
    @NotBlank(message = "Patient ID is required")
    private String patientId;
    
    @NotBlank(message = "Diagnosis is required")
    @Size(max = 1000, message = "Diagnosis must not exceed 1000 characters")
    private String diagnosis;
    
    @NotBlank(message = "Treatment plan is required")
    @Size(max = 2000, message = "Treatment plan must not exceed 2000 characters")
    private String treatmentPlan;
    
    @Size(max = 2000, message = "Medications must not exceed 2000 characters")
    private String medications;
    
    private String clinicalNotes;
    
    // Getters and setters
}
```

**UpdateMedicalRecordRequest:**
```java
public class UpdateMedicalRecordRequest {
    @Size(max = 1000, message = "Diagnosis must not exceed 1000 characters")
    private String diagnosis;
    
    @Size(max = 2000, message = "Treatment plan must not exceed 2000 characters")
    private String treatmentPlan;
    
    @Size(max = 2000, message = "Medications must not exceed 2000 characters")
    private String medications;
    
    private String clinicalNotes;
    
    // Getters and setters
}
```

**MedicalRecordDTO:**
```java
public class MedicalRecordDTO {
    private Long id;
    private String patientId;
    private String patientName;
    private String doctorId;
    private String diagnosis;
    private String treatmentPlan;
    private String medications;
    private String clinicalNotes;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String createdBy;
    private String lastModifiedBy;
    private boolean blockchainVerified;
    
    // Getters and setters
}
```

**AuditEventDTO:**
```java
public class AuditEventDTO {
    private int blockIndex;
    private double timestamp;
    private String action;
    private String userId;
    private String userRole;
    private String patientId;
    private Long recordId;
    private Map<String, Object> metadata;
    private String blockHash;
    private boolean verified;
    
    // Getters and setters
}
```

### Blockchain Integration

The system uses the existing `SimpleBlockchain` class with the following characteristics:

- **Difficulty**: 2 (requires hash to start with "00")
- **Hash Algorithm**: SHA-256
- **Block Structure**: index, timestamp, prevHash, data (Map), nonce, hash
- **Proof-of-Work**: Mining to find valid nonce
- **Persistence**: Dual storage (in-memory + database via BlockEntity)

**Blockchain Operations:**

1. **Add Audit Event**: `AuditService.addAudit(Map<String, Object> data)`
   - Creates new block with audit data
   - Performs proof-of-work mining
   - Links to previous block via prevHash
   - Persists to database

2. **Verify Chain**: `AuditService.verifyChain()`
   - Validates all block hashes
   - Checks prevHash linkage
   - Verifies proof-of-work difficulty
   - Returns validation result

3. **Load Chain**: `AuditService.loadPersisted()`
   - Loads blocks from database on startup
   - Reconstructs in-memory blockchain
   - Verifies integrity after loading


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property Reflection

After analyzing all acceptance criteria, I identified several areas where properties can be consolidated:

1. **Audit event metadata properties (5.1, 12.1-12.5)** can be combined into comprehensive properties that verify all required fields are present
2. **Chain verification properties (6.1-6.3)** are all part of the overall chain validation and can be tested together
3. **Record field validation (1.2, 11.2)** both test that required fields are present and can be combined
4. **Blockchain persistence properties (5.4, 10.2)** both test dual storage and can be combined
5. **Authorization properties (3.4, 4.3, 8.5)** all test permission enforcement and can be consolidated

The following properties represent the unique, non-redundant correctness guarantees for this system:

### Medical Record Management Properties

**Property 1: Medical record creation persistence**
*For any* valid medical record data (with patient ID, diagnosis, treatment plan), when a doctor creates the record, the system should store it in the database and return a record with a valid ID.
**Validates: Requirements 1.1, 1.5**

**Property 2: Medical record completeness**
*For any* created medical record, the stored record should contain all submitted fields including patient identifier, diagnosis, treatment plan, medications, and clinical notes.
**Validates: Requirements 1.2, 11.2**

**Property 3: Invalid record rejection**
*For any* medical record data with missing required fields (patient ID, diagnosis, or treatment plan), the system should reject the creation and return validation errors.
**Validates: Requirements 1.4, 11.2**

**Property 4: Patient existence validation**
*For any* medical record submission with a patient identifier that does not exist in the system, the system should reject the submission with a patient not found error.
**Validates: Requirements 11.1**

**Property 5: Medical record update persistence**
*For any* existing medical record and valid update data, when a doctor updates the record, the system should persist all changes and return the updated record with a new modification timestamp.
**Validates: Requirements 2.1, 2.5**

**Property 6: Invalid update rejection**
*For any* medical record update with invalid data (exceeding length limits or invalid formats), the system should reject the update and return specific validation errors.
**Validates: Requirements 2.4, 11.4**

**Property 7: Medical record retrieval completeness**
*For any* patient identifier, when records are requested, the system should return all medical records associated with that patient, including all historical modification timestamps.
**Validates: Requirements 3.1, 3.2**

**Property 8: Date validation**
*For any* medical record with date fields, the system should validate that dates are in valid formats and within logical ranges (not in the future, not before 1900).
**Validates: Requirements 11.3**

**Property 9: Valid record acceptance**
*For any* medical record that passes all validation rules, the system should successfully create or update the record without errors.
**Validates: Requirements 11.5**

### Blockchain Audit Properties

**Property 10: Audit event creation for all actions**
*For any* medical action (create, update, view), the blockchain should contain a corresponding audit event with action type, user identifier, user role, patient identifier, and timestamp.
**Validates: Requirements 1.3, 2.2, 3.3, 5.1, 12.1, 12.2, 12.3, 12.5**

**Property 11: Update event change tracking**
*For any* medical record update, the blockchain audit event should include the specific fields that were modified and their previous values.
**Validates: Requirements 2.2, 12.4**

**Property 12: Blockchain hash linkage**
*For any* two consecutive blocks in the blockchain, the later block's prevHash field should equal the earlier block's hash field.
**Validates: Requirements 5.2, 6.1**

**Property 13: Proof-of-work compliance**
*For any* block in the blockchain (except genesis), the block's hash should start with the required number of zeros (difficulty = 2) and be computable from the block's data using SHA-256.
**Validates: Requirements 5.3, 6.2, 6.3**

**Property 14: Dual persistence**
*For any* audit event added to the blockchain, the block should exist in both the in-memory chain and the database with identical data.
**Validates: Requirements 5.4, 10.2**

**Property 15: Audit event chronological ordering**
*For any* query for audit events, the returned events should be ordered by timestamp from oldest to newest, and block indices should be strictly increasing.
**Validates: Requirements 5.5, 8.3**

**Property 16: Chain verification completeness**
*For any* blockchain state, verification should check all three conditions: prevHash linkage, hash recomputation, and proof-of-work difficulty, returning success only if all pass.
**Validates: Requirements 6.1, 6.2, 6.3, 6.5**

**Property 17: Tamper detection precision**
*For any* blockchain where a single block's data is modified, verification should detect the tampering and return the exact block index where the modification occurred.
**Validates: Requirements 7.1, 7.2, 6.4**

**Property 18: Chain break detection**
*For any* blockchain where a block's prevHash is modified to not match the previous block's hash, verification should detect the break in chain linkage and identify the affected block.
**Validates: Requirements 7.3, 6.4**

**Property 19: Legitimate operations preserve validity**
*For any* valid blockchain, after adding a new legitimate audit event through the proper API, chain verification should still return valid.
**Validates: Requirements 7.5**

**Property 20: Audit trail filtering**
*For any* patient identifier, the audit trail query should return only events where the patient identifier in the event data matches the requested patient identifier.
**Validates: Requirements 8.1**

**Property 21: Audit event completeness**
*For any* audit event returned in an audit trail, the event should include action type, performing user, timestamp, event metadata, block hash, and verification status.
**Validates: Requirements 8.2, 8.4**

**Property 22: Blockchain persistence and recovery**
*For any* blockchain state with N blocks, after persisting to database and reloading on system restart, the in-memory chain should contain exactly N blocks with identical data.
**Validates: Requirements 10.1, 10.3**

### Role-Based Access Control Properties

**Property 23: Doctor permissions**
*For any* medical action (create, update, view), when performed by a user with DOCTOR role, the system should allow the action to proceed.
**Validates: Requirements 4.1**

**Property 24: Patient read-only permissions**
*For any* read operation on a patient's own records, when performed by a user with PATIENT role matching the patient ID, the system should allow the operation; for any write operation, the system should deny it.
**Validates: Requirements 4.2**

**Property 25: Unauthorized action denial**
*For any* medical action where the user lacks proper permissions (patient trying to write, or patient accessing another patient's records), the system should deny the action and return an authorization error.
**Validates: Requirements 3.4, 4.3, 8.5**

**Property 26: Role inclusion in audit events**
*For any* audit event in the blockchain, the event data should include the role of the user who performed the action.
**Validates: Requirements 4.4**

**Property 27: Blockchain verification status in responses**
*For any* medical record retrieval response, the response should include a blockchain verification status indicating whether the chain is currently valid.
**Validates: Requirements 3.5**

## Error Handling

### Exception Hierarchy

```java
// Base exception for medical record operations
public class MedicalRecordException extends RuntimeException {
    public MedicalRecordException(String message) {
        super(message);
    }
}

// Specific exceptions
public class RecordNotFoundException extends MedicalRecordException {
    public RecordNotFoundException(Long recordId) {
        super("Medical record not found: " + recordId);
    }
}

public class PatientNotFoundException extends MedicalRecordException {
    public PatientNotFoundException(String patientId) {
        super("Patient not found: " + patientId);
    }
}

public class ValidationException extends MedicalRecordException {
    private final Map<String, String> fieldErrors;
    
    public ValidationException(Map<String, String> fieldErrors) {
        super("Validation failed");
        this.fieldErrors = fieldErrors;
    }
    
    public Map<String, String> getFieldErrors() {
        return fieldErrors;
    }
}

public class UnauthorizedException extends MedicalRecordException {
    public UnauthorizedException(String message) {
        super("Unauthorized: " + message);
    }
}

public class BlockchainIntegrityException extends MedicalRecordException {
    private final int failedBlockIndex;
    
    public BlockchainIntegrityException(int blockIndex, String reason) {
        super("Blockchain integrity violation at block " + blockIndex + ": " + reason);
        this.failedBlockIndex = blockIndex;
    }
    
    public int getFailedBlockIndex() {
        return failedBlockIndex;
    }
}
```

### Error Response Format

All API errors return a consistent JSON structure:

```json
{
    "timestamp": "2024-01-15T10:30:00Z",
    "status": 400,
    "error": "Bad Request",
    "message": "Validation failed",
    "path": "/api/medical-records",
    "details": {
        "diagnosis": "Diagnosis is required",
        "patientId": "Patient not found: P12345"
    }
}
```

### Error Handling Strategy

1. **Validation Errors (400 Bad Request)**
   - Missing required fields
   - Invalid data formats
   - Length constraint violations
   - Invalid patient references

2. **Authorization Errors (403 Forbidden)**
   - Insufficient permissions for action
   - Patient accessing another patient's records
   - Patient attempting write operations

3. **Not Found Errors (404 Not Found)**
   - Medical record does not exist
   - Patient does not exist

4. **Blockchain Integrity Errors (500 Internal Server Error)**
   - Chain verification failures
   - Tamper detection
   - Persistence failures

5. **Transaction Rollback**
   - If blockchain audit logging fails, rollback the medical record operation
   - Maintain consistency between medical records and audit trail
   - Log errors for investigation

### Validation Rules

**Medical Record Validation:**
- `patientId`: Required, must exist in patients table
- `diagnosis`: Required, max 1000 characters, not blank
- `treatmentPlan`: Required, max 2000 characters, not blank
- `medications`: Optional, max 2000 characters
- `clinicalNotes`: Optional, no length limit
- `doctorId`: Required, not blank
- Timestamps: Auto-generated, must be valid dates

**Authorization Validation:**
- User ID header: Required for all requests
- User role header: Required, must be DOCTOR or PATIENT
- For PATIENT role: Can only access own records (userId == patientId)
- For DOCTOR role: Can access all records

**Blockchain Validation:**
- Genesis block: Must exist on system initialization
- Block hash: Must start with "00" (difficulty 2)
- Previous hash: Must match previous block's hash
- Block index: Must be sequential (no gaps)
- Timestamp: Must be positive, reasonable value

## Testing Strategy

### Dual Testing Approach

This feature requires both **unit tests** and **property-based tests** for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs using randomized test data

Both testing approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing Configuration

**Framework**: Use **jqwik** (Java property-based testing library)

**Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `@Tag("Feature: medical-record-blockchain, Property {number}: {property_text}")`

**Example Property Test Structure**:

```java
@Property
@Tag("Feature: medical-record-blockchain, Property 1: Medical record creation persistence")
void medicalRecordCreationPersistence(
    @ForAll @AlphaChars @StringLength(min = 5, max = 20) String patientId,
    @ForAll @StringLength(min = 10, max = 1000) String diagnosis,
    @ForAll @StringLength(min = 10, max = 2000) String treatmentPlan
) {
    // Arrange: Create valid medical record data
    CreateMedicalRecordRequest request = new CreateMedicalRecordRequest();
    request.setPatientId(patientId);
    request.setDiagnosis(diagnosis);
    request.setTreatmentPlan(treatmentPlan);
    
    // Act: Create record
    MedicalRecordDTO result = medicalRecordService.createRecord(request, "doctor123");
    
    // Assert: Record is persisted with valid ID
    assertThat(result.getId()).isNotNull();
    assertThat(result.getId()).isGreaterThan(0L);
    
    // Verify it can be retrieved
    MedicalRecordDTO retrieved = medicalRecordService.getRecordById(
        result.getId(), "doctor123", UserRole.DOCTOR
    );
    assertThat(retrieved).isNotNull();
}
```

### Unit Testing Strategy

**Focus Areas for Unit Tests**:
1. **Specific examples**: Concrete test cases demonstrating correct behavior
2. **Edge cases**: Empty databases, genesis block creation, non-existent records
3. **Error conditions**: Invalid inputs, authorization failures, validation errors
4. **Integration points**: Controller-Service-Repository interactions

**Avoid**: Writing too many unit tests for scenarios that property tests already cover (e.g., testing many variations of valid inputs)

### Test Coverage Goals

**Medical Record Management**:
- Unit tests: 15-20 tests covering CRUD operations, validation, and error cases
- Property tests: 9 tests (Properties 1-9) covering record operations

**Blockchain Audit**:
- Unit tests: 10-15 tests covering specific blockchain scenarios and edge cases
- Property tests: 13 tests (Properties 10-22) covering audit logging and verification

**Access Control**:
- Unit tests: 8-10 tests covering specific authorization scenarios
- Property tests: 5 tests (Properties 23-27) covering role-based permissions

### Integration Testing

**Blockchain Integration Tests**:
1. End-to-end flow: Create record → Verify audit event → Verify chain
2. Tamper detection: Modify database directly → Verify detection
3. Persistence recovery: Add blocks → Restart system → Verify reload
4. Concurrent operations: Multiple threads adding records → Verify chain integrity

**API Integration Tests**:
1. Full request-response cycles through controllers
2. Authentication and authorization flows
3. Error response format validation
4. CORS and cross-origin request handling

### Test Data Generation

**For Property Tests**:
- Use jqwik generators for random strings, numbers, dates
- Create custom generators for domain objects (MedicalRecord, Patient)
- Ensure generated data respects constraints (length limits, valid formats)

**For Unit Tests**:
- Use test fixtures for common scenarios
- Builder pattern for test data creation
- Separate test data from test logic

### Mocking Strategy

**Mock External Dependencies**:
- Database repositories (for service layer tests)
- AuditService (for controller tests)
- Authentication/authorization services

**Do NOT Mock**:
- SimpleBlockchain (test actual blockchain logic)
- Domain models and DTOs
- Validation logic

### Test Execution

**Local Development**:
```bash
mvn test                    # Run all tests
mvn test -Dtest=*Property*  # Run only property tests
mvn test -Dtest=*Unit*      # Run only unit tests
```

**CI/CD Pipeline**:
- Run all tests on every commit
- Fail build if any test fails
- Generate coverage reports (target: >80% line coverage)
- Run property tests with increased iterations (500+) on nightly builds

### Performance Testing

**Blockchain Performance**:
- Measure time to add 1000 blocks
- Measure time to verify chain with 1000 blocks
- Measure database persistence overhead
- Target: <100ms per block addition, <1s for chain verification

**API Performance**:
- Measure response times for CRUD operations
- Test with concurrent requests (10, 50, 100 users)
- Target: <200ms for record creation, <100ms for retrieval
