# HadatHub - API Architecture Design

## Project Overview

HadatHub is a regional event and ticketing platform designed for East Africa, enabling seamless management of events from tech conferences in Kigali to music festivals in Dar es Salaam. This repository contains the complete REST API design specification for the platform.

## Project Context

**Challenge**: Replace manual Excel-based event management with a centralized digital system  
**Solution**: RESTful API architecture supporting event creation, venue management, ticket sales, and attendee check-ins  
**Region**: East Africa (Kenya, Tanzania, Uganda)  
**Scale**: Multi-city, multi-currency, multi-event type platform

## Documentation

The complete API design specification is available in:

- **[REST_API_DESIGN.md](REST_API_DESIGN.md)** - Full technical specification

## Key Features

### Core Resources

- **Events** - Conferences, concerts, festivals, workshops
- **Venues** - Physical locations with capacity management
- **Attendees** - Users purchasing and attending events
- **Tickets** - Proof of purchase with validation
- **Organizers** - Event creators and managers

### API Capabilities

- Full CRUD operations for all resources
- Advanced search and filtering (location, date, category, price)
- Bulk operations (check-ins, batch purchases)
- Real-time capacity management
- Multi-currency support (KES, TZS, UGX)
- Domain-specific operations (analytics, transfers, refunds)

## API Highlights

**Base URL**: `/api/v1/`

**Key Endpoints**:

- `GET /events/search?city=Nairobi&category=concert` - Find events
- `POST /tickets` - Purchase tickets
- `PATCH /tickets/{id}/checkin` - Check-in attendees
- `GET /events/{id}/analytics` - View event metrics
- `POST /events/{id}/checkin/bulk` - Bulk check-in

**Status Codes**: 200, 201, 204, 400, 403, 404, 409, 410, 422, 500

## Design Principles

1. **RESTful Architecture** - Resource-oriented design with proper HTTP semantics
2. **Developer-First** - Clear, consistent patterns for easy integration
3. **Business-Driven** - Constraints reflect real-world ticketing scenarios
4. **Regional Context** - Multi-currency, flexible phone formats, East African cities
5. **Scalable** - Pagination, filtering, and efficient data modeling

## Technical Specifications

- **Format**: JSON
- **Authentication**: JWT/OAuth2 (implementation detail)
- **Versioning**: URI-based (`/api/v1/`)
- **Date Format**: ISO 8601
- **Currency Codes**: ISO 4217 (KES, TZS, UGX)
- **IDs**: UUID v4

## Business Rules

- Tickets sold cannot exceed venue capacity
- Events cannot overlap at the same venue
- Cancellations allowed up to 24 hours before event
- Check-ins only during event time window
- Email uniqueness enforced across system
- Refunds available 48+ hours before event

## Project Structure

HadatHub---API-Architecture-Design/
├── README.md                 # This file
└── REST_API_DESIGN.md       # Complete API specification

## Use Cases

**For Organizers**:

- Create and publish events
- Monitor ticket sales and revenue
- Manage attendee check-ins
- View analytics and demographics

**For Attendees**:

- Search events by location, date, category
- Purchase tickets with multiple currencies
- View purchase history
- Transfer tickets (if allowed)

**For Venue Managers**:

- Register venues with capacity limits
- Check availability and occupancy
- Track upcoming events

## Getting Started

1. Review the [REST_API_DESIGN.md](REST_API_DESIGN.md) specification
2. Understand the resource models and relationships
3. Explore endpoint documentation with request/response examples
4. Implement authentication layer (not covered in design spec)
5. Build backend services following the API contract

## Assignment Context

This project was completed as part of a REST API design assignment focusing on:

- Breaking down business requirements into RESTful resources
- Designing developer-friendly API endpoints
- Writing standardized specifications with proper HTTP methods
- Planning comprehensive error handling

**Deliverable**: Design-only specification (no implementation code)

## Author

Leroy Carew, Lead API Architect, HadatHub  
January 2024

---

**Note**: This is a design specification document. Implementation details (database schema, authentication, payment processing) are intentionally omitted and left to the engineering team.
