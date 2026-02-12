# HadatHub - REST API Design Specification

## Regional Event & Ticketing System

---

## Executive Summary

HadatHub's API architecture follows RESTful principles to provide a scalable, intuitive platform for managing regional events across East Africa. The design centers on four core resources (Events, Venues, Attendees, Tickets) with clear relationships and business constraints that prevent overbooking while enabling flexible event management. This specification prioritizes developer experience through consistent URI patterns, proper HTTP semantics, and comprehensive error handling that supports both web and mobile clients.

---

## 1. Domain Analysis

### Business Problem

HadatHub replaces manual Excel-based event management with a centralized digital platform. The system must handle diverse event types (conferences, concerts, festivals) across multiple cities while managing venue capacity, ticket sales, and attendee check-ins in real-time.

### Primary Entities

**1. Events**

- **Purpose**: Core business entity representing any ticketed gathering
- **Why**: Events are the primary product being sold; everything else supports event delivery
- **Key Operations**: Create, publish, cancel, reschedule, check attendance

**2. Venues**

- **Purpose**: Physical locations hosting events
- **Why**: Venues impose capacity constraints and provide location context for discovery
- **Key Operations**: Register venues, manage capacity, track availability

**3. Attendees**

- **Purpose**: Users who purchase tickets and attend events
- **Why**: Need to track who bought what for check-in, refunds, and marketing
- **Key Operations**: Register, purchase tickets, check-in, view history

**4. Tickets**

- **Purpose**: Proof of purchase linking attendees to specific events
- **Why**: Represents the transaction; enforces capacity limits and enables access control
- **Key Operations**: Purchase, validate, refund, transfer

**5. Organizers**

- **Purpose**: Users/companies creating and managing events
- **Why**: Different permissions needed; organizers manage events, attendees consume them
- **Key Operations**: Create events, view sales analytics, manage check-ins

### Critical Relationships

```
Organizer (1) ----< (Many) Events
Venue (1) ----< (Many) Events
Event (1) ----< (Many) Tickets
Attendee (1) ----< (Many) Tickets
Event (Many) >----< (Many) Attendees [through Tickets]
```

### Business Rules & Constraints

1. **Capacity Management**: Total tickets sold cannot exceed venue capacity
2. **Temporal Integrity**: Events cannot overlap at the same venue
3. **Ticket Validity**: Tickets are event-specific and non-transferable (unless explicitly allowed)
4. **Cancellation Policy**: Events can be cancelled up to 24 hours before start time
5. **Check-in Logic**: Attendees can only check-in within event time window
6. **Regional Context**: Support for multiple currencies (KES, TZS, UGX) and cities

---

## 2. Resource Specifications

### Resource 1: Events

**Purpose**: Represents scheduled gatherings that attendees can purchase tickets for.

**Schema**:

```json
{
  "event_id": "UUID (Primary Key)",
  "organizer_id": "UUID (Foreign Key -> Organizers)",
  "venue_id": "UUID (Foreign Key -> Venues)",
  "title": "String (max 200 chars)",
  "description": "Text",
  "category": "Enum [conference, concert, festival, workshop, sports, other]",
  "event_date": "ISO 8601 DateTime",
  "end_date": "ISO 8601 DateTime",
  "ticket_price": "Decimal (2 decimal places)",
  "currency": "String (ISO 4217 code: KES, TZS, UGX)",
  "available_tickets": "Integer",
  "status": "Enum [draft, published, cancelled, completed]",
  "image_url": "String (URL)",
  "created_at": "ISO 8601 DateTime",
  "updated_at": "ISO 8601 DateTime"
}
```

**Constraints**:

- `available_tickets` ≤ venue capacity
- `end_date` must be after `event_date`
- Cannot publish event without venue assignment
- Cannot cancel event within 24 hours of start time (business rule)

---

### Resource 2: Venues

**Purpose**: Physical locations where events take place.

