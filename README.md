# User-storiese-example# User-storiese-example
## Auth & Security (Auth Module)

### 1) User Registration (Email + Role + Invite Codes)
As a user (patient/doctor/nurse), I want to sign up with email and role so that I can access role-specific features and honor invite-only onboarding.
- Acceptance Criteria:
  - POST `/auth/signup` accepts `email`, `password`, `role`, optional `inviteCode` tied to tenant/clinic.
  - Validates uniqueness of `email`; rejects expired/incorrect invite codes with `410`.
  - Hashes password and stores securely; audit log records inviter and timestamp.
  - Returns `201` with created user summary (no password).

### 2) User Login with JWT + Device Fingerprinting
As a user, I want to log in so that I can receive a token for authenticated actions while tracking trusted devices.
- Acceptance Criteria:
  - POST `/auth/login` accepts `email`, `password`, captures device fingerprint (agent + IP).
  - Validates credentials; returns `200` with access + refresh JWT pair.
  - Uses `JwtStrategy` + `JwtAuthGuard`; stores refresh token hash per device for revocation.

### 3) Password Reset (Request + Confirm + MFA Fallback)
As a user, I want to reset my password so that I can regain access if I forget it even when MFA is enforced.
- Acceptance Criteria:
  - POST `/auth/password/request` accepts `email`; generates time-limited token and invalidates old tokens.
  - POST `/auth/password/confirm` accepts `token`, `newPassword`, optional backup code when MFA is enabled.
  - Invalid or expired token returns `400`; success logs security event and revokes active sessions.

### 4) Role-Based Access Control (RBAC + Scoped Actions)
As an admin, I want role-based permissions enforced so that users only access authorized resources with clear action scopes.
- Acceptance Criteria:
  - Guards enforce routes using role + action matrix maintained in config.
  - Unauthorized access returns `403` and records attempted action for audit.
  - Documentation lists public vs protected routes and required scopes per module.

## User Management (Patients/Doctors/Nurses Modules)

### 5) Create Patient Profile (Demographics + Emergency Contacts)
As a patient, I want a profile so that my personal and medical data is stored along with emergency contacts.
- Acceptance Criteria:
  - POST `/patients` creates profile with demographics, contact info, emergency contact block.
  - Validates required fields (name, DOB, contact); duplicates detected via email/phone.
  - Returns `201` with profile body including profile completeness flag.

### 6) Update Patient Profile (Preferences + Insurance)
As a patient, I want to update my profile so that my information stays current, including communication preferences and insurance data.
- Acceptance Criteria:
  - PATCH `/patients/:id` updates editable fields plus nested `preferences` and `insurance` objects.
  - Ownership or admin role required; change history tracked with timestamp + actor.
  - Returns `200` with updated profile.

### 7) Add Patient Notes (Clinician + Attachments)
As a clinician, I want to add notes to a patient record so that clinical history is maintained along with optional attachments.
- Acceptance Criteria:
  - POST `/patients/:id/notes` accepts note DTO plus optional `attachmentIds` from uploads.
  - Role `doctor|nurse` required; audit trail recorded with note type.
  - Returns `201` with note metadata including attachment references.

### 8) Create Doctor Profile (Specialty + Availability Snapshot)
As a doctor, I want a professional profile so that patients can find and book with me and see my default availability window.
- Acceptance Criteria:
  - POST `/doctors` creates profile with specialty, qualifications, and default weekly availability skeleton.
  - Returns `201` with profile and flag if credential review pending.

### 9) Create Nurse Profile (Skills + Shift Preferences)
As a nurse, I want a professional profile so that I can be assigned to patient care with my skills noted.
- Acceptance Criteria:
  - POST `/nurses` records certifications, languages, and shift preference tags.
  - Returns `201` with profile data consumable by scheduling engine.

## Appointments (Appointments Module)

### 10) Book Appointment (Clinic + Telehealth Support)
As a patient, I want to book an appointment so that I can see a clinician at a specific time either onsite or via telehealth.
- Acceptance Criteria:
  - POST `/appointments` with patientId, doctorId, serviceId, locationId or `telehealth=true`, and timeslot.
  - Validates availability; prevents double booking; reserves telehealth room token when needed.
  - Returns `201` with appointment details and channel (in-person/telehealth).

### 11) Reschedule/Cancel Appointment (Notifications)
As a patient, I want to reschedule or cancel an appointment so that I can manage my schedule with automatic notifications.
- Acceptance Criteria:
  - PATCH `/appointments/:id` to reschedule; checks conflicts and emits confirmation notification.
  - DELETE `/appointments/:id` to cancel; updates status and notifies clinician/care team.
  - Emits audit events with reason codes.

