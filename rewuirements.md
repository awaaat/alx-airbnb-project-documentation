# Airbnb Clone Backend Requirement Specifications

## Overview
This document outlines the technical and functional requirements for the key backend features of the Airbnb Clone project. The features covered include User Authentication, Property Management, and Booking System. Each section details API endpoints, input/output specifications, validation rules, and performance criteria to ensure a scalable, secure, and efficient system.

---

## 1. User Authentication

### 1.1 Functional Requirements
- **User Registration**: Allow users to sign up as guests or hosts using email and password or OAuth (Google, Facebook).
- **User Login**: Enable users to log in with email/password or OAuth, returning a JWT for session management.
- **Role-Based Access**: Differentiate permissions for guests, hosts, and admins using role-based access control (RBAC).
- **Session Management**: Use JWT to manage user sessions securely.

### 1.2 Technical Requirements

#### API Endpoints

##### `POST /api/auth/register`
- **Description**: Register a new user (guest or host).
- **Input**:
  - `email`: String (required, valid email format)
  - `password`: String (required, min 8 characters, must include 1 uppercase, 1 lowercase, 1 number)
  - `role`: String (required, either `"guest"` or `"host"`)
  - `first_name`: String (required, max 50 characters)
  - `last_name`: String (required, max 50 characters)
- **Output**:
  - Success: `201 Created`
    ```json
    { "message": "User registered successfully", "user_id": "uuid" }
    ```
  - Error: `400 Bad Request`
    ```json
    { "error": "Email already exists" }
    ```
- **Validation Rules**:
  - Email must be unique
  - Password must meet complexity
  - Role must be `"guest"` or `"host"`

---

##### `POST /api/auth/login`
- **Description**: Authenticate a user and return a JWT.
- **Input**:
  - `email`: String (required)
  - `password`: String (required)
- **Output**:
  - Success: `200 OK`
    ```json
    { "message": "Login successful", "token": "jwt_token", "user": { "id": "uuid", "email": "string", "role": "string" } }
    ```
  - Error: `401 Unauthorized`
    ```json
    { "error": "Invalid credentials" }
    ```
- **Validation Rules**:
  - Must match existing user credentials

---

##### `POST /api/auth/oauth`
- **Description**: Authenticate a user via OAuth.
- **Input**:
  - `provider`: `"google"` or `"facebook"` (required)
  - `access_token`: String (required)
- **Output**:
  - Success: `200 OK`
    ```json
    { "message": "Login successful", "token": "jwt_token", "user": { "id": "uuid", "email": "string", "role": "string" } }
    ```
  - Error: `401 Unauthorized`
    ```json
    { "error": "Invalid OAuth token" }
    ```
- **Validation Rules**:
  - Token must be valid
  - Auto-register if user does not exist

---

#### Database Schema: `Users`
| Field       | Type              | Constraints                            |
|-------------|-------------------|----------------------------------------|
| id          | UUID              | Primary Key                            |
| email       | VARCHAR(255)      | Unique, Not Null                       |
| password    | VARCHAR(255)      | Hashed, Not Null for email users       |
| role        | ENUM              | 'guest', 'host', 'admin'               |
| first_name  | VARCHAR(50)       | Not Null                               |
| last_name   | VARCHAR(50)       | Not Null                               |
| created_at  | TIMESTAMP         | Default: CURRENT_TIMESTAMP             |

---

### 1.3 Performance Criteria
- Login/register response time: < 500ms
- Handle 1,000 concurrent requests with < 1% failure
- JWT generation/validation: < 50ms
- Password hashing: bcrypt with cost factor 12

### 1.4 Security Requirements
- Passwords hashed with bcrypt
- JWT expiry: 24 hours, signed securely
- Rate limit: 5 failed logins/IP per 5 minutes
- HTTPS for all requests

---

## 2. Property Management

### 2.1 Functional Requirements
- **Add Listing**: Hosts can add listings
- **Edit/Delete Listing**: Hosts can update or remove listings
- **Search Listings**: Guests can search by location, price, guests, amenities (with pagination)

### 2.2 Technical Requirements

#### API Endpoints

##### `POST /api/properties`
- **Description**: Create a new listing (host only)
- **Input**:
  - `title`: String, max 100 chars
  - `description`: String, max 1000 chars
  - `location`: Object `{ city, country, lat, lng }`
  - `price_per_night`: Number, > 0
  - `amenities`: Array (optional)
  - `availability`: Array of `{ start, end }`
  - `images`: Array of S3 URLs
- **Output**:
  - Success: `201 Created`
    ```json
    { "message": "Property created", "property_id": "uuid" }
    ```
  - Error: `400 Bad Request`
    ```json
    { "error": "Invalid price" }
    ```
- **Validation**:
  - Authenticated host
  - Positive price
  - Valid availability dates

---

##### `PUT /api/properties/{id}`
- **Description**: Update listing (host only)
- **Input**: Same fields as POST (optional)
- **Output**:
  - Success: `200 OK`
    ```json
    { "message": "Property updated" }
    ```
  - Error: `404 Not Found`
