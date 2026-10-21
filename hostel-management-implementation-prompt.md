# Implementation Prompt: Hostel Management System — Spring Boot Backend

## Project Context
Build the backend for a **Hostel Management System** for a university hostel (DIT-style academic project). Students apply for accommodation each academic term, get allocated a specific bed in a room, receive invoices for their term fee, make payments, and can log maintenance requests. Staff (via `SYSTEM_USER`/`ROLE`) manage terms, rooms, applications, allocations, invoices, payments, and maintenance.

## Tech Stack
- Java 21
- Spring Boot (latest stable 3.x/4.x)
- Spring Data JPA + PostgreSQL
- Spring Security 7.x with JWT authentication
- Lombok
- Maven
- Bean Validation (`jakarta.validation`)

## Package Structure
Use exactly this layout under `com.university.hostel`:

```
com.university.hostel
├── HostelApplication.java
├── entity/          (JPA @Entity classes)
├── repository/       (Spring Data JpaRepository interfaces, suffix *Repository)
├── service/           (business logic, suffix *Service, interface + impl or just impl class)
├── controller/         (REST controllers, suffix *Controller)
├── dto/                 (request/response DTOs — add this package)
├── security/             (JWT filter, JwtService, SecurityConfig — add this package)
├── exception/             (custom exceptions + GlobalExceptionHandler — add this package)
└── config/                 (any beans/config — add this package)
```

## Entities (from ERD)

### AcademicTerm
`term_id (PK)`, `term_name`, `academic_year`, `start_date`, `end_date`, `is_current`

### RoomType
`room_type_id (PK)`, `type_name`, `bed_capacity`, `base_fee_per_term`, `description`

### Room
`room_id (PK)`, `room_number`, `building_block`, `floor_number`, `room_type_id (FK → RoomType)`, `room_status`, `is_active`

### Bed
`bed_id (PK)`, `room_id (FK → Room)`, `bed_label`, `bed_status`

### Student
`student_id (PK)`, `full_name`, `student_reg_number`, `phone_number`, `email_address`, `gender`, `national_id_number`, `date_of_birth`, `program_of_study`, `guardian_name`, `guardian_phone`, `is_active`

### Application
`application_id (PK)`, `student_id (FK → Student)`, `term_id (FK → AcademicTerm)`, `preferred_room_type_id (FK → RoomType)`, `application_date`, `application_status`, `reviewed_by_user_id (FK → SystemUser, nullable)`, `review_date (nullable)`

### Allocation
`allocation_id (PK)`, `application_id (FK → Application)`, `student_id (FK → Student)`, `bed_id (FK → Bed)`, `term_id (FK → AcademicTerm)`, `checkin_date`, `checkout_date (nullable)`, `allocation_status`, `fee_amount`, `created_by_user_id (FK → SystemUser)`

### MaintenanceRequest
`request_id (PK)`, `room_id (FK → Room)`, `reported_by_student_id (FK → Student)`, `description`, `priority`, `status`, `logged_date`, `assigned_user_id (FK → SystemUser, nullable)`, `resolved_date (nullable)`

### Invoice
`invoice_id (PK)`, `allocation_id (FK → Allocation)`, `issue_date`, `due_date`, `amount_due`, `amount_paid`, `balance`, `invoice_status`

### Payment
`payment_id (PK)`, `invoice_id (FK → Invoice)`, `amount_paid`, `payment_method`, `transaction_reference`, `payment_date`, `recorded_by_user_id (FK → SystemUser)`, `payment_status`

### SystemUser
`user_id (PK)`, `username`, `password_hash`, `role_id (FK → Role)`, `full_name`, `email_address`, `is_active`

### Role
`role_id (PK)`, `role_name`, `permission_set`

### Notification *(new — not in the original ERD, add it)*
`notification_id (PK)`, `recipient_student_id (FK → Student, nullable)`, `recipient_user_id (FK → SystemUser, nullable)`, `title`, `message`, `notification_type` (e.g. APPLICATION_STATUS, INVOICE_DUE, MAINTENANCE_UPDATE), `is_read`, `created_date`

