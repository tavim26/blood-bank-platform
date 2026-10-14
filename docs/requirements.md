# Requirements Specification – Blood Bank Platform

**Version:** 0.1 (draft)
**Status:** Phase 0 – Design

## 1. Introduction

### 1.1 Purpose
This document defines the functional and non-functional requirements, business rules and MVP scope of the Blood Bank Platform.

### 1.2 Scope
The platform supports the operations of a hospital blood bank:
- donor registration, eligibility screening and appointment scheduling
- recording donations and tracking each resulting blood unit through its lifecycle
- handling blood requests from hospital wards, with compatibility-aware allocation
- notifications, reporting and audit logging

Out of scope: clinical laboratory software, patient medical records, transfusion administration, billing.

### 1.3 Glossary

| Term | Definition |
|---|---|
| Donation center | Physical location where donations take place |
| Blood unit | A single bag of a blood component, tracked individually |
| Component | Type of blood product: red blood cells (RBC), plasma, platelets |
| Quarantine | State of a unit awaiting screening results |
| Deferral | Temporary or permanent period during which a donor may not donate |
| Screening | Laboratory tests required before a unit can be used |
| FEFO | First Expired, First Out – units closest to expiry are allocated first |

## 2. Actors and Roles

| Role | Description |
|---|---|
| **Donor** | Self-registered person who donates blood |
| **Donation Staff** | Blood bank personnel: reviews eligibility, records donations, manages inventory, processes requests |
| **Hospital Staff** | Ward personnel (e.g. doctors, nurses) who submit blood requests |
| **Administrator** | Manages staff accounts, centers, slots and settings; has read access to reports and audit logs |

Only donors can self-register. All staff and administrator accounts are created by an administrator. The first administrator is created through a CLI command or seed script.

## 3. Domain Model

### 3.1 Main entities
- **User** – credentials, role, account status
- **DonorProfile** – personal data, national ID (CNP), blood group (with confirmation status), deferral information
- **DonationCenter** – name, address, opening hours
- **TimeSlot** – center, start time, capacity
- **Appointment** – donor, time slot, status
- **Questionnaire** – donor's answers, automatic evaluation result, staff decision
- **Donation** – donor, appointment, collected volume, date, staff member
- **BloodUnit** – donation, component, blood group, volume, collection date, expiry date, status
- **BloodRequest** – ward, requester, patient blood group, component, quantity, urgency, status
- **Allocation** – link between a request and the units reserved or issued for it
- **Notification** – recipient, type, message, read status
- **AuditLog** – actor, action, entity, timestamp, details

### 3.2 State machines

**Appointment**
`BOOKED → CHECKED_IN → COMPLETED`
`BOOKED → CANCELLED` (by donor or staff)
`BOOKED → NO_SHOW` (set automatically or by staff)

**BloodUnit**
`QUARANTINE → AVAILABLE → RESERVED → ISSUED`
`QUARANTINE → DISCARDED` (failed screening)
`AVAILABLE / RESERVED → EXPIRED` (set automatically)
`RESERVED → AVAILABLE` (reservation released)
`AVAILABLE → DISCARDED` (damaged or other reason)

**BloodRequest**
`SUBMITTED → APPROVED → PARTIALLY_FULFILLED → FULFILLED`
`SUBMITTED → REJECTED`
`SUBMITTED / APPROVED → CANCELLED`

Every state transition is validated on the server; invalid transitions are rejected.

## 4. Functional Requirements

Priority: **MVP** = required for the first complete release; **Later** = planned after MVP.

### 4.1 Authentication and accounts

| ID | Requirement | Priority |
|---|---|---|
| FR-AUTH-01 | Donors can register with email and password | MVP |
| FR-AUTH-02 | Users can log in and log out; sessions use access and refresh tokens | MVP |
| FR-AUTH-03 | Every API endpoint enforces role-based access control | MVP |
| FR-AUTH-04 | Administrators can create, deactivate and reactivate staff accounts | MVP |
| FR-AUTH-05 | Users can reset their password via email | Later |
| FR-AUTH-06 | Email verification at registration | Later |
| FR-AUTH-07 | Two-factor authentication for staff and administrators | Later |

### 4.2 Donor profile

| ID | Requirement | Priority |
|---|---|---|
| FR-DON-01 | Donors can view and edit their profile | MVP |
| FR-DON-02 | The CNP is validated (format and checksum); date of birth and sex are derived from it | MVP |
| FR-DON-03 | Self-declared blood group is marked as unconfirmed until confirmed by staff | MVP |
| FR-DON-04 | Donors can see their donation history and next eligible donation date | MVP |

### 4.3 Eligibility questionnaire

| ID | Requirement | Priority |
|---|---|---|
| FR-ELG-01 | Donors complete a pre-donation questionnaire before booking | MVP |
| FR-ELG-02 | The system evaluates answers automatically: ELIGIBLE, DEFERRED (with date) or NEEDS_REVIEW | MVP |
| FR-ELG-03 | Staff make the final eligibility decision at check-in | MVP |
| FR-ELG-04 | Staff can record a temporary or permanent deferral | MVP |
| FR-ELG-05 | Administrators can configure questions and evaluation rules | Later |

