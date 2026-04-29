# API Versioning & Deprecation Strategy

To ensure stability for our frontend applications, mobile clients, and B2B integrators, the Frugal Fortress adheres to a strict API versioning and deprecation policy.

## 1. URI Versioning
All REST endpoints are prefixed with the API version (e.g., `/api/v1/`).
*   **Current Version:** `v1`
*   We do not use header-based or query-based versioning, as URI versioning provides the most explicit caching and routing boundaries at the CDN/Edge layer.

## 2. Non-Breaking Changes (Allowed in current version)
The following changes will be made to `/api/v1/` without bumping the version:
*   Adding new endpoints.
*   Adding new optional fields to request payloads.
*   Adding new fields to response payloads.
*   Changing the order of fields in a JSON response.

*Clients must be designed to ignore unrecognized fields in JSON responses.*

## 3. Breaking Changes (Requires new version)
The following changes require bumping the API to `/api/v2/`:
*   Removing or renaming an endpoint.
*   Removing or renaming a field in a response payload.
*   Changing the data type of a field (e.g., from `integer` to `string`).
*   Adding a new *required* field to a request payload.

## 4. Deprecation Policy (Sunset)
When an endpoint or an entire API version is scheduled for removal:
1.  **Notice:** We will add the `Deprecation: true` HTTP header to the response.
2.  **Sunset Header:** We will include the `Sunset: <HTTP-date>` header indicating the exact date the endpoint will stop functioning.
3.  **Grace Period:** A deprecated API version will be supported for a minimum of **6 months** after the release of the new version to allow clients sufficient time to migrate.