**Schema**:

```json
{
  "venue_id": "UUID (Primary Key)",
  "name": "String (max 150 chars)",
  "address": "String",
  "city": "String",
  "country": "String (default: Kenya, Tanzania, Uganda)",
  "capacity": "Integer (min: 1)",
  "venue_type": "Enum [indoor, outdoor, hybrid]",
  "amenities": "Array<String> [parking, wifi, accessibility, catering]",
  "contact_email": "String (email format)",
  "contact_phone": "String",
  "latitude": "Decimal (optional)",
  "longitude": "Decimal (optional)",
  "created_at": "ISO 8601 DateTime",
  "updated_at": "ISO 8601 DateTime"
}
```

**Constraints**:

- `capacity` must be positive integer
- Cannot delete venue if it has upcoming events
- City must be valid East African city

---

### Resource 3: Attendees

**Purpose**: Users who purchase tickets and attend events.

**Schema**:

```json
{
  "attendee_id": "UUID (Primary Key)",
  "first_name": "String (max 100 chars)",
  "last_name": "String (max 100 chars)",
  "email": "String (unique, email format)",
  "phone": "String",
  "date_of_birth": "ISO 8601 Date (optional)",
  "city": "String",
  "country": "String",
  "account_status": "Enum [active, suspended, deleted]",
  "created_at": "ISO 8601 DateTime",
  "updated_at": "ISO 8601 DateTime"
}
```

**Constraints**:

- Email must be unique across system
- Must be 13+ years old (if date_of_birth provided)
- Phone format validation for regional numbers

---

### Resource 4: Tickets

**Purpose**: Represents purchased access to a specific event for an attendee.

**Schema**:

```json
{
  "ticket_id": "UUID (Primary Key)",
  "event_id": "UUID (Foreign Key -> Events)",
  "attendee_id": "UUID (Foreign Key -> Attendees)",
  "purchase_date": "ISO 8601 DateTime",
  "ticket_code": "String (unique, alphanumeric 12 chars)",
  "price_paid": "Decimal",
  "currency": "String (ISO 4217 code)",
  "payment_status": "Enum [pending, completed, refunded, failed]",
  "ticket_status": "Enum [valid, used, cancelled, expired]",
  "checked_in_at": "ISO 8601 DateTime (nullable)",
  "created_at": "ISO 8601 DateTime",
  "updated_at": "ISO 8601 DateTime"
}
```

**Constraints**:

- `ticket_code` must be unique and scannable
- Cannot purchase ticket if event is sold out
- Cannot check-in before event start time
- Refunds only allowed if event is cancelled or 48+ hours before event

---

### Resource 5: Organizers

**Purpose**: Users/entities that create and manage events.

**Schema**:

```json
{
  "organizer_id": "UUID (Primary Key)",
  "organization_name": "String (max 200 chars)",
  "contact_person": "String",
  "email": "String (unique, email format)",
  "phone": "String",
  "business_registration": "String (optional)",
  "city": "String",
  "country": "String",
  "account_status": "Enum [active, suspended, pending_verification]",
  "created_at": "ISO 8601 DateTime",
  "updated_at": "ISO 8601 DateTime"
}
```

**Constraints**:

- Email must be unique
- Must be verified before creating published events
- Cannot delete organizer with active events

---

## 3. Complete Endpoint Specification

### 3.1 Events Endpoints

