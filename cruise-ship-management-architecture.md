# Cruise Ship Management Platform
## Software Architecture & Foundation Document — Phase 1

**Stack:** Laravel 12 · PHP 8.4 · MySQL 8 · Tailwind CSS · Livewire 3 · Alpine.js · Laravel Sanctum · Laravel Queues · Laravel Notifications · OpenAI API

**Document status:** Foundation / pre-development planning. No code has been written yet — this document is the contract the team builds against.

---

## 0. Planning Notes & Assumptions

Before the architecture, a few decisions that shape everything downstream:

| Decision | Choice | Why |
|---|---|---|
| Tenancy model | Single operator, multiple ships | You're building *a* cruise line's platform, not a marketplace of cruise lines. If that's wrong, flag it now — multi-tenancy changes the DB design (tenant_id everywhere) and is much cheaper to add at the start than later. |
| Starter kit | **Laravel Livewire starter kit** (not Breeze/Jetstream) | Laravel 12 retired Breeze/Jetstream in favor of new starter kits. The Livewire kit ships with Flux UI + Laravel Volt. Flux UI is optional — you can strip it and use plain Tailwind if you want full visual control, which matters since this is a customer-facing booking site, not an admin-only tool. |
| Payment gateway | Stripe (primary), pluggable gateway interface | Keeps Hubtel/Paystack/local gateways addable later via the same interface pattern you're already using on the voucher portal. |
| AI approach | RAG (Retrieval-Augmented Generation), not fine-tuning | Cruise inventory changes daily. Fine-tuning a model on it would be stale within a week. RAG pulls live data at query time — covered in detail in Section 6. |
| Admin UI | Livewire 3 + Alpine.js, server-rendered | No separate SPA needed for admin. Customer-facing site can share the same stack; API/Sanctum exists for future mobile app or third-party integrations. |

If any of these assumptions are wrong, say so before Phase 1 build starts — the DB schema in Section 2 already encodes some of these choices (e.g., no `tenant_id` columns).

---

## 1. System Architecture

### 1.1 High-Level Component Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                │
│                                                                            │
│   ┌─────────────────────┐        ┌─────────────────────────┐            │
│   │  Customer Web App    │        │   Admin Dashboard        │            │
│   │  (Livewire 3 + Alpine)│        │   (Livewire 3 + Alpine)  │            │
│   │  - Browse ships       │        │   - Ships/Cabins/Venues  │            │
│   │  - Book cruises       │        │   - Bookings/Customers   │            │
│   │  - AI Chat widget     │        │   - Reports/Analytics    │            │
│   └──────────┬────────────┘        └────────────┬─────────────┘            │
│              │                                   │                          │
│   ┌──────────▼───────────────────────────────────▼─────────┐               │
│   │           Mobile / 3rd-Party Clients (future)           │               │
│   │           via REST API + Sanctum tokens                 │               │
│   └──────────────────────────┬────────────────────────────┘               │
└──────────────────────────────┼────────────────────────────────────────────┘
                               │  HTTPS
┌──────────────────────────────▼────────────────────────────────────────────┐
│                         LARAVEL APPLICATION LAYER                         │
│                                                                            │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │   Web       │  │  API Routes  │  │  Livewire     │  │  Admin Routes   │ │
│  │   Routes    │  │  (Sanctum)   │  │  Components   │  │  (RBAC guarded) │ │
│  └─────┬──────┘  └──────┬───────┘  └──────┬───────┘  └────────┬────────┘ │
│        └────────────────┴──────────────────┴───────────────────┘          │
│                                  │                                         │
│        ┌─────────────────────────▼─────────────────────────┐              │
│        │              Application Service Layer             │              │
│        │  CruiseService · BookingService · VenueService      │              │
│        │  PaymentService · InventoryService · AIService       │              │
│        └─────────────────────────┬─────────────────────────┘              │
│                                  │                                         │
│        ┌─────────────────────────▼─────────────────────────┐              │
│        │            Domain / Eloquent Model Layer            │              │
│        │   (Modules — see Section 3)                          │              │
│        └─────────────────────────┬─────────────────────────┘              │
│                                  │                                         │
│   ┌──────────────┬───────────────┼───────────────┬──────────────────┐     │
│   │   Queues      │  Notifications │   Events/      │   Policies/      │     │
│   │  (Redis/DB)   │   (mail/sms)   │   Listeners     │   Gates (RBAC)    │     │
│   └──────────────┴───────────────┴───────────────┴──────────────────┘     │
└──────────────────────────────┬────────────────────────────────────────────┘
                               │
        ┌──────────────────────┼───────────────────────┬─────────────────────┐
        │                      │                       │                     │
┌───────▼────────┐   ┌─────────▼──────────┐   ┌─────────▼─────────┐ ┌────────▼────────┐
│   MySQL 8       │   │  AI Services        │   │  Payment Gateway   │ │  Notification    │
│   Database      │   │  - OpenAI API        │   │  - Stripe (primary)│ │  Services         │
│                 │   │  - RAG context       │   │  - Webhooks        │ │  - Mail (SMTP/SES)│
│  (Section 2)    │   │    builder            │   │  - Pluggable       │ │  - SMS (Twilio)   │
│                 │   │  - Embeddings store   │   │    gateway interface│ │  - Push (future)  │
└─────────────────┘   └─────────────────────┘   └────────────────────┘ └──────────────────┘
        │