### 12) List Upcoming Appointments (Filters + Export)
As a user, I want to list my upcoming appointments so that I can plan ahead and export them to my calendar.
- Acceptance Criteria:
  - GET `/appointments?userId=...&role=...&range=upcoming` supports pagination, date filters, and status filters.
  - Optional query `format=ics` returns calendar export.
  - Returns `200` with list.

## Services & Locations (Services + Locations Modules)

### 13) Manage Services Catalog (Pricing + Duration)
As an admin, I want to manage medical services so that offerings are standardized with price and duration metadata.
- Acceptance Criteria:
  - CRUD on `/services` (POST, GET, PATCH, DELETE) with validation of code uniqueness and default duration.
  - Updates propagate to appointment booking validation.
  - Returns appropriate status codes with audit trail on mutations.

### 14) Manage Locations (Rooms + Capacity)
As an admin, I want to manage clinic locations so that appointments can be scheduled properly with room capacity constraints.
- Acceptance Criteria:
  - CRUD on `/locations` capturing address, timezone, and room capacity list.
  - Associates services/providers to locations and enforces max simultaneous bookings per room.
  - Returns appropriate status codes.

## Assets & Uploads (Assets + Uploads Modules)

### 15) Upload Medical Documents (Virus Scan + Tagging)
As a user, I want to upload medical documents so that my records are available to clinicians safely.
- Acceptance Criteria:
  - POST `/uploads` supports multipart file upload, virus scan status, and document tagging (lab, imaging, other).
  - Validates file type/size; stores metadata including uploaderId.
  - Returns `201` with asset ID and signed download hint.

### 16) Secure Asset Retrieval (Time-Limited Links + Audit)
As an authorized user, I want secure access to my assets so that my data is protected.
- Acceptance Criteria:
  - GET `/assets/:id` enforces ownership/role and consent checks.
  - Returns short-lived presigned URL; logs access event with viewer role.
  - Unauthorized returns `403` and increments security metric.


## Analytics & Ops (Analytics Module)

### 19) Admin KPIs Dashboard API (Trendlines)
As an admin, I want key metrics so that I can monitor platform performance with trendlines.
- Acceptance Criteria:
  - GET `/analytics/kpis` returns metrics: new users, bookings per day, active subscriptions, each with 7-day trendline data.
  - Filters by date range; returns `200` with cache headers.

### 20) Operational Audit Feed (Export + Filters)
As compliance, I want an operational audit feed so that I can export filtered events without relying on AI features.
- Acceptance Criteria:
  - GET `/analytics/audit-feed` streams audit events filtered by module, actor, date range.
  - Supports CSV export for regulators; enforces admin-only guard.
  - Pagination cursor prevents gaps; export metadata logged.


## AI Assistant Module (MCP + API Knowledge Base, No RAG)

### 1) Postgres Read-Only MCP Server (Clinic Data Context)
As a doctor, I want an MCP server that can read clinic data from Postgres so that AI answers use authoritative patient and clinic records without write access.
- Acceptance Criteria:
  - MCP server connects to Postgres with a **read-only role**; exports tools for patient records, appointments, medical notes, services, and locations.
  - MCP runs on separate port (`3001` default), requires bearer JWT auth, enforces RBAC on doctor/patient scope; rate-limit 50 req/min per doctor.
  - All queries are logged with `doctorId`, `patientId`, timestamp, and query type for audit; no INSERT/UPDATE/DELETE permitted.
  - Returns `200` with structured JSON context objects defined in API reference; errors return `403` on forbidden tables and `429` on rate limit.

### 2) API Medical Knowledge Base Integration (No RAG)
As a doctor, I want AI replies to include trusted medical references from an external API so that I can check evidence-based guidance.
- Acceptance Criteria:
  - Integrate a medical KB API (e.g., **MedlinePlus REST**) via backend service, **without vector search/RAG**; query by condition, drug, or procedure code.
  - POST `/ai-assistant/ask` accepts `doctorId`, `patientId`, `question`, optional `conversationId`; pipeline fetches Postgres context (story 1) + KB API facts.
  - Response includes `answer`, `kbReferences` (title, URL, snippet, source API), `disclaimer`, `latencyMs`.
  - Example: Query "type 2 diabetes foot care" returns MedlinePlus article link and summary paragraph; returns `201` with combined answer.
  - Timeouts handled in 5s per KB call; fallback returns partial context with `kbReferences=[]` and warning.