| Action | Method | URI | Request Body | Success Response | Error Response |
|--------|--------|-----|--------------|------------------|----------------|
| List all events | GET | `/api/v1/events` | None | 200 OK<br>`{"events": [...], "total": 45, "page": 1}` | 500 Internal Server Error |
| Get event details | GET | `/api/v1/events/{event_id}` | None | 200 OK<br>`{"event_id": "...", "title": "..."}` | 404 Not Found<br>`{"error": "Event not found"}` |
| Create event | POST | `/api/v1/events` | `{"organizer_id": "...", "venue_id": "...", "title": "Tech Summit 2024", "event_date": "2024-06-15T09:00:00Z", "ticket_price": 5000.00, "currency": "KES", "available_tickets": 200}` | 201 Created<br>`{"event_id": "...", "status": "draft"}` | 400 Bad Request<br>`{"error": "Invalid date format"}`<br>422 Unprocessable Entity<br>`{"error": "Tickets exceed venue capacity"}` |
| Update event | PUT | `/api/v1/events/{event_id}` | `{"title": "Updated Title", "ticket_price": 6000.00}` | 200 OK<br>`{"event_id": "...", "updated_at": "..."}` | 404 Not Found<br>403 Forbidden<br>`{"error": "Cannot modify published event"}` |
| Delete event | DELETE | `/api/v1/events/{event_id}` | None | 204 No Content | 404 Not Found<br>409 Conflict<br>`{"error": "Cannot delete event with sold tickets"}` |
| Publish event | PATCH | `/api/v1/events/{event_id}/publish` | None | 200 OK<br>`{"status": "published"}` | 400 Bad Request<br>`{"error": "Event missing required venue"}` |
| Cancel event | PATCH | `/api/v1/events/{event_id}/cancel` | `{"reason": "Venue unavailable"}` | 200 OK<br>`{"status": "cancelled", "refund_initiated": true}` | 400 Bad Request<br>`{"error": "Cannot cancel within 24 hours"}` |

---

### 3.2 Venues Endpoints

| Action | Method | URI | Request Body | Success Response | Error Response |
|--------|--------|-----|--------------|------------------|----------------|
| List all venues | GET | `/api/v1/venues` | None | 200 OK<br>`{"venues": [...], "total": 23}` | 500 Internal Server Error |
| Get venue details | GET | `/api/v1/venues/{venue_id}` | None | 200 OK<br>`{"venue_id": "...", "name": "KICC", "capacity": 5000}` | 404 Not Found |
| Create venue | POST | `/api/v1/venues` | `{"name": "Sarit Centre", "city": "Nairobi", "country": "Kenya", "capacity": 300, "venue_type": "indoor"}` | 201 Created<br>`{"venue_id": "...", "name": "Sarit Centre"}` | 400 Bad Request<br>`{"error": "Capacity must be positive"}` |
| Update venue | PUT | `/api/v1/venues/{venue_id}` | `{"capacity": 350, "amenities": ["wifi", "parking"]}` | 200 OK<br>`{"venue_id": "...", "updated_at": "..."}` | 404 Not Found<br>422 Unprocessable Entity<br>`{"error": "Cannot reduce capacity below sold tickets"}` |
| Delete venue | DELETE | `/api/v1/venues/{venue_id}` | None | 204 No Content | 404 Not Found<br>409 Conflict<br>`{"error": "Venue has upcoming events"}` |
| Check availability | GET | `/api/v1/venues/{venue_id}/availability` | Query: `?start_date=2024-06-01&end_date=2024-06-30` | 200 OK<br>`{"available_dates": [...], "booked_dates": [...]}` | 404 Not Found |

---

### 3.3 Attendees Endpoints

| Action | Method | URI | Request Body | Success Response | Error Response |
|--------|--------|-----|--------------|------------------|----------------|
| List all attendees | GET | `/api/v1/attendees` | None | 200 OK<br>`{"attendees": [...], "total": 1250}` | 500 Internal Server Error |
| Get attendee details | GET | `/api/v1/attendees/{attendee_id}` | None | 200 OK<br>`{"attendee_id": "...", "email": "user@example.com"}` | 404 Not Found |
| Register attendee | POST | `/api/v1/attendees` | `{"first_name": "John", "last_name": "Doe", "email": "john@example.com", "phone": "+254712345678", "city": "Nairobi"}` | 201 Created<br>`{"attendee_id": "...", "email": "john@example.com"}` | 400 Bad Request<br>409 Conflict<br>`{"error": "Email already registered"}` |
| Update attendee | PUT | `/api/v1/attendees/{attendee_id}` | `{"phone": "+254798765432", "city": "Mombasa"}` | 200 OK<br>`{"attendee_id": "...", "updated_at": "..."}` | 404 Not Found<br>400 Bad Request |
| Delete attendee | DELETE | `/api/v1/attendees/{attendee_id}` | None | 204 No Content | 404 Not Found<br>409 Conflict<br>`{"error": "Cannot delete attendee with active tickets"}` |