┌───────▼────────┐
│  File Storage   │
│  S3 / local      │
│  (ship images,   │
│  deck plans,     │
│  receipts)        │
└─────────────────┘
```

### 1.2 Request Flow Examples

**Customer books a cabin:**
`Livewire BookingWizard component → BookingService::createBooking() → DB transaction (bookings, passengers, cabins.status update) → PaymentService::initiateCharge() → Stripe → webhook confirms → BookingConfirmed event → queued job sends email/SMS via Notification → AI conversation context refreshed if customer has an active chat`

**Customer asks AI Assistant a question:**
`Livewire ChatWidget → AIService::handleMessage() → builds context (structured DB query, NOT full DB dump) → sends to OpenAI Chat Completions API with system prompt + retrieved context → response stored in ai_messages → streamed back to widget`

**Admin views occupancy report:**
`Admin Livewire dashboard component → ReportingService → cached aggregate query (or pre-computed via scheduled job) → Chart rendered with Alpine/Chart.js`

### 1.3 Why this shape

- **Service layer between Livewire/Controllers and Eloquent models** — keeps business logic (e.g. "can this cabin be booked for these dates") out of components and controllers so it's testable and reusable between the web UI, the API, and the AI assistant's tool-calling layer.
- **Queues for anything slow or external** — payment confirmation emails, AI embedding refresh jobs, report pre-computation. Keeps the request/response cycle fast.
- **Sanctum from day one** even though Phase 1 is web-only — it costs nothing now and means a mobile app or partner integration later doesn't require retrofitting auth.

---

## 2. Database Design

Conventions used throughout: `id` = unsigned bigint PK, `created_at`/`updated_at` on every table, soft deletes (`deleted_at`) on customer-facing and financial records, snake_case naming, money stored as integers in minor units (pesewas/cents) to avoid float rounding errors, currency stored alongside amounts.

### 2.1 Entity Relationship Overview

```
users ──┬── roles (pivot: role_user)
        │
        ├──< bookings >── cruise_schedules ──< cruises >── cruise_ships
        │         │                                  │
        │         │                                  └──< destinations (pivot: cruise_destination)
        │         ├──< passengers
        │         ├──< payments
        │         ├──< package_bookings >── packages
        │         └──< dining_reservations >── dining_options
        │
        ├──< venue_bookings >── venues ──< events
        │
        └──< ai_conversations >──< ai_messages