### 3) Multi-Patient Thread Management (One Conversation, Many Patients/Agents)
As a doctor, I want to manage threads that include multiple patients and AI agents so that I can discuss several cases in one conversation.
- Acceptance Criteria:
  - Conversations support `participantPatientIds` array; doctor can add/remove patients via PATCH `/ai-assistant/conversations/:id`.
  - Each message stores `patientContextIds` to show which patients the question refers to; validation ensures doctor is assigned to those patients.
  - GET `/ai-assistant/conversations/:id` returns messages grouped by patient context with timestamps and agent used.
  - POST `/ai-assistant/ask` accepts `patientIds` (array) to address multiple patients; response includes per-patient `answers[]` with sources.
  - Export preserves multi-patient context; access control enforces doctor’s scope per patient.

### 4) Like/Dislike Feedback on Agent Replies (Quality Loop)
As a doctor, I want to like or dislike AI answers so that low-quality responses get flagged and high-quality ones are reinforced.
- Acceptance Criteria:
  - POST `/ai-assistant/feedback` accepts `messageId`, `vote` (`like|dislike`), optional `comment`; idempotent per doctor per message.
  - Stores feedback per message; aggregates satisfaction score per agent/model and per question type.
  - GET `/ai-assistant/feedback-analytics` (admin-only) returns: like %, dislike %, top flagged intents, and recent flagged answers.
  - Disliked answers trigger alert log entry; liked answers marked for prompt reuse; returns `201` for feedback and `200` for analytics.


---

## Invoices Module (Complete CRUD + Payments)

### 6) Create Invoice (Patient/Service Charges)
As a clinic admin, I want to create invoices so that patients are billed for services rendered.
- Acceptance Criteria:
  - POST `/invoices` accepts `patientId`, `createdBy` (admin/doctor), array of `lineItems` (serviceId + quantity + customAmount), optional `discountPercent`, `taxPercent`.
  - Calculates subtotal, tax, discount, total automatically; validates serviceIds and patientId exist.
  - Generates unique invoice number with clinic prefix (e.g., `CLINIC-001-2026-001`).
  - Sets initial status: `draft` (unpublished), returns `201` with invoice summary.
  - Audit logs creator and timestamp.

### 7) Publish Invoice (Send to Patient)
As a clinic admin, I want to publish and send invoices so that patients receive them for payment.
- Acceptance Criteria:
  - PATCH `/invoices/:id` with `status=published` marks invoice immutable and triggers notification to patient.
  - Generates PDF invoice using template (clinic header, line items, totals, due date, payment instructions).
  - Notification includes: invoice PDF link, due date (default 30 days), payment methods available, clinic contact.
  - Updates `publishedAt` timestamp; status transitions logged.
  - Returns `200` with updated invoice; attempting to edit published invoice returns `409`.

### 8) Track Invoice Payments (Payment Recording)
As a clinic admin, I want to record payments against invoices so that I can track which ones are paid.
- Acceptance Criteria:
  - POST `/invoices/:id/payments` accepts `amount`, `paymentMethod` (cash, card, transfer, check), optional `referenceNumber`.
  - Validates amount ≤ outstanding balance; updates `paidAmount`, `remainingBalance`, `paymentHistory` array.
  - Records `paidAt`, `recordedBy` actor; if remaining balance = 0, auto-transitions status to `paid`.
  - Emits `invoice.paid` event if fully paid; sends confirmation to patient.
  - Returns `201` with payment record.