---

### 3.4 Tickets Endpoints

| Action | Method | URI | Request Body | Success Response | Error Response |
|--------|--------|-----|--------------|------------------|----------------|
| List all tickets | GET | `/api/v1/tickets` | None | 200 OK<br>`{"tickets": [...], "total": 3420}` | 500 Internal Server Error |
| Get ticket details | GET | `/api/v1/tickets/{ticket_id}` | None | 200 OK<br>`{"ticket_id": "...", "ticket_code": "ABC123XYZ789"}` | 404 Not Found |
| Purchase ticket | POST | `/api/v1/tickets` | `{"event_id": "...", "attendee_id": "...", "quantity": 2}` | 201 Created<br>`{"tickets": [{...}, {...}], "total_amount": 10000.00}` | 400 Bad Request<br>409 Conflict<br>`{"error": "Event sold out"}`<br>422 Unprocessable Entity<br>`{"error": "Insufficient tickets available"}` |
| Validate ticket | GET | `/api/v1/tickets/{ticket_id}/validate` | None | 200 OK<br>`{"valid": true, "event": {...}, "attendee": {...}}` | 404 Not Found<br>410 Gone<br>`{"error": "Ticket already used"}` |
| Check-in ticket | PATCH | `/api/v1/tickets/{ticket_id}/checkin` | None | 200 OK<br>`{"ticket_status": "used", "checked_in_at": "..."}` | 404 Not Found<br>400 Bad Request<br>`{"error": "Check-in not allowed before event start"}`<br>409 Conflict<br>`{"error": "Ticket already checked in"}` |
| Refund ticket | POST | `/api/v1/tickets/{ticket_id}/refund` | `{"reason": "Cannot attend"}` | 200 OK<br>`{"ticket_status": "cancelled", "refund_amount": 5000.00}` | 404 Not Found<br>400 Bad Request<br>`{"error": "Refund window expired"}` |

---

### 3.5 Organizers Endpoints

| Action | Method | URI | Request Body | Success Response | Error Response |
|--------|--------|-----|--------------|------------------|----------------|
| List all organizers | GET | `/api/v1/organizers` | None | 200 OK<br>`{"organizers": [...], "total": 87}` | 500 Internal Server Error |
| Get organizer details | GET | `/api/v1/organizers/{organizer_id}` | None | 200 OK<br>`{"organizer_id": "...", "organization_name": "..."}` | 404 Not Found |
| Register organizer | POST | `/api/v1/organizers` | `{"organization_name": "TechEvents Ltd", "contact_person": "Jane Smith", "email": "info@techevents.com", "city": "Kigali"}` | 201 Created<br>`{"organizer_id": "...", "account_status": "pending_verification"}` | 400 Bad Request<br>409 Conflict<br>`{"error": "Email already registered"}` |
| Update organizer | PUT | `/api/v1/organizers/{organizer_id}` | `{"phone": "+250788123456"}` | 200 OK<br>`{"organizer_id": "...", "updated_at": "..."}` | 404 Not Found |
| Delete organizer | DELETE | `/api/v1/organizers/{organizer_id}` | None | 204 No Content | 404 Not Found<br>409 Conflict<br>`{"error": "Organizer has active events"}` |

---

## 4. Advanced Features & Specialized Endpoints

### 4.1 Association Endpoints