cruise_ships ──< ship_decks ──< cabins >── cabin_types
cruise_ships ──< venues
cruise_ships ──< inventory_items (via inventory)
```

### 2.2 Table Definitions

#### `users`
Core auth table for both customers and staff. Staff/admin distinction is via roles, not a separate table — keeps auth logic in one place.

| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(255) | |
| email | varchar(255) | unique, indexed |
| phone | varchar(20) | nullable, indexed |
| password | varchar(255) | hashed |
| user_type | enum('customer','staff') | indexed — quick filter without joining roles |
| email_verified_at | timestamp | nullable |
| two_factor_secret | text | nullable, encrypted |
| last_login_at | timestamp | nullable |
| status | enum('active','suspended','deleted') | default 'active' |
| remember_token | varchar(100) | nullable |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** `email` (unique), `phone`, `user_type`, `status`

---

#### `roles`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(100) | unique — e.g. `super_admin`, `operations_manager`, `cruise_manager`, `venue_manager`, `customer_service`, `finance_officer`, `customer` |
| slug | varchar(100) | unique, indexed |
| description | text | nullable |
| created_at / updated_at | timestamp | |

#### `permissions`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(150) | e.g. `cabins.manage`, `bookings.refund`, `reports.view` |
| slug | varchar(150) | unique, indexed |
| module | varchar(100) | indexed — groups permissions by module for UI rendering |
| created_at / updated_at | timestamp | |

#### `permission_role` (pivot)
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| role_id | bigint unsigned, FK → roles.id, cascade delete | |
| permission_id | bigint unsigned, FK → permissions.id, cascade delete | |

**Indexes:** unique composite (`role_id`, `permission_id`)

#### `role_user` (pivot)
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| user_id | bigint unsigned, FK → users.id, cascade delete | |
| role_id | bigint unsigned, FK → roles.id, cascade delete | |

**Indexes:** unique composite (`user_id`, `role_id`)

> Note: this is a custom lightweight RBAC schema. In build, this maps cleanly onto **spatie/laravel-permission** if you'd rather not hand-roll it — same table shapes, battle-tested package, saves real time. Flagging as a build-phase decision, not a Phase 1 blocker.

---

#### `cruise_ships`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(150) | indexed |
| slug | varchar(160) | unique, indexed |
| description | text | |
| year_built | year | nullable |
| gross_tonnage | int unsigned | nullable |
| total_decks | smallint unsigned | |
| max_passenger_capacity | int unsigned | |
| max_crew_capacity | int unsigned | |
| status | enum('active','maintenance','retired') | default 'active', indexed |
| home_port | varchar(150) | nullable |
| main_image_path | varchar(255) | nullable |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** `slug` (unique), `status`, `name`

#### `ship_decks`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_ship_id | bigint unsigned, FK → cruise_ships.id, cascade delete | indexed |
| deck_number | smallint | |
| name | varchar(100) | e.g. "Deck 7 - Promenade" |
| deck_plan_image_path | varchar(255) | nullable |
| created_at / updated_at | timestamp | |

**Indexes:** composite unique (`cruise_ship_id`, `deck_number`)

#### `cabin_types`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(100) | e.g. "Interior", "Ocean View", "Balcony", "Suite" |
| slug | varchar(110) | unique |
| description | text | |
| base_occupancy | tinyint unsigned | default occupants |
| max_occupancy | tinyint unsigned | |
| amenities | json | nullable — list of amenity strings |
| created_at / updated_at | timestamp | |

#### `cabins`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_ship_id | bigint unsigned, FK → cruise_ships.id, cascade delete | indexed |
| ship_deck_id | bigint unsigned, FK → ship_decks.id, cascade delete | indexed |
| cabin_type_id | bigint unsigned, FK → cabin_types.id, restrict delete | indexed |
| cabin_number | varchar(20) | e.g. "7102" |
| status | enum('available','booked','maintenance','blocked') | default 'available', indexed |
| base_price | decimal(10,2) | per-night or per-cruise base rate; currency in `currency` |
| currency | varchar(3) | default 'GHS' or 'USD' |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** composite unique (`cruise_ship_id`, `cabin_number`), `status`, `cabin_type_id`

> Important design note: `cabins.status` reflects current operational state (maintenance/blocked), **not** booking availability for a specific date range — that's derived from `bookings` + `cruise_schedules`. A cabin can be "available" generally but fully booked for a specific sailing. Availability queries join `cabins` against `bookings` for the requested `cruise_schedule_id`, never rely on a single static status flag for booking logic.

---

#### `destinations`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(150) | indexed |
| slug | varchar(160) | unique |
| country | varchar(100) | indexed |
| port_name | varchar(150) | nullable |
| description | text | |
| latitude | decimal(10,7) | nullable |
| longitude | decimal(10,7) | nullable |
| image_path | varchar(255) | nullable |
| created_at / updated_at / deleted_at | timestamp | |

#### `cruises`
A "cruise" is a product template (e.g. "7-Night Caribbean Explorer"); `cruise_schedules` are actual dated sailings of that product.

| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_ship_id | bigint unsigned, FK → cruise_ships.id, restrict delete | indexed |
| name | varchar(180) | |
| slug | varchar(190) | unique |
| description | text | |
| duration_nights | smallint unsigned | |
| base_price | decimal(10,2) | starting-from price for marketing display |
| currency | varchar(3) | |
| status | enum('draft','published','archived') | default 'draft', indexed |
| created_at / updated_at / deleted_at | timestamp | |

#### `cruise_destination` (pivot — itinerary stops, ordered)
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_id | bigint unsigned, FK → cruises.id, cascade delete | indexed |
| destination_id | bigint unsigned, FK → destinations.id, restrict delete | indexed |
| stop_order | tinyint unsigned | sequence in itinerary |
| arrival_offset_days | tinyint unsigned | days from departure |
| departure_offset_days | tinyint unsigned | |

**Indexes:** composite (`cruise_id`, `stop_order`)

#### `cruise_schedules`
The actual bookable, dated sailings.

| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_id | bigint unsigned, FK → cruises.id, restrict delete | indexed |
| departure_date | date | indexed |
| return_date | date | |
| departure_port | varchar(150) | |
| status | enum('scheduled','boarding','in_progress','completed','cancelled') | default 'scheduled', indexed |
| total_cabins_available | int unsigned | snapshot at schedule creation, for quick capacity checks |
| created_at / updated_at | timestamp | |

**Indexes:** `departure_date`, `status`, composite (`cruise_id`, `departure_date`)

---

#### `bookings`
The central transactional table — one booking can cover a cruise, one or more cabins, and one or more passengers.

| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| booking_reference | varchar(20) | unique, indexed — customer-facing code |
| user_id | bigint unsigned, FK → users.id, restrict delete | indexed |
| cruise_schedule_id | bigint unsigned, FK → cruise_schedules.id, restrict delete | indexed |
| cabin_id | bigint unsigned, FK → cabins.id, restrict delete | indexed — nullable if multi-cabin handled via separate `booking_cabins` (see note) |
| status | enum('pending','confirmed','checked_in','completed','cancelled','refunded') | default 'pending', indexed |
| total_amount | decimal(10,2) | |
| currency | varchar(3) | |
| booked_at | timestamp | |
| cancelled_at | timestamp | nullable |
| cancellation_reason | text | nullable |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** `booking_reference` (unique), `user_id`, `cruise_schedule_id`, `status`, composite (`cruise_schedule_id`, `cabin_id`)

> **Scaling note:** This schema models one cabin per booking for simplicity in Phase 1 (matches "Reserve cabins" as a singular flow). If group bookings spanning multiple cabins under one reservation are needed, add a `booking_cabins` pivot table (`booking_id`, `cabin_id`, `price_at_booking`) and drop `cabin_id` from `bookings`. Flagging now since it's a schema-shape decision, cheap to do before Phase 3 (Booking System), expensive after.

#### `passengers`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| booking_id | bigint unsigned, FK → bookings.id, cascade delete | indexed |
| first_name | varchar(100) | |
| last_name | varchar(100) | |
| date_of_birth | date | |
| passport_number | varchar(50) | nullable, encrypted at rest |
| nationality | varchar(100) | nullable |
| is_lead_passenger | boolean | default false |
| special_requirements | text | nullable — dietary, accessibility |
| created_at / updated_at | timestamp | |

**Indexes:** `booking_id`

#### `payments`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| payable_type | varchar(100) | polymorphic — `Booking`, `VenueBooking`, `PackageBooking` |
| payable_id | bigint unsigned | polymorphic target |
| user_id | bigint unsigned, FK → users.id, restrict delete | indexed |
| gateway | varchar(50) | e.g. `stripe`, `hubtel` |
| gateway_reference | varchar(255) | nullable, indexed — transaction ID from gateway |
| amount | decimal(10,2) | |
| currency | varchar(3) | |
| status | enum('pending','processing','successful','failed','refunded') | default 'pending', indexed |
| paid_at | timestamp | nullable |
| failure_reason | text | nullable |
| raw_response | json | nullable — full gateway payload for debugging |
| created_at / updated_at | timestamp | |

**Indexes:** composite (`payable_type`, `payable_id`), `gateway_reference`, `status`, `user_id`

---

#### `venues`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_ship_id | bigint unsigned, FK → cruise_ships.id, cascade delete | indexed |
| name | varchar(150) | e.g. "Grand Ballroom", "Sky Lounge" |
| slug | varchar(160) | |
| description | text | |
| capacity | int unsigned | indexed — critical for AI "venue for 250 guests" queries |
| venue_type | enum('ballroom','theater','lounge','deck','dining_hall','conference') | indexed |
| hourly_rate | decimal(10,2) | nullable |
| flat_rate | decimal(10,2) | nullable |
| currency | varchar(3) | |
| amenities | json | nullable |
| image_path | varchar(255) | nullable |
| status | enum('available','maintenance','retired') | default 'available' |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** `capacity`, `venue_type`, `cruise_ship_id`

#### `venue_bookings`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| booking_reference | varchar(20) | unique |
| venue_id | bigint unsigned, FK → venues.id, restrict delete | indexed |
| user_id | bigint unsigned, FK → users.id, restrict delete | indexed |
| cruise_schedule_id | bigint unsigned, FK → cruise_schedules.id, nullable, restrict delete | indexed — venue booking may be tied to a specific sailing |
| event_id | bigint unsigned, FK → events.id, nullable, restrict delete | indexed — null if it's a private/custom booking, not tied to a published event |
| guest_count | int unsigned | |
| start_datetime | datetime | indexed |
| end_datetime | datetime | |
| status | enum('pending','confirmed','cancelled','completed') | default 'pending', indexed |
| total_amount | decimal(10,2) | |
| currency | varchar(3) | |
| notes | text | nullable |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** composite (`venue_id`, `start_datetime`, `end_datetime`) for overlap-checking queries, `status`, `user_id`

#### `events`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| venue_id | bigint unsigned, FK → venues.id, restrict delete | indexed |
| cruise_schedule_id | bigint unsigned, FK → cruise_schedules.id, nullable, restrict delete | indexed |
| name | varchar(180) | |
| description | text | |
| event_type | enum('show','party','dining','excursion_briefing','workshop','other') | indexed |
| start_datetime | datetime | indexed |
| end_datetime | datetime | |
| capacity | int unsigned | nullable — may differ from venue max |
| is_ticketed | boolean | default false |
| ticket_price | decimal(10,2) | nullable |
| status | enum('scheduled','cancelled','completed') | default 'scheduled' |
| created_at / updated_at | timestamp | |

**Indexes:** `start_datetime`, `event_type`, `venue_id`

---

#### `dining_options`
(Implied by requirements — "dining options" the AI must answer about — added as a supporting table.)

| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_ship_id | bigint unsigned, FK → cruise_ships.id, cascade delete | indexed |
| name | varchar(150) | e.g. "The Captain's Table", "Buffet Deck 9" |
| cuisine_type | varchar(100) | nullable, indexed |
| dress_code | varchar(100) | nullable |
| is_included_in_fare | boolean | default true |
| extra_cost | decimal(10,2) | nullable |
| operating_hours | json | nullable — structured open/close per meal period |
| description | text | |
| created_at / updated_at / deleted_at | timestamp | |

#### `dining_reservations`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| booking_id | bigint unsigned, FK → bookings.id, cascade delete | indexed |
| dining_option_id | bigint unsigned, FK → dining_options.id, restrict delete | indexed |
| reservation_datetime | datetime | indexed |
| party_size | tinyint unsigned | |
| status | enum('confirmed','cancelled','completed') | default 'confirmed' |
| created_at / updated_at | timestamp | |

**Indexes:** composite (`dining_option_id`, `reservation_datetime`)

---

#### `inventory_categories` and `inventory`
(Splitting into two tables — categories vs items — since "manage inventory" implies more than one item type: galley stock, retail, maintenance parts.)

**`inventory_categories`**
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(100) | e.g. "Galley Supplies", "Retail", "Maintenance Parts" |
| created_at / updated_at | timestamp | |

**`inventory`**
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| cruise_ship_id | bigint unsigned, FK → cruise_ships.id, cascade delete | indexed |
| inventory_category_id | bigint unsigned, FK → inventory_categories.id, restrict delete | indexed |
| sku | varchar(100) | nullable, indexed |
| name | varchar(180) | |
| unit | varchar(30) | e.g. "kg", "unit", "litre" |
| quantity_on_hand | decimal(12,2) | default 0 |
| reorder_threshold | decimal(12,2) | nullable — triggers low-stock alerts |
| unit_cost | decimal(10,2) | nullable |
| currency | varchar(3) | nullable |
| last_restocked_at | timestamp | nullable |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** `sku`, composite (`cruise_ship_id`, `inventory_category_id`)

> `inventory_movements` (stock in/out audit log) is a natural Phase 2/3 addition once basic CRUD is live — not blocking for Phase 1 schema but worth planning for: `inventory_id`, `change_quantity`, `reason`, `staff_user_id`, `created_at`.

---

#### `packages`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| name | varchar(150) | e.g. "Honeymoon Package", "All-Inclusive Drinks" |
| slug | varchar(160) | unique |
| description | text | |
| package_type | enum('honeymoon','family','luxury','spa','excursion','dining','other') | indexed — supports AI recommendation queries directly |
| price | decimal(10,2) | |
| currency | varchar(3) | |
| includes | json | nullable — structured list of inclusions for both display and AI context |
| applicable_cruise_id | bigint unsigned, FK → cruises.id, nullable, restrict delete | indexed — null means applies to all cruises |
| status | enum('active','inactive') | default 'active', indexed |
| created_at / updated_at / deleted_at | timestamp | |

**Indexes:** `package_type`, `status`, `slug` (unique)

#### `package_bookings`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| booking_id | bigint unsigned, FK → bookings.id, cascade delete | indexed |
| package_id | bigint unsigned, FK → packages.id, restrict delete | indexed |
| quantity | smallint unsigned | default 1 |
| price_at_booking | decimal(10,2) | snapshot — package price may change later |
| currency | varchar(3) | |
| status | enum('active','cancelled','refunded') | default 'active' |
| created_at / updated_at | timestamp | |

**Indexes:** composite (`booking_id`, `package_id`)

---

#### `ai_conversations`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| user_id | bigint unsigned, FK → users.id, nullable, cascade delete | indexed — nullable for anonymous pre-login visitors |
| session_id | varchar(100) | indexed — for anonymous tracking before login |
| title | varchar(255) | nullable — auto-generated summary for history list |
| status | enum('active','closed') | default 'active', indexed |
| last_message_at | timestamp | indexed |
| created_at / updated_at | timestamp | |

#### `ai_messages`
| Column | Type | Notes |
|---|---|---|
| id | bigint unsigned, PK | |
| ai_conversation_id | bigint unsigned, FK → ai_conversations.id, cascade delete | indexed |
| role | enum('user','assistant','system') | indexed |
| content | text | |
| context_used | json | nullable — what DB records/context were injected for this response, for debugging/auditing AI answers |
| tokens_used | int unsigned | nullable — cost tracking |
| created_at | timestamp | |

**Indexes:** composite (`ai_conversation_id`, `created_at`)

### 2.3 Cross-Cutting Schema Notes

- **Money:** stored as `decimal(10,2)` here for readability in this doc; in migration code, store as integer minor units (`amount_minor` bigint) to eliminate float/rounding issues entirely — this is a stronger guarantee than decimal columns provide once currency conversion or partial refunds enter the picture.
- **Soft deletes** on `users`, `cruise_ships`, `cabins`, `bookings`, `venues`, `venue_bookings`, `packages`, `destinations`, `dining_options`, `inventory` — financial/customer history must never hard-delete.
- **Restrict vs cascade deletes:** cascade only where the child record is meaningless without the parent and has no independent financial/audit value (e.g. `passengers` belong wholly to one `booking`). Restrict everywhere a delete could silently orphan financial history (e.g. you cannot delete a `cabin_type` that has cabins still referencing it).
- **Full-text search:** add MySQL FULLTEXT indexes on `cruises.description`, `destinations.description`, `venues.description` in Phase 6 once content volume justifies it — premature on day one.
## 3. Laravel Module Structure

Laravel doesn't enforce a module system out of the box, so "modules" here means a **consistent internal folder convention** inside the standard `app/` structure — not a package-per-module setup, which would be overkill for a single-team build. Each module gets its own namespace under `app/Domain/{Module}/` containing Models, Services, Actions, and Policies, with thin Livewire components and Controllers in the standard `app/Livewire/` and `app/Http/Controllers/` calling into them.

```
app/
├── Domain/
│   ├── Auth/
│   │   ├── Services/ (AuthService, TwoFactorService)
│   │   └── Policies/
│   ├── Customer/
│   │   ├── Models/ (extends base User scope)
│   │   ├── Services/ (CustomerService)
│   │   └── Actions/ (CreateCustomer, SuspendCustomer)
│   ├── Cruise/
│   │   ├── Models/ (CruiseShip, ShipDeck, Cabin, CabinType, Destination, Cruise, CruiseSchedule)
│   │   ├── Services/ (CruiseService, ScheduleService, AvailabilityService)
│   │   └── Actions/
│   ├── Booking/
│   │   ├── Models/ (Booking, Passenger)
│   │   ├── Services/ (BookingService, CancellationService)
│   │   ├── Actions/ (CreateBooking, CancelBooking, CheckInPassenger)
│   │   └── Events/ (BookingCreated, BookingConfirmed, BookingCancelled)
│   ├── Venue/
│   │   ├── Models/ (Venue, VenueBooking, Event)
│   │   ├── Services/ (VenueService, VenueAvailabilityService)
│   │   └── Actions/
│   ├── Dining/
│   │   ├── Models/ (DiningOption, DiningReservation)
│   │   └── Services/ (DiningService)
│   ├── Inventory/
│   │   ├── Models/ (InventoryCategory, InventoryItem)
│   │   └── Services/ (InventoryService, StockAlertService)
│   ├── Package/
│   │   ├── Models/ (Package, PackageBooking)
│   │   └── Services/ (PackageService, PackageRecommendationService)
│   ├── Payment/
│   │   ├── Models/ (Payment)
│   │   ├── Services/ (PaymentService, RefundService)
│   │   ├── Contracts/ (PaymentGatewayInterface)
│   │   └── Gateways/ (StripeGateway, HubtelGateway — pluggable)
│   ├── AI/
│   │   ├── Models/ (AiConversation, AiMessage)
│   │   ├── Services/ (AIAssistantService, ContextBuilderService, OpenAIClientService)
│   │   └── Tools/ (FindShipsTool, CheckAvailabilityTool, RecommendPackageTool — function-calling definitions, see Section 6)
│   └── Reporting/
│       ├── Services/ (RevenueReportService, OccupancyReportService, VenueUtilizationService)
│       └── Queries/ (raw aggregate query builders, kept out of models)
│
├── Livewire/
│   ├── Customer/ (ShipBrowser, CruiseDetail, BookingWizard, CabinSelector, ChatWidget, MyBookings)
│   └── Admin/ (ShipManager, CabinManager, ScheduleManager, BookingManager, VenueManager,
│               InventoryManager, CustomerManager, StaffManager, ReportsDashboard)
│
├── Http/
│   ├── Controllers/Api/ (thin — delegate to Domain services)
│   └── Middleware/ (RoleMiddleware, PermissionMiddleware)
│
├── Notifications/ (BookingConfirmed, PaymentReceived, BookingCancelled, LowStockAlert, VenueBookingConfirmed)
├── Policies/ (one per module's primary model, registered against roles/permissions)
└── Jobs/ (SendBookingConfirmation, RefreshAIEmbeddings, GenerateDailyOccupancyReport, ProcessRefund)
```

**Why this structure over package-based modules** (e.g. `nwidart/laravel-modules`): for a single-team, single-app build, package-based modularity adds ceremony (separate composer autoloading, module-specific service providers) without a real payoff — there's no plan to extract modules into separately-deployed services. The `Domain/` convention gives the same logical separation and discoverability while staying inside standard Laravel app structure, which keeps onboarding new developers fast.

---

## 4. User Roles & Permissions

### 4.1 Role Definitions

| Role | Scope |
|---|---|
| **Super Admin** | Full system access, including staff/role management and system configuration |
| **Operations Manager** | Cross-module oversight: ships, schedules, bookings, inventory, reports — but not staff/role management or financial refund approval |
| **Cruise Manager** | Ships, decks, cabins, cruises, schedules — read/write. No access to payments, customers, or other modules' management screens |
| **Venue Manager** | Venues, venue bookings, events — read/write. Read-only on cruise schedules (for date-conflict context) |
| **Customer Service** | Customers, bookings, passengers — read/write, including booking modifications and cancellations. No financial refund authority, no ship/cabin configuration |
| **Finance Officer** | Payments, refunds, financial reports — read/write. Read-only on bookings (to see what's being paid for) |
| **Customer** | Self-service only: own bookings, own profile, browsing, AI chat |

### 4.2 Permission Matrix

`✓✓` = full CRUD · `✓` = read-only or limited action · blank = no access

| Permission Group | Super Admin | Ops Manager | Cruise Mgr | Venue Mgr | Customer Service | Finance Officer | Customer |
|---|---|---|---|---|---|---|---|
| ships.manage | ✓✓ | ✓✓ | ✓✓ | | | | |
| cabins.manage | ✓✓ | ✓✓ | ✓✓ | | | | |
| schedules.manage | ✓✓ | ✓✓ | ✓✓ | ✓ | ✓ | | |
| destinations.manage | ✓✓ | ✓✓ | ✓✓ | | | | |
| venues.manage | ✓✓ | ✓✓ | | ✓✓ | | | |
| venue_bookings.manage | ✓✓ | ✓✓ | | ✓✓ | ✓ | | own only |
| events.manage | ✓✓ | ✓✓ | | ✓✓ | | | |
| bookings.manage | ✓✓ | ✓✓ | ✓ | | ✓✓ | ✓ | own only |
| bookings.cancel | ✓✓ | ✓✓ | | | ✓✓ | | own (policy-limited) |
| customers.manage | ✓✓ | ✓✓ | | | ✓✓ | ✓ | own profile |
| staff.manage | ✓✓ | | | | | | |
| roles.manage | ✓✓ | | | | | | |
| inventory.manage | ✓✓ | ✓✓ | | | | | |
| payments.view | ✓✓ | ✓ | | | ✓ | ✓✓ | own only |
| payments.refund | ✓✓ | | | | | ✓✓ | |
| packages.manage | ✓✓ | ✓✓ | ✓ | | | | |
| reports.view | ✓✓ | ✓✓ | module-scoped | module-scoped | | financial only | |
| ai_conversations.manage | ✓✓ | ✓ | | | ✓ | | own only |
| system.settings | ✓✓ | | | | | | |

### 4.3 Implementation Approach

- Roles and permissions stored relationally (Section 2: `roles`, `permissions`, pivots) rather than hardcoded in middleware, so admins can adjust the matrix without a deploy.
- Enforced at three layers: **route middleware** (`role:` / `permission:`) for whole-page access, **Laravel Policies** for per-model action checks (e.g. a Customer Service rep can cancel a booking but only within policy-defined windows), and **Blade/Livewire `@can` directives** for UI element visibility.
- Customers are a `role` like any other (not a separate auth guard) — this keeps a single login system and lets the same `User` model serve both staff and customers, distinguished by `user_type` and role assignment. A customer's own-record access (e.g. "own bookings only") is enforced via Policy methods comparing `booking.user_id === auth()->id()`, not via a separate permission.

---

## 5. Admin Dashboard Design

### 5.1 Widget Inventory

| Widget | Data Shown | Refresh Strategy |
|---|---|---|
| **Revenue Today / MTD / YTD** | Sum of successful `payments`, broken down by source (cruise / venue / package) | Cached 15 min, recalculated via scheduled job for YTD |
| **Revenue Trend Chart** | Daily revenue, last 30/90 days, line chart | Pre-aggregated nightly into a `daily_revenue_snapshots` reporting table (Phase 6) to avoid scanning `payments` live |
| **Bookings Today / This Week** | Count + list of new `bookings` by status | Live query, lightweight |
| **Upcoming Departures** | Next 7 days of `cruise_schedules`, with booked vs. capacity | Live query |
| **Occupancy Rate by Ship** | Booked cabins ÷ total cabins, per active `cruise_schedule` | Computed on demand, cached 5 min |
| **Occupancy Heatmap** | Cabin-type occupancy across upcoming schedules — helps pricing decisions | Computed nightly |
| **Venue Utilization** | Booked hours ÷ available hours per venue, this week/month | Live query against `venue_bookings` |
| **Top Venues by Bookings** | Ranked list | Cached 1 hr |
| **Customer Growth** | New signups over time | Live query, lightweight |
| **Customer Lifetime Value (top spenders)** | Sum of `payments` per `user_id`, ranked | Cached 1 hr |
| **Low Stock Alerts** | `inventory` rows where `quantity_on_hand` ≤ `reorder_threshold` | Live query + triggers `LowStockAlert` notification |
| **AI Assistant Usage** | Conversations started, resolution indicators (did booking follow chat) | Daily aggregate |
| **Pending Refunds** | Payments with status needing Finance Officer action | Live query, visible only to roles with `payments.refund` |

### 5.2 Dashboard Composition by Role

The dashboard is **role-aware**, not one-size-fits-all — each role's landing dashboard surfaces only what they act on:

- **Super Admin / Operations Manager:** all widgets
- **Cruise Manager:** occupancy, upcoming departures, ship-specific revenue
- **Venue Manager:** venue utilization, top venues, upcoming events
- **Customer Service:** bookings today, customer growth, pending support-relevant bookings
- **Finance Officer:** revenue widgets, pending refunds, payment status breakdown

### 5.3 Technical Notes

- Built as Livewire components with `wire:poll` for near-real-time widgets (e.g. bookings today) and standard page-load queries for trend charts.
- Chart rendering via Alpine.js + Chart.js (lightweight, no need for a heavier charting framework for Phase 1 dashboard needs).
- Heavy aggregate queries (YTD revenue, occupancy heatmap) move to scheduled jobs writing to dedicated reporting tables as soon as live-query performance becomes noticeable — this is explicitly Phase 6 work (Section 8), not Phase 1, but the dashboard should be built against a `ReportingService` interface from day one so swapping the underlying query for a pre-aggregated table later doesn't touch the UI layer.

---

## 6. AI Architecture

### 6.1 Core Principle: RAG, Not Fine-Tuning

Fine-tuning a model on cruise inventory would freeze it in time — cabin availability and schedules change constantly. Instead, the system uses **Retrieval-Augmented Generation**: the OpenAI model gets fresh, structured context pulled from MySQL at the moment of each query, plus the ability to call back into the system mid-conversation via **function calling (tool use)**.

### 6.2 How Laravel Stores and Serves Knowledge

| Knowledge Type | Storage | How AI Accesses It |
|---|---|---|
| Ship/cabin/cruise data | Standard relational tables (Section 2) | Queried live via `ContextBuilderService` based on detected intent, or via tool-calling at response time |
| Cruise schedules & availability | `cruise_schedules`, `bookings`, `cabins` | Live availability query — never cached for AI use, since stale availability data is worse than no answer |
| Venue capacity/availability | `venues`, `venue_bookings` | Live query, same reasoning |
| Packages | `packages` (with `includes` JSON for structured inclusions) | Queried by `package_type` for recommendation-style questions |
| Dining options | `dining_options` | Queried directly |
| Static company knowledge (policies, FAQs, "what's included in a cruise fare") | A new `knowledge_base_articles` table (title, content, tags, embedding vector) | Semantic search via embeddings (Section 6.4) — this is genuinely document-like content, unlike live inventory |

**Why not dump the whole DB into the prompt:** that's slow, expensive (token cost scales with context size), and increases hallucination risk by burying relevant rows in irrelevant ones. Instead the system narrows context first, then sends only what's relevant.

### 6.3 Request Flow

```
Customer message arrives
        │
        ▼
AIAssistantService::handleMessage()
        │
        ├─→ Store user message in ai_messages
        │
        ├─→ Send to OpenAI with:
        │     - System prompt (role, tone, scope boundaries)
        │     - Conversation history (recent ai_messages, windowed)
        │     - Available "tools" (function definitions — see 6.5)
        │
        ▼
OpenAI responds either with:
        │
        ├─→ A direct text answer (no tool needed)
        │         → store in ai_messages → stream to widget
        │
        └─→ A tool_call request (e.g. "call check_venue_availability with capacity=250")
                  │
                  ▼
        Laravel executes the actual tool: AI/Tools/CheckVenueAvailabilityTool
                  │
                  ▼
        Runs a real, scoped Eloquent query against `venues`/`venue_bookings`
                  │
                  ▼
        Result sent back to OpenAI as tool result
                  │
                  ▼
        OpenAI generates final natural-language answer using that real data
                  │
                  ▼
        Stored in ai_messages, context_used logged for auditing → streamed to widget
```

This tool-calling loop is what makes the two examples in your spec work correctly:

- **"I need a honeymoon cruise"** → model calls a `find_packages` tool with `package_type=honeymoon`, gets real rows back from the `packages` table, then composes a recommendation grounded in actual current packages/prices — not invented ones.
- **"I need a venue for 250 guests"** → model calls `check_venue_availability` with `min_capacity=250`, gets real `venues` rows where `capacity >= 250` and no conflicting `venue_bookings`, and answers from that.

### 6.4 Embeddings for Unstructured Knowledge

For the smaller slice of knowledge that's genuinely document-like (policies, marketing copy, long-form FAQs) rather than relational:

- `knowledge_base_articles` table stores `title`, `content`, `tags`, and an `embedding` column (JSON array or a dedicated vector extension if MySQL setup supports it).
- A queued job (`RefreshAIEmbeddings`) regenerates embeddings via OpenAI's embeddings endpoint whenever an article is created/updated.
- At query time, the customer's message is embedded and compared via cosine similarity against stored embeddings to retrieve the top N relevant articles, which get added to context.
- This is **separate** from the live-query tool-calling path above — embeddings answer "what's your cancellation policy," tool-calling answers "is cabin 7102 available on this date."

### 6.5 Tool (Function) Definitions — Phase 1 Set

| Tool name | Purpose | Underlying query |
|---|---|---|
| `find_ships` | List ships matching filters (capacity, amenities) | `cruise_ships` |
| `find_cruises` | List cruises by destination, duration, date range | `cruises` + `cruise_schedules` + `destinations` |
| `check_cabin_availability` | Cabins available for a given schedule + type | `cabins` minus booked for that `cruise_schedule_id` |
| `find_packages` | Packages by type (honeymoon, family, etc.) | `packages` |
| `check_venue_availability` | Venues by minimum capacity + date/time window, excluding conflicts | `venues` minus `venue_bookings` overlaps |
| `find_dining_options` | Dining venues by cuisine/cost/ship | `dining_options` |
| `get_event_schedule` | Events in a date range or on a given schedule | `events` |

Each tool is a small, single-purpose class implementing a shared `AIToolInterface` (`name()`, `schema()`, `execute(array $args): array`) — this keeps adding new tools (e.g. later phases: `get_loyalty_points`, `suggest_upsell`) a matter of writing one new class and registering it, not touching the core AI service.

### 6.6 Guardrails

- System prompt explicitly scopes the assistant to cruise-related queries and instructs it to decline off-topic requests and never invent availability, prices, or policies not returned by a tool call.
- All tool queries are scoped server-side (e.g. only `status != 'cancelled'` schedules, only published cruises) — the model never gets raw unrestricted query access.
- `ai_messages.context_used` logs exactly what was retrieved for every response, so any answer can be audited against what data it was actually given.
- Rate limiting and cost tracking via `tokens_used` per message, aggregated for the AI Assistant Usage dashboard widget.

---

## 7. API Architecture (REST, Sanctum-authenticated)

Built primarily for the future mobile app / third-party integrations; the web app itself can use these same endpoints or talk more directly through Livewire — both paths funnel through the same Domain service layer, so behavior can't drift between web and API.

### 7.1 Authentication

```
POST   /api/v1/auth/register
POST   /api/v1/auth/login                  → returns Sanctum token
POST   /api/v1/auth/logout
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
GET    /api/v1/auth/me                      → current user + roles
```

### 7.2 Cruises & Schedules

```
GET    /api/v1/ships
GET    /api/v1/ships/{ship}
GET    /api/v1/cruises
GET    /api/v1/cruises/{cruise}
GET    /api/v1/cruises/{cruise}/schedules
GET    /api/v1/schedules/{schedule}/availability   → cabin counts by type
GET    /api/v1/destinations
```

### 7.3 Bookings

```
POST   /api/v1/bookings                     → create booking (pending)
GET    /api/v1/bookings                     → current user's bookings
GET    /api/v1/bookings/{booking}
PATCH  /api/v1/bookings/{booking}/cancel
POST   /api/v1/bookings/{booking}/passengers
POST   /api/v1/bookings/{booking}/dining-reservations
POST   /api/v1/bookings/{booking}/packages
```

### 7.4 Payments

```
POST   /api/v1/payments/initiate            → { payable_type, payable_id, gateway }
POST   /api/v1/payments/webhook/{gateway}    → unauthenticated, signature-verified
GET    /api/v1/payments/{payment}/status
```

### 7.5 Venues

```
GET    /api/v1/venues
GET    /api/v1/venues/{venue}/availability   → ?start=&end=&guest_count=
POST   /api/v1/venue-bookings
GET    /api/v1/venue-bookings                → current user's venue bookings
```

### 7.6 AI Assistant

```
POST   /api/v1/ai/conversations              → start conversation
POST   /api/v1/ai/conversations/{id}/messages → send message, returns streamed/complete response
GET    /api/v1/ai/conversations/{id}/messages → history
```

### 7.7 Admin API namespace

All admin-facing CRUD (ships, cabins, schedules, venues, inventory, staff, reports) lives under `/api/v1/admin/*`, gated by `role:` middleware and Sanctum token scopes (`abilities`) — so a staff member's token can be scoped to only what their role permits even at the token level, not just app-level policy checks. This matters once staff start using a future mobile admin app where token theft risk is non-trivial.

### 7.8 Conventions

- All list endpoints support `?page=`, `?per_page=`, and consistent filter query params.
- Responses follow `{ data, meta, links }` shape (Laravel's default API Resource pagination format) for predictability.
- Errors follow `{ message, errors }` shape matching Laravel's default validation exception format.
- Versioned from day one (`/api/v1/`) since this is a long-lived platform — breaking changes later get a `/v2/` rather than breaking existing integrations.

---

## 8. Development Roadmap

### Phase 1 — Foundation (this document + initial scaffolding)
- Laravel 12 project setup, Livewire starter kit, Tailwind config, Sanctum installed
- Full database schema migrated (Section 2)
- RBAC system built and seeded (roles, permissions, seeders for the 7 roles)
- Base auth flows (register/login/password reset) for both customer and staff
- Domain folder structure scaffolded with empty Service/Action classes per module
- CI pipeline, environment configs (local/staging/production), error tracking (e.g. Sentry) wired in

### Phase 2 — Cruise Management
- Ships, decks, cabins, cabin types — full admin CRUD
- Destinations CRUD
- Cruises + itinerary builder (ordered destination pivot)
- Cruise schedules CRUD with capacity snapshotting
- Customer-facing ship/cruise browsing pages (Livewire)
- Image upload handling (ship photos, deck plans) to S3/local storage

### Phase 3 — Booking System
- Cabin availability engine (the core "is this cabin free for this schedule" logic — gets its own test suite given how central it is)
- Booking wizard (Livewire multi-step): select schedule → select cabin → passenger details → review
- Passengers CRUD tied to bookings
- Payment integration: Stripe primary gateway, `PaymentGatewayInterface` built so a second gateway is a class, not a rewrite
- Payment webhooks, booking status transitions, confirmation notifications (email/SMS)
- Customer "My Bookings" portal
- Booking cancellation flow with policy-based rules (e.g. cancellation windows)

### Phase 4 — Venue & Event Management
- Venues CRUD, capacity-based search
- Venue booking flow with overlap-conflict checking
- Events CRUD, ticketed events with capacity tracking
- Dining options + dining reservations
- Packages CRUD + package booking attachment to bookings
- Admin venue utilization views

### Phase 5 — AI Assistant
- `ai_conversations` / `ai_messages` schema live, conversation UI widget (Livewire, streaming)
- OpenAI integration service, system prompt design, tool-calling framework (`AIToolInterface`)
- Phase 1 tool set built: `find_ships`, `find_cruises`, `check_cabin_availability`, `find_packages`, `check_venue_availability`, `find_dining_options`, `get_event_schedule`
- `knowledge_base_articles` + embeddings pipeline for policy/FAQ content
- Auditing (`context_used` logging), cost tracking, rate limiting
- AI usage dashboard widget

### Phase 6 — Reporting & Analytics
- Reporting tables for pre-aggregated data (daily revenue snapshots, occupancy snapshots)
- Scheduled jobs to populate them nightly
- Full admin dashboard build-out per Section 5 (role-aware widgets)
- Export functionality (CSV/PDF) for finance and operations reports
- Full-text search indexes added where content volume now justifies them

### Phase 7 — Production Deployment & Hardening
- Load testing on booking/payment concurrency paths (race conditions on cabin booking are the highest-risk area — needs explicit locking strategy, e.g. `lockForUpdate()` or optimistic locking on cabin selection)
- Security review: encrypted PII fields (passport numbers), Sanctum token scoping audit, payment webhook signature verification audit
- Backup/disaster recovery strategy for MySQL
- Queue worker supervision (Horizon recommended), monitoring/alerting
- Production environment provisioning, SSL, CDN for static assets (ship images, deck plans)
- Staff training documentation, customer-facing help content
- Soft launch / phased rollout plan

---

## 9. Open Questions to Resolve Before Build Starts

These aren't blockers for continuing the design conversation, but they do affect schema/architecture decisions enough that resolving them before Phase 1 coding starts will save rework:

1. **Single ship line or multi-tenant?** Confirmed assumption in Section 0 — flag if wrong.
2. **Multi-cabin bookings:** does one booking ever need to cover more than one cabin (e.g. a family booking two adjoining cabins under one reference)? Affects whether `bookings.cabin_id` stays a direct FK or becomes a `booking_cabins` pivot (Section 2.2 note).
3. **Currency:** single currency (GHS or USD) or multi-currency display with one settlement currency? Affects whether `currency` columns are decorative or load-bearing.
4. **Payment gateway priority:** Stripe assumed as primary — confirm, especially given your existing Hubtel integration experience; might make sense to build Hubtel as the first gateway implementation instead, reusing lessons from the voucher portal work.
5. **Mobile app timeline:** if a mobile app is coming soon rather than "eventually," the API (Section 7) should be prioritized earlier than Phase 5, not built incidentally alongside it.