### 8.5) Invoice Promo Code Application (Per-Invoice Discount)
As a clinic admin, I want to apply a promo code on an invoice so that eligible patients get discounted pricing.
- Acceptance Criteria:
  - Invoice entity includes `promoCode` (string, case-sensitive), `promoDiscountPercent`, `promoAppliedAt`, `promoId` reference.
  - POST `/invoices/:id/apply-promo` accepts `promoCode`; validates promo is **active**, **case-sensitive exact match**, **not expired**, and **remaining usages > 0`.
  - Calculates discount = `subtotal * promoDiscountPercent`; recalculates `total` after discount and tax; stores applied promo snapshot.
  - If promo invalid/expired/maxed: returns `400` with reason; does not mutate invoice.
  - Applying a promo decrements `remainingUsages`; audit logs actor/time and promo code used.
  - GET `/invoices/:id` returns applied promo details in the response payload.

### 9) Automatic Invoice Reminders (Scheduled Notifications)
As the system, I want to auto-send payment reminders so that unpaid invoices are followed up without manual effort.
- Acceptance Criteria:
  - Background job runs daily; finds invoices with status `published`, `dueDate` within range: upcoming (3 days before), overdue (1+ days past).
  - Sends `upcoming` reminder 3 days before due date; sends `overdue` reminder every 7 days after.
  - Reminders include invoice number, amount, due date, payment link.
  - Config endpoint: GET/PUT `/invoices/reminders-config` (admin-only) to adjust thresholds.
  - Max 2 reminders per invoice per cycle; logs all sent reminders.
  - Returns `200` with config or reminder statistics.

### 10) Invoice Reconciliation Report (Admin Dashboard)
As a clinic admin, I want a reconciliation report so that I can verify income vs. invoiced amounts.
- Acceptance Criteria:
  - GET `/invoices/reconciliation?month=YYYY-MM` returns:
    - Total invoiced amount
    - Total collected amount
    - Outstanding amount (by patient)
    - Overdue count + total overdue value
    - Payment method breakdown
  - Supports date range filters and clinic filter (for multi-clinic setups).
  - Exportable as CSV for accounting team.
  - Returns `200` with structured report.

---
## Prescriptions Module (Doctor-Issued Medical Prescriptions)

### 11) Create & Issue Prescription (Doctor → Patient)
As a doctor, I want to issue prescriptions to patients so that they have documented medication guidance for pharmacy dispensing.
- Acceptance Criteria:
  - POST `/prescriptions` accepts `patientId`, `medications` array with: `drugName`, `dosage`, `frequency`, `duration` (days), optional `instructions`, `refills` (0-5).
  - Validates patientId exists and doctor has access; generates unique prescription ID with timestamp.
  - Optional `appointmentId` links to visit; auto-calculates `expiresAt` = now + 180 days (pharmacy standard).
  - Sets status: `issued`, `issuedAt`, `issuedBy` actor (doctor).
  - Emits `prescription.issued` event; sends patient notification with prescription summary.
  - Returns `201` with prescription object including QR code for pharmacy scanning.

### 12) View & Download Prescriptions (Patient + Pharmacy Access)
As a patient, I want to view my prescriptions so that I can take them to a pharmacy or request refills.
- Acceptance Criteria:
  - GET `/prescriptions?patientId=...&status=active|filled|expired` returns paginated list with filters.
  - GET `/prescriptions/:id` returns full prescription details including doctor info, medications, issued date.
  - POST `/prescriptions/:id/download` generates PDF with prescription header, medications, instructions, doctor signature placeholder.
  - POST `/prescriptions/:id/share` generates secure token-based link (valid 24h) to share with pharmacy; audit logs recipient.
  - Access control: patient sees own prescriptions; pharmacy staff can view if prescription shared; doctor sees issued prescriptions.
  - Returns `200` with appropriate payload; `403` for unauthorized access.

### 13) Prescription Status Tracking (Issued → Filled → Completed)
As a patient, I want to track my prescription status so that I know if it's been filled at the pharmacy.
- Acceptance Criteria:
  - Prescription lifecycle: `issued` → `filled` (optional pharmacy webhook or manual update) → `completed` (when patient confirms use).
  - PATCH `/prescriptions/:id` updates `status`, captures `filledAt` timestamp, optional `pharmacyName`, `filledBy` (pharmacist).
  - GET `/prescriptions/:id/history` returns full audit trail: issued → filled → completed with timestamps + actors.
  - Status `expired` auto-set if `expiresAt` < today; cannot be filled after expiry.
  - Emits events per state transition; notifies patient of status changes.
  - Returns `200` with updated prescription; `410` if attempting to fill expired prescription.

### 14) Prescription Refill Requests (Patient → Doctor Approval)
As a patient, I want to request refills of my prescriptions so that I can continue medication without new doctor visit.
- Acceptance Criteria:
  - POST `/prescriptions/:id/refill-request` accepts optional `reason`, submits refill request if `refills` count > 0.
  - Sets `refillRequestStatus=pending`; notifies prescribing doctor with request details.
  - Doctor review: PATCH `/prescriptions/:id/refill-request` with `approved=true|false`, optional `notes`.
  - On approval: increments `filledCount`, decrements `refills`; generates new prescription ID (child of parent); emits `prescription.refill_approved`.
  - On rejection: notifies patient with doctor's reason; `refillRequestStatus=denied`.
  - Refill requests expire after 30 days if not reviewed; returns `200` with request status; `409` if no refills remaining.

### 15) Nurse Prescription Administration (In-Clinic Medicine Dispensing)
As a nurse, I want to receive prescriptions and administer medicines to patients so that I can track in-clinic medication delivery.
- Acceptance Criteria:
  - GET `/prescriptions?status=issued|ready_for_administration&nurseId=...` returns list of prescriptions assigned to nurse, filtered by clinic location.
  - Nurse marks prescription `status=ready_for_administration` after doctor approves; can view via WebSocket real-time updates.
  - POST `/prescriptions/:id/administer` accepts `administeredAt`, `batchNumber` (medicine lot), `dosageGiven`, `routeOfAdministration` (or