| Purpose | Method | URI | Success Response | Error Response |
|---------|--------|-----|------------------|----------------|
| Get all tickets for an attendee | GET | `/api/v1/attendees/{attendee_id}/tickets` | 200 OK<br>`{"tickets": [...], "total": 5}` | 404 Not Found |
| Get all attendees for an event | GET | `/api/v1/events/{event_id}/attendees` | 200 OK<br>`{"attendees": [...], "checked_in": 45, "total": 120}` | 404 Not Found |
| Get all events at a venue | GET | `/api/v1/venues/{venue_id}/events` | 200 OK<br>`{"events": [...], "upcoming": 3, "past": 12}` | 404 Not Found |
| Get all events by organizer | GET | `/api/v1/organizers/{organizer_id}/events` | 200 OK<br>`{"events": [...], "total": 8}` | 404 Not Found |
| Get tickets for an event | GET | `/api/v1/events/{event_id}/tickets` | 200 OK<br>`{"tickets": [...], "sold": 150, "available": 50}` | 404 Not Found |

---

### 4.2 Search & Filter Endpoints

| Purpose | Method | URI | Query Parameters | Success Response |
|---------|--------|-----|------------------|------------------|
| Search events | GET | `/api/v1/events/search` | `?city=Nairobi&category=concert&date_from=2024-06-01&date_to=2024-06-30&min_price=0&max_price=10000&status=published` | 200 OK<br>`{"events": [...], "total": 12, "filters_applied": {...}}` |
| Search venues | GET | `/api/v1/venues/search` | `?city=Kampala&min_capacity=100&venue_type=indoor&amenities=wifi,parking` | 200 OK<br>`{"venues": [...], "total": 5}` |
| Search attendees | GET | `/api/v1/attendees/search` | `?email=john@example.com&city=Dar es Salaam` | 200 OK<br>`{"attendees": [...], "total": 1}` |
| Filter tickets | GET | `/api/v1/tickets/search` | `?event_id=...&payment_status=completed&ticket_status=valid` | 200 OK<br>`{"tickets": [...], "total": 89}` |

**Pagination Support** (applies to all list/search endpoints):

- Query params: `?page=1&limit=20&sort_by=created_at&order=desc`
- Response includes: `{"data": [...], "page": 1, "limit": 20, "total": 450, "total_pages": 23}`

---

### 4.3 Domain-Specific Business Operations

| Operation | Method | URI | Request Body | Success Response | Error Response |
|-----------|--------|-----|--------------|------------------|----------------|
| Bulk check-in | POST | `/api/v1/events/{event_id}/checkin/bulk` | `{"ticket_codes": ["ABC123", "XYZ789"]}` | 200 OK<br>`{"checked_in": 2, "failed": [], "summary": {...}}` | 400 Bad Request<br>`{"checked_in": 1, "failed": [{"code": "XYZ789", "reason": "Already used"}]}` |
| Event analytics | GET | `/api/v1/events/{event_id}/analytics` | None | 200 OK<br>`{"tickets_sold": 150, "revenue": 750000, "attendance_rate": 0.85, "demographics": {...}}` | 404 Not Found<br>403 Forbidden |
| Venue occupancy | GET | `/api/v1/venues/{venue_id}/occupancy` | Query: `?date=2024-06-15` | 200 OK<br>`{"date": "...", "events": 3, "total_attendees": 450, "capacity_used": 0.75}` | 404 Not Found |
| Attendee history | GET | `/api/v1/attendees/{attendee_id}/history` | None | 200 OK<br>`{"events_attended": 12, "total_spent": 45000, "upcoming_events": 2, "favorite_categories": ["concert", "festival"]}` | 404 Not Found |
| Transfer ticket | POST | `/api/v1/tickets/{ticket_id}/transfer` | `{"new_attendee_id": "...", "reason": "Gift"}` | 200 OK<br>`{"ticket_id": "...", "new_attendee": {...}, "transferred_at": "..."}` | 404 Not Found<br>403 Forbidden<br>`{"error": "Event does not allow transfers"}` |
| Event capacity check | GET | `/api/v1/events/{event_id}/capacity` | None | 200 OK<br>`{"total_capacity": 200, "sold": 175, "available": 25, "percentage_sold": 0.875}` | 404 Not Found |
| Batch ticket purchase | POST | `/api/v1/tickets/batch` | `{"purchases": [{"event_id": "...", "attendee_id": "...", "quantity": 2}, {...}]}` | 201 Created<br>`{"tickets": [...], "total_amount": 25000, "successful": 2, "failed": 0}` | 400 Bad Request<br>422 Unprocessable Entity |