### 4.4 Appointments

| ID | Requirement | Priority |
|---|---|---|
| FR-APP-01 | Administrators define time slots per center, each with a capacity | MVP |
| FR-APP-02 | Eligible donors can book an available slot | MVP |
| FR-APP-03 | Donors can cancel an appointment up to a configurable time before it | MVP |
| FR-APP-04 | Staff can check in donors and mark no-shows | MVP |
| FR-APP-05 | A reminder is sent before the appointment | MVP |
| FR-APP-06 | Appointments can be exported to a calendar (.ics) | Later |

### 4.5 Donations and blood units

| ID | Requirement | Priority |
|---|---|---|
| FR-UNIT-01 | Staff record a donation for a checked-in donor, including collected volume | MVP |
| FR-UNIT-02 | A donation produces one or more blood units, by component | MVP |
| FR-UNIT-03 | New units start in QUARANTINE, with an expiry date computed from the component | MVP |
| FR-UNIT-04 | Staff mark screening as passed (unit becomes AVAILABLE) or failed (unit becomes DISCARDED) | MVP |
| FR-UNIT-05 | A failed screening flags the donor for medical follow-up | MVP |
| FR-UNIT-06 | The first confirmed screening confirms the donor's blood group | MVP |
| FR-UNIT-07 | Staff can view inventory filtered by blood group, component, status and expiry | MVP |
| FR-UNIT-08 | Units can be transferred between donation centers | Later |

### 4.6 Hospital blood requests

| ID | Requirement | Priority |
|---|---|---|
| FR-REQ-01 | Hospital staff create requests: patient blood group, component, number of units, urgency, required-by date | MVP |
| FR-REQ-02 | Donation staff approve or reject requests | MVP |
| FR-REQ-03 | The system proposes compatible units following FEFO | MVP |
| FR-REQ-04 | Staff reserve units for a request and then mark them as issued | MVP |
| FR-REQ-05 | Hospital staff can follow the status of their requests | MVP |
| FR-REQ-06 | Reservations not issued within a configurable period are released automatically | Later |

### 4.7 Background processing and notifications

| ID | Requirement | Priority |
|---|---|---|
| FR-JOB-01 | A daily job marks expired units as EXPIRED | MVP |
| FR-JOB-02 | Staff are alerted when stock for a blood group and component falls below a configurable threshold | MVP |
| FR-JOB-03 | Users receive in-app notifications for relevant events | MVP |
| FR-JOB-04 | Key notifications are also sent by email | MVP |
| FR-JOB-05 | In critical shortages, staff can send an emergency call to eligible donors of a given blood group | Later |

### 4.8 Reporting and audit

| ID | Requirement | Priority |
|---|---|---|
| FR-REP-01 | Dashboard with current stock per blood group and component, and units close to expiry | MVP |
| FR-REP-02 | All state changes and administrative actions are recorded in the audit log automatically | MVP |
| FR-REP-03 | Administrators can browse and filter the audit log | MVP |
| FR-REP-04 | Statistical reports (donations, requests, expired units) exportable to CSV/PDF | Later |
| FR-REP-05 | Donation certificate available to donors as PDF | Later |
| FR-REP-06 | Real-time dashboard updates | Later |
| FR-REP-07 | Demand and stock forecasting per blood group (separate service) | Later |

## 5. Business Rules

All numeric values are configurable defaults and must be validated against current regulations before real-world use.

| ID | Rule |
|---|---|
| BR-01 | Donors must be between 18 and 65 years old, computed from the date of birth |
| BR-02 | Donors must meet a minimum body weight (configurable) |
| BR-03 | Minimum interval between whole blood donations: 3 months for men, 4 months for women |
| BR-04 | A donor can have at most one active (BOOKED) appointment |
| BR-05 | Appointments cannot be booked in the past or in a slot that is full |
| BR-06 | Recent tattoos or piercings result in a temporary deferral (configurable period, default 6 months) |
| BR-07 | Shelf life depends on the component and is configurable per component |
| BR-08 | Only AVAILABLE units can be reserved; only RESERVED units can be issued |
| BR-09 | A unit can belong to at most one active allocation; concurrent reservations must be prevented |
| BR-10 | Allocation follows FEFO: compatible units closest to expiry are proposed first |
| BR-11 | Red blood cell compatibility (recipient → acceptable donor groups): see table below |
| BR-12 | MVP simplification: plasma and platelets are allocated only to the identical blood group |
| BR-13 | Collected volume must fall within a configurable range |
| BR-14 | Expired units cannot be reserved or issued |

**Red blood cell compatibility (BR-11)**

| Recipient | Compatible donor groups |
|---|---|
| O- | O- |
| O+ | O+, O- |
| A- | A-, O- |
| A+ | A+, A-, O+, O- |
| B- | B-, O- |
| B+ | B+, B-, O+, O- |
| AB- | AB-, A-, B-, O- |
| AB+ | all groups |

When several compatible groups are available, units of the identical group are