## Key Business Rules
1. **Allocation creation**: when an Allocation is created, set the target `Bed.bed_status = OCCUPIED` and, if it's the last free bed in the room, set `Room.room_status = FULL`. When an allocation ends (checkout), reverse this.
2. **Invoice.balance**: treat as a **derived/computed value** (`amount_due - amount_paid`), recalculated whenever a Payment is recorded — do not let it drift out of sync; recompute in the service layer rather than trusting client input.
3. **Application → Allocation**: an Application only produces an Allocation once `application_status = APPROVED`; enforce this in `AllocationService` (reject creating an Allocation for a non-approved/already-allocated Application).
4. **Payment → Invoice sync**: recording a Payment must update the parent Invoice's `amount_paid`, `balance`, and `invoice_status` (e.g. PARTIAL, PAID) atomically (same transaction).
5. **Role-based access**: implement three roles for MVP — `ADMIN`, `HOSTEL_STAFF`, `STUDENT`. `Role.permission_set` can store a comma-separated list of fine-grained permissions for later use, but authorization checks in this phase should be role-based (`@PreAuthorize("hasRole('ADMIN')")` etc.), not permission-string parsing.
6. **Soft status fields**: use enums (not free-text strings) for all `*_status` fields (`room_status`, `bed_status`, `application_status`, `allocation_status`, `invoice_status`, `payment_status`, `priority`, `status` on MaintenanceRequest).

## Deliverables — implement all layers together, entity by entity:
1. **Entities** — JPA `@Entity` classes with correct relationships (`@ManyToOne`, `@OneToMany`, `@JoinColumn`), enums for status fields, `@CreationTimestamp`/`@UpdateTimestamp` where useful, Lombok `@Getter/@Setter/@NoArgsConstructor/@AllArgsConstructor/@Builder`.
2. **Repositories** — `JpaRepository<Entity, Long>` interfaces, with sensible derived query methods (e.g. `findByStudentIdAndTermId`, `findByApplicationStatus`, `findByRoomIdAndBedStatus`).
3. **DTOs** — request and response DTOs per entity (don't expose entities directly in controllers); include validation annotations (`@NotNull`, `@NotBlank`, `@Email`, etc.) on request DTOs.
4. **Services** — implement business logic listed above, throw custom exceptions (`ResourceNotFoundException`, `InvalidStateException`, etc.) for invalid operations.
5. **Controllers** — REST endpoints following standard conventions (`GET/POST/PUT/DELETE /api/v1/<resource>`), proper HTTP status codes, `@Valid` on request bodies.
6. **Security** — JWT auth: `AuthController` (`/api/v1/auth/register`, `/api/v1/auth/login`), `JwtService` (generate/validate tokens), `JwtAuthFilter`, `SecurityConfig` with role-based endpoint protection (students can only see/act on their own records; staff/admin have broader access).
7. **Global exception handling** — `@RestControllerAdvice` mapping custom exceptions to appropriate HTTP responses with a consistent error body shape.
8. **Seed data** — a `CommandLineRunner`/seeder that creates the `ADMIN`/`HOSTEL_STAFF`/`STUDENT` roles and one default admin account on first run.

## Order of Implementation
Please implement in this order so each layer compiles on top of a working one: `Role` → `SystemUser` (+ auth) → `AcademicTerm` → `RoomType` → `Room` → `Bed` → `Student` → `Application` → `Allocation` → `Invoice` → `Payment` → `MaintenanceRequest` → `Notification`.

## Notes / Assumptions Made in This Prompt (flag if any are wrong)
- Added a `dto`, `security`, `exception`, `config` package not in the original structure — standard Spring Boot layering.
- Added the `Notification` entity (it was in the package plan but missing from the ERD) with a reasonable minimal shape.
- Assumed `Invoice.balance` is computed, not client-supplied.
- Assumed a 3-role MVP (`ADMIN`, `HOSTEL_STAFF`, `STUDENT`) rather than fully dynamic permission parsing.
- Assumed PostgreSQL + JWT auth to match the stack used on the existing Gym Management System project, for consistency across your projects.