---

## 5. Design Rationale & Key Decisions

### 5.1 Why Five Resources?

**Events, Venues, Attendees, Tickets, Organizers** form the minimum viable model. I considered adding Categories and Payments as separate resources but decided:

- **Categories**: Better as an enum within Events (simpler, fewer joins)
- **Payments**: Abstracted into ticket payment_status (payment gateway integration is implementation detail)

### 5.2 URI Design Philosophy

Used `/api/v1/` prefix for versioning flexibility. Resource names are plural nouns (`/events` not `/event`) following REST conventions. Nested routes only for clear ownership relationships (e.g., `/attendees/{id}/tickets`), avoiding deep nesting that creates tight coupling.

### 5.3 Status Code Strategy

- **201 Created**: All POST operations that create resources
- **204 No Content**: DELETE operations (no body needed)
- **409 Conflict**: Business rule violations (sold out, duplicate email)
- **422 Unprocessable Entity**: Valid syntax but business logic failure (tickets exceed capacity)
- **410 Gone**: Ticket already used (semantically different from 404)

### 5.4 Capacity Management

The critical constraint is `available_tickets ≤ venue.capacity`. This is enforced at:

1. Event creation (validation)
2. Ticket purchase (atomic decrement with transaction lock)
3. Venue capacity updates (cannot reduce below sold tickets)

### 5.5 Regional Considerations

- **Multi-currency support**: Each event stores its currency (KES, TZS, UGX)
- **Phone format flexibility**: String type to handle different regional formats
- **City-based search**: Primary discovery mechanism for users
- **Timezone handling**: All dates in ISO 8601 with UTC; client converts to local time

### 5.6 Scalability Decisions

- **Pagination**: Default limit of 20, max 100 to prevent large payloads
- **Filtering at database level**: Query params translate to WHERE clauses
- **Ticket codes**: 12-char alphanumeric for fast scanning and uniqueness
- **Soft deletes**: account_status enum instead of hard deletes for audit trail

### 5.7 Security Considerations (Design Level)

- **Authentication**: Not specified in endpoints but assumed (JWT/OAuth2 in headers)
- **Authorization**: Organizers can only modify their own events (403 Forbidden)
- **Rate limiting**: Implied for ticket purchase endpoints to prevent bot attacks
- **PII protection**: Email uniqueness enforced; phone/DOB optional

---

## 6. Error Response Standard

All error responses follow this structure:

```json
{
  "error": "Human-readable error message",
  "error_code": "TICKET_SOLD_OUT",
  "details": {
    "field": "available_tickets",
    "constraint": "Must be > 0"
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "request_id": "req_abc123xyz"
}
```

---

## 7. Future Enhancements (Out of Scope)

- **Waitlist system**: When events sell out
- **Dynamic pricing**: Surge pricing based on demand
- **Seat selection**: For venues with assigned seating
- **Multi-day passes**: Single ticket for festival spanning multiple days
- **Promo codes**: Discount system
- **Social features**: Attendee profiles, friend invites

---

**Document Version**: 1.0  
**Last Updated**: Feb. 12, 2026  
**Author**: Leroy Carew Lead API Architect, HadatHub