- **Validation**:
  - Must be owner
  - Updated availability must not conflict with bookings

---

##### `DELETE /api/properties/{id}`
- **Description**: Delete listing (host only)
- **Output**:
  - Success: `200 OK`
    ```json
    { "message": "Property deleted" }
    ```
  - Error: `404 Not Found`
- **Validation**:
  - Must be owner
  - No active bookings

---

##### `GET /api/properties`
- **Description**: Search listings
- **Query Parameters**:
  - `location`, `price_min`, `price_max`, `guests`, `amenities[]`, `page`, `limit`
- **Output**:
  - Success: `200 OK`
    ```json
    { "properties": [...], "total": number, "page": number, "limit": number }
    ```
  - Error: `400 Bad Request`
- **Validation**:
  - Valid price range
  - Pagination limits

---

#### Database Schema: `Properties`
| Field           | Type          | Constraints                             |
|------------------|---------------|------------------------------------------|
| id              | UUID           | Primary Key                              |
| host_id         | UUID           | FK to Users, Not Null                    |
| title           | VARCHAR(100)   | Not Null                                 |
| description     | TEXT           | Not Null                                 |
| location        | JSONB          | Not Null (city, country, lat, lng)       |
| price_per_night | DECIMAL(10,2)  | Not Null                                 |
| amenities       | JSONB          | Default: []                              |
| availability    | JSONB          | Not Null                                 |
| images          | JSONB          | Not Null                                 |
| created_at      | TIMESTAMP      | Default: CURRENT_TIMESTAMP               |

---

### 2.3 Performance Criteria
- Creation/update: < 300ms
- Search: < 500ms for 10,000 listings (Redis cached)
- 500 concurrent search requests < 1% failure
- Pagination: < 200ms per page

### 2.4 Security Requirements
- Authenticated host-only actions
- Validate image URLs
- Rate limit: 100 property creations/hour/user

---

## 3. Booking System

### 3.1 Functional Requirements
- **Booking Creation**: Guests can book available listings
- **Cancellation**: Guests or hosts can cancel under policy
- **Status Tracking**: Track booking lifecycle

### 3.2 Technical Requirements

#### API Endpoints

##### `POST /api/bookings`
- **Description**: Create booking (guest only)
- **Input**:
  - `property_id`, `start_date`, `end_date`, `guests`
- **Output**:
  - Success: `201 Created`
    ```json
    { "message": "Booking created, proceed to payment", "booking_id": "uuid" }
    ```
  - Error: `400 Bad Request`
    ```json
    { "error": "Property not available" }
    ```
- **Validation**:
  - Dates in future
  - Available and valid property

---

##### `POST /api/bookings/{id}/pay`
- **Description**: Confirm booking via payment
- **Input**:
  - `payment_method`: `{ type, token }`
- **Output**:
  - Success: `200 OK`
    ```json
    { "message": "Booking confirmed" }
    ```
  - Error: `400 Bad Request`
- **Validation**:
  - Status must be `"pending"`
  - Valid payment token

---

##### `DELETE /api/bookings/{id}`
- **Description**: Cancel a booking
- **Output**:
  - Success: `200 OK`
    ```json
    { "message": "Booking canceled" }
    ```
  - Error: `404 Not Found`
- **Validation**:
  - Only guest or host
  - Must comply with cancellation policy

---

##### `GET /api/bookings/{id}`
- **Description**: Get booking details
- **Output**:
  - Success: `200 OK`
    ```json
    { "booking": { ... } }
    ```
  - Error: `404 Not Found`
- **Validation**:
  - Accessible to guest, host, or admin

---

#### Database Schema: `Bookings`
| Field        | Type      | Constraints                               |
|--------------|-----------|--------------------------------------------|
| id           | UUID      | Primary Key                                |
| property_id  | UUID      | FK to Properties, Not Null                 |
| guest_id     | UUID      | FK to Users, Not Null                      |
| start_date   | DATE      | Not Null                                   |
| end_date     | DATE      | Not Null                                   |
| guests       | INTEGER   | Not Null                                   |
| status       | ENUM      | 'pending', 'confirmed', 'canceled', 'completed' |
| created_at   | TIMESTAMP | Default: CURRENT_TIMESTAMP                 |

---

### 3.3 Performance Criteria
- Booking creation: < 400ms
- Payment processing: < 1 second
- 300 concurrent booking requests: < 1% failure
- Status update latency: < 200ms

### 3.4 Security Requirements
- Only guests can create bookings
- Guest or host can cancel
- Secure token processing over HTTPS
- Rate limit: 50 bookings/hour/user

---

## Conclusion
These specification guidelines define the technical foundation for the Airbnb Clone Backend, covering authentication, property management, and bookings. They emphasize security, scalability, and performance to ensure a production-ready application.

**Last Updated**: May 11, 2025
