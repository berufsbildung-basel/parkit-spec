# Project P.A.R.K.I.T Specification & API
This document outlines the general specification for the PARKIT Reservation System and provides the OpenAPI specification for HTTP JSON API.

## Requirements

### Actors
- Anonymous: any unauthenticated user.
- Employee: any active Adobe employee.
- Administrator: an active Adobe employee having received administrative privileges in the reservation system.

### Assets
- Users: user accounts, mostly synced from Okta (username/email, first & last name) plus local profile settings
  (preferred language, ...), plus a "disabled" flag (set by administrators to prevent a user from reserving)
- Vehicles: known vehicles, managed by their owners (users) under their profile. Fields: license plate number,
  make, model, vehicle type (car or motorcycle), ev (electric vehicle, true/false)
- Parking Spots: available parking spots, managed by administrative users. Fields: Parking spot number, location.
- Reservations: represents a reservation by users for a particular parking spot and vehicle. Has a start/end time
  on a specific date, being either a half- or full-day reservation.

### Roles
- Staff: default group for authenticated users. Can create, modify and cancel reservations for themselves and their 
  vehicles. Can create, modify and delete their own vehicles. Can read/view reservation calendar and reservations from
  other users on that calendar.
- Administrators: members of the Okta group `DX_AEM_PARKIT_ADMINS`. Administrators can enable/disable users and read/modify
  user profiles local to the reservation system, create/read/modify/cancel reservations, create/read/modify/delete user vehicles,
  create/read/modify/delete parking spots.
- LED-Matrix: Each parking spot is fitted with an LED-Matrix to display reservation information. The controller of
  those matrices needs access to the day's reservations including vehicle info for displaying relevant info on the LEDs.
  This role cannot be given to staff users of the webapp and will require specialized service-to-service authentication.
- Anonymous: all other users.

### Authentication and Authorization
- Authentication should happen via corporate Okta
  - Exception: service-to-service communication between LED-Matrix-controller and the webapp (use API key instead)
- Authorization:
  - Users successfully authenticated via Okta receive the role "Staff" by default
  - Users in the group `DX_AEM_PARKIT_ADMINS` receive administrative privileges
  - Users with `disabled = true` in their profile are denied access irrespective of their role
  - Authentication with API key for service-to-service authentication automatically applies the `LED-Matrix` role

### Matrix
Permissions: CRU(D/N)
- C = Create
- R = Read
- U = Update (modify)
- D = Delete or N = Cancel

| Role / Asset   | Users                    | Vehicles                | Parking Spots | Reservations             |
|----------------|--------------------------|-------------------------|---------------|--------------------------|
| Staff          | ---- (except self: -RU-) | ---- (except own: CRUD) | R---          | R--- (except own: CRUN)  |
| Administrators | -RU-                     | CRUD                    | CRUD          | CRUN                     |
| LED-Matrix     | ----                     | ----                    | ----          | R--- (including vehicle) |
| Anonymous      | ----                     | ----                    | ----          | ----                     |

### Rules

#### Parking Spot
- A parking spot can be occupied by one car at the same time
- A parking spot can be set "unavailable" (including an optional "reason" comment) by
  administrative users for a specified duration (start date/time, end date/time).
  Parking spots are "available" by default. When a parking spot is set to "unavailable",
  existing reservations overlapping with the unavailability-window
  should be automatically cancelled and the reservations owner be notified.
  When a parking spot is set back to available, any optional `unavailabilityReason` needs to be automatically cleared
- A parking spot can have an EV-charger installed. This is indicated by setting `charger = true`
- A parking spot cannot be deleted if referenced by reservations

#### Reservations
- A parking spot can be reserved for one half-day or one full-day per date
- A half-day reservation is either midnight to noon or noon to midnight
- A user can reserve up to two weeks into the future
- A user cannot have more than 3 reservations per week
- A user cannot have more than one reservation per day
- A reservation cannot be deleted for history/tracking reasons, but set to cancelled instead
- The time window (start-time => end-time) of reservations must not overlap

#### Vehicles
- A vehicle cannot be deleted if referenced by reservations
- A vehicle license plate number must be unique

#### Users
- Users are auto-created after first successful login
- The user's Okta ID must be unique
- Each successive login updates the user data (first name, last name, email, username) from the session
- Users cannot be deleted, only set to disabled

### Technical
- All date/times must be persisted in the UTC / ISO 8601 full-time format

## LED-Matrix Displays
The controller for the displays can obtain the list of parking spots and any active reservations + vehicles by 
accessing the following endpoint: `/parking-spots/today`. The controller can then determine the content to show on
each parking spot's individual LED-Matrix.

### LED-Matrix states:
- No reservation today: show idle animation
- No active reservation, but a reservation later today: show a mix of idle animation with
  time countdown until next reservation
- Active reservation: show reserved state with license plate number

## User Interface

### Administrative Panels
Administrative panels (and backend APIs) are only accessible to users with role `administrators`.

These panels have commonalities that should be implemented generically.
- List view
  - Filter (server-side)
  - Pagination (server-side)
  - Sorting (server-side)
  - Bulk checkboxes (for Delete/Cancel for example)
  - Actions (Create, View, Edit, Delete/Cancel)
- Detail view
  - Actions (Edit, Delete/Cancel)

Specific panels leveraging the above:
- User management
  - Users cannot be deleted, but set `disabled`
  - Users cannot be created. They are auto-created by the backend upon successful Okta login
- Vehicle management
  - Vehicles cannot be deleted if referenced by existing reservations
- Parking Spot management
  - Parking spots cannot be deleted if referenced by existing reservations 
- Reservation management
  - Reservations cannot be deleted, just cancelled 

### End User Flow
Staff reservation flow as per Miro-board and the Rules specified above.

Additionally:
- Parking spot visualization should include an icon if it supports EV-charging
- The UI should make an intelligent pre-selection of an available parking spot:
  - If the vehicle is an EV, select a parking spot with charger available
  - If the vehicle IS NOT an EV, select a parking spot without charger. If none is available, fall back to one with
    charger.

## Payment
In the interest of UX, the reservations could be made without directly paying while reserving.

Instead, reservations should be billed asynchronously. For example, the first of every month, an async job tabulates
the reservations of the past month and sends a TWINT QR-Code invoice to users.
