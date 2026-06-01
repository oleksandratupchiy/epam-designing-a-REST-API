# REST API Design — Veterinary Clinic Management System

## 1. Introduction & Domain Overview
This document outlines the RESTful API design for a **Veterinary Clinic Management System**. The system serves as a hybrid platform that combines a **Catalogue** (managing owners, pets, veterinarians, and medical records) with a **Scheduling/Alerting System** (managing appointments, vaccination schedules, and reminders). The API is designed for client applications used by clinic administrators, veterinarians, and pet owners.

## 2. Functional Requirements
* **Catalogue Management:** The system must allow creating, retrieving, updating, and deleting records for Owners, Pets, Veterinarians, and Medications.
* **Scheduling:** Users must be able to book, update, and cancel medical appointments.
* **Alerting:** The system must track vaccination schedules and generate reminders for upcoming treatments.
* **Medical Records:** Veterinarians must be able to append clinical notes and link medications to a pet's profile.

## 3. Non-Functional Requirements
* **Performance:** API response time must be under 200ms for 95% of GET requests.
* **Scalability:** The system must support concurrent scheduling operations without data anomalies.
* **Security & Compliance:** Data must be protected via TLS 1.3. The API must comply with GDPR regarding owner personal data (right to be forgotten, data masking).
* **Availability:** 99.9% uptime target.

## 4. Entity Model
The domain consists of the following core entities:

* **Owner:** `id` (UUID), `firstName`, `lastName`, `email`, `phone`, `registeredAt`.
* **Pet:** `id` (UUID), `name`, `species` (e.g., Cat, Dog), `breed`, `dateOfBirth`, `ownerId`.
* **Veterinarian:** `id` (UUID), `fullName`, `specialization`, `licenseNumber`.
* **Appointment:** `id` (UUID), `petId`, `vetId`, `appointmentDate`, `status` (SCHEDULED, COMPLETED, CANCELLED), `reason`.
* **MedicalRecord:** `id` (UUID), `petId`, `vetId`, `visitDate`, `diagnosis`, `treatmentPlan`.

## 5. REST API Design (Richardson Maturity Model - Level 3)
The API strictly adheres to REST principles, utilizing appropriate HTTP methods, standard URI structures, and **HATEOAS** (Hypermedia as the Engine of Application State) for state transitions.

**Base URL:** `https://api.vetclinic.com/v1`

### 5.1. Pets Resource Collection
**Endpoint:** `GET /pets`
* **Description:** Retrieves a paginated list of pets.
* **Query Parameters:** `species`, `ownerId`, `page`, `size`, `sort`.
* **Response (200 OK):**
    ```json
    {
      "_embedded": {
        "pets": [
          {
            "id": "a1b2c3d4",
            "name": "Dunya",
            "species": "Cat",
            "breed": "Longhair",
            "ownerId": "f9e8d7c6",
            "_links": {
              "self": { "href": "/v1/pets/a1b2c3d4" },
              "owner": { "href": "/v1/owners/f9e8d7c6" },
              "appointments": { "href": "/v1/pets/a1b2c3d4/appointments" }
            }
          }
        ]
      },
      "page": { "size": 20, "totalElements": 1, "totalPages": 1, "number": 0 },
      "_links": {
        "self": { "href": "/v1/pets?page=0&size=20" }
      }
    }
    ```

**Endpoint:** `POST /pets`
* **Description:** Registers a new pet.
* **Request Body:** JSON containing `name`, `species`, `breed`, `ownerId`.
* **Response (201 Created):** Returns the created pet object with HATEOAS links. Includes `Location` header pointing to the new resource.

### 5.2. Single Pet Resource
**Endpoint:** `GET /pets/{id}`
* **Response (200 OK):** Returns pet details.
* **Response (404 Not Found):** If the pet does not exist.

**Endpoint:** `PATCH /pets/{id}`
* **Description:** Partially updates pet details.
* **Response (200 OK):** Returns updated pet details.

**Endpoint:** `DELETE /pets/{id}`
* **Description:** Removes a pet record.
* **Response (204 No Content):** Empty body on successful deletion.

### 5.3. Appointments Resource
**Endpoint:** `POST /appointments`
* **Description:** Books a new appointment.
* **Status Codes:** * `201 Created`: Successfully booked.
    * `409 Conflict`: If the requested time slot is already taken by the veterinarian.

## 6. Pagination
All collection endpoints returning multiple records (e.g., `/pets`, `/owners`, `/appointments`) implement **Offset-based pagination**.
* **Parameters:** `page` (0-indexed default), `size` (default 20, max 100).
* Responses include a metadata block detailing `totalElements`, `totalPages`, and `number`, along with HATEOAS `next` and `prev` navigation links.

## 7. Filtering & Sorting
Filtering is achieved via query parameters.
* **Example:** `GET /appointments?status=SCHEDULED&vetId=123`
* **Sorting:** Uses the `sort` parameter formatted as `field,direction`.
* **Example:** `GET /pets?sort=name,asc`

## 8. Error Handling
The API implements standardized error responses according to **RFC 7807 (Problem Details for HTTP APIs)**.

**Example Error Response (400 Bad Request):**
```json
{
  "type": "[https://api.vetclinic.com/errors/validation-error](https://api.vetclinic.com/errors/validation-error)",
  "title": "Invalid Request Parameters",
  "status": 400,
  "detail": "The 'appointmentDate' must be in the future.",
  "instance": "/v1/appointments",
  "invalidParams": [
    {
      "field": "appointmentDate",
      "reason": "Must not be a past date."
    }
  ]
}
```


## 9. Authentication & Authorization
* **Authentication:** The API relies on **OAuth 2.0** with **JSON Web Tokens (JWT)**. Clients must include the token in the `Authorization: Bearer <token>` header. If the token is missing or invalid, the API returns **`401 Unauthorized`**.
* **Authorization:** Role-Based Access Control (RBAC) is enforced at the endpoint level. If a user attempts to access a resource without proper permissions, the API returns **`403 Forbidden`**.
    * **ADMIN:** Full access to all resources.
    * **VET:** Can Read/Update medical records and view appointments.
    * **OWNER:** Can Read/Update their own profile, register pets, and book appointments.

## 10. Caching Strategy
To minimize database load and improve response times, caching is heavily utilized for read-heavy operations:
* **Reference Data** (e.g., Species/Breeds lists): Cached aggressively using `Cache-Control: max-age=86400, public`.
* **Resource Endpoints** (e.g., `GET /pets/{id}`): Utilizes `ETag` headers. If a client sends an `If-None-Match` header matching the current ETag, the server responds with **`304 Not Modified`**, saving bandwidth.
* **Non-Cached Operations:** `POST`, `PATCH`, `PUT`, and `DELETE` requests are never cached, as they mutate state.

## 11. Rate Limiting
To prevent abuse, rate limiting is applied globally per client IP and authenticated user token:
* **Standard Tier:** 100 requests per minute.
* If exceeded, the API responds with **`429 Too Many Requests`** and includes a `Retry-After` header indicating when to try again.

## 12. API Versioning
The API uses **URI Versioning** (e.g., `/api/v1/...`). This approach ensures that breaking changes (like payload restructuring) can be introduced in `/v2` without disrupting existing client applications integrating with `/v1`.
