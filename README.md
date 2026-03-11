# STAC API - Multi-Tenant Catalogs Endpoint Extension

- **Title:** Multi-Tenant Catalogs Endpoint
- **Conformance Classes:**
  - `https://api.stacspec.org/v1.0.0/core` (required)
  - `https://api.stacspec.org/v1.0.0-beta.3/multi-tenant-catalogs` (required)
  - `https://api.stacspec.org/v1.0.0-beta.3/multi-tenant-catalogs/transaction` (optional)
  - `https://api.stacspec.org/v1.0.0-rc.2/children` (recommended)
- **Scope:** STAC API - Core
- **Extension Maturity Classification:** Proposal
- **Dependencies:**
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/blob/main/core)
  - [STAC API - Collections](https://github.com/radiantearth/stac-api-spec/tree/main/ogcapi-features)
  - [STAC API - Children](https://github.com/stac-api-extensions/children)
  - [STAC API - Transaction](https://github.com/radiantearth/stac-api-spec/tree/main/ogcapi-features/extensions/transaction) (Reference pattern)
- **Owner**: @jonhealy1

## Introduction

This extension introduces a **Recursive Catalog Endpoint** (`/catalogs`) to the STAC API, enabling a **Multi-Tenant** architecture.

It adds a dedicated registry for **logical sub-catalogs**, allowing a single API instance to serve multiple, distinct catalog trees. Unlike the standard flat STAC structure (`Root -> Collections`), this extension enables a deep, **Recursive Hierarchy**, where catalogs can contain nested sub-catalogs for unlimited organizational depth (e.g., `Provider -> Theme -> Year -> Project`).

While technically "Multi-Tenant" (capable of hosting isolated providers), this architecture inherently supports **Virtual Organization**. It allows Collections to be shared across multiple catalogs simultaneously (Poly-hierarchy), enabling users to create curated "playlists," semantic themes, or project-specific views without duplicating the underlying data.


### The Management Plane (Optional)
This extension is not just for *viewing* hierarchy; it provides an **Optional Transactional Management Plane** to build it. 

APIs designed strictly for public, read-only discovery SHOULD NOT implement the `/transaction` conformance class or expose the `POST`, `PUT`, and `DELETE` endpoints. If an implementation chooses to support the Management Plane, it MUST advertise the `.../multi-tenant-catalogs/transaction` conformance class and define endpoints to:
* **Create** arbitrary catalog structures.
* **Link** existing collections to multiple parents (Poly-hierarchy).
* **Update** metadata and organization dynamically.

### Safety-First Architecture
A core tenet of this extension is **Data Safety**. It strictly separates "Organization" from "Data."

* **The `/catalogs` endpoints are for Organization:** Operations here (like deleting a catalog) are **Non-Destructive**. You can disband a catalog or unlink a collection, and the extension guarantees that **no actual data (Collections or Items) is ever deleted**. If a resource is unlinked from its last parent, it is automatically adopted by the Root Catalog to prevent data loss.
* **The `/collections` endpoints are for Data:** Actual destruction of data is reserved for the core STAC API endpoints.

**Note on Dynamic Linking:** To ensure data consistency, support poly-hierarchy, and accommodate flat URL routing, implementations SHOULD NOT persist static `rel="child"` or `rel="parent"` link objects in the database. 

Instead, implementations SHOULD maintain a backend mapping of parent-child relationships (e.g., an array of `parent_ids` on the catalog record) and dynamically generate the `rel="parent"`, `rel="child"`, and `rel="related"` HATEOAS links at runtime based on the requested endpoint.

## Endpoints

### Discovery (Read-Only)
| Method | URI | Description |
| :--- | :--- | :--- |
| `GET` | `/catalogs` | **The Registry.** Lists all available sub-catalogs. |
| `GET` | `/catalogs/{catalogId}` | **Sub-Catalog Root.** Acts as the Landing Page for the provider. |
| `GET` | `/catalogs/{catalogId}/conformance` | Conformance classes specific to this sub-catalog. |
| `GET` | `/catalogs/{catalogId}/queryables` | Filter Extension. Lists fields available for filtering in this sub-catalog. |
| `GET` | `/catalogs/{catalogId}/children` | **Children.** Lists all child resources (Catalogs and Collections). Supports filtering via `?type=Catalog` or `?type=Collection`. |
| `GET` | `/catalogs/{catalogId}/catalogs` | **Sub-Catalogs List.** Lists only the child catalogs of this catalog (for hierarchy traversal). |
| `GET` | `/catalogs/{catalogId}/collections` | Lists collections belonging to this sub-catalog. |
| `GET` | `/catalogs/{catalogId}/collections/{collectionId}` | Gets a specific collection definition. |
| `GET` | `/catalogs/{catalogId}/collections/{collectionId}/items` | **Item Search.** Fetches items from this specific collection. |
| `GET` | `/catalogs/{catalogId}/collections/{collectionId}/items/{itemId}` | Gets a single specific item. |

### Transactions (Management)
These endpoints allow for the dynamic creation and deletion of the federation structure. 
**Note:** These endpoints are OPTIONAL and MUST only be exposed if the API advertises the `.../multi-tenant-catalogs/transaction` conformance class.

| Method | URI | Description |
| :--- | :--- | :--- |
| `POST` | `/catalogs` | **Create Root Catalog.** Registers a new top-level catalog. |
| `PUT`  | `/catalogs/{catalogId}` | **Update Catalog.** Updates metadata (Title, Description). **Safety: Preserves existing hierarchy links.** |
| `DELETE` | `/catalogs/{catalogId}` | **Disband Catalog.** Removes a sub-catalog. **Safety: Never deletes linked collections.** |
| `POST` | `/catalogs/{catalogId}/catalogs` | **Link or Create Sub-Catalog.** Links an existing catalog OR creates a new one. |
| `DELETE` | `/catalogs/{catalogId}/catalogs/{subCatalogId}` | **Unlink Sub-Catalog.** Removes the link to the sub-catalog. **Safety: Does not delete the sub-catalog.** |
| `POST` | `/catalogs/{catalogId}/collections` | **Link or Create Collection.** Links an existing collection OR creates a new one. |
| `DELETE` | `/catalogs/{catalogId}/collections/{collectionId}` | **Unlink Collection.** Removes the link from the parent catalog. **Safety: Never deletes the collection data.** |

## Poly-Hierarchy (Multi-Parenting)

This extension explicitly supports **Poly-hierarchy**, allowing a single STAC Collection or Catalog to belong to **multiple Catalogs simultaneously**.

Unlike a standard file system where a folder can only live in one path, this architecture allows for logical grouping across different dimensions without duplicating data. 

* **Example:** A `Sentinel-2` collection can be linked as a child of the `USGS` Catalog (Provider) AND the `Optical-Data` Catalog (Theme).
* **Contextual Navigation (Scoped Route):** When accessing a Collection via a scoped endpoint (e.g., `/catalogs/{id}/collections/{col_id}`), the API MUST generate exactly one `rel="parent"` link pointing exclusively back to that specific `{catalogId}` to preserve the user's current contextual breadcrumb trail in UI clients. Alternative parents in the poly-hierarchy MUST be exposed as `rel="related"` links.
* **Global Discovery (Global Route):** When accessing a Collection via the global root endpoint (`/collections/{collectionId}`), the API MUST generate exactly one `rel="parent"` link pointing to the Global Root (`/`). To expose the poly-hierarchy, the API MUST include a `rel="related"` link for every Catalog that claims this collection as a child, provided the current authenticated user has read access to those parent catalogs.
  * If the API implements Role-Based Access Control (RBAC), it MUST filter out links to restricted catalogs to prevent information disclosure.

## Transaction Behavior

Implementations supporting the Transaction endpoints MUST adhere to the following behaviors:

### 1. Catalog Creation (`POST /catalogs`)
* **Body:** Accepts a standard [STAC Catalog](https://github.com/radiantearth/stac-spec/blob/master/catalog-spec/catalog-spec.md) JSON object.
* **Behavior:** The API creates the Catalog resource and makes it available in the `/catalogs` registry list.

### 2. Catalog Update (`PUT /catalogs/{id}`)
* **Body:** Accepts a standard STAC Catalog JSON object.
* **Behavior:** Updates the metadata (Title, Description, etc.) of the catalog.
* **Safety:** This operation MUST NOT modify the structural links (`parent_ids`) of the catalog unless explicitly handled, ensuring the catalog remains in its current hierarchy.

### 3. Sub-Catalog Creation (`POST /catalogs/{id}/catalogs`)

This endpoint supports two distinct modes of operation: **Creation** and **Linking**.

#### Mode A: Creation (Full Body)
* **Body:** A full [STAC Catalog](https://github.com/radiantearth/stac-spec/blob/master/catalog-spec/catalog-spec.md) JSON object.
* **Condition:** The `id` in the body does not currently exist in the database.
* **Behavior:** Creates a new Catalog and links it as a child of `{catalogId}`.

#### Mode B: Linking (Reference Only)
* **Body:** A JSON object containing only the `id`. Example: `{"id": "existing-catalog-id"}`.
* **Condition:** The `id` already exists in the database.
* **Behavior:**
    1.  **Establishes Reciprocal Links:** Adds `{catalogId}` to the sub-catalog's internal parent list (which dynamically resolves to `rel="parent"` or `rel="related"` links at read-time).
    2.  Returns `200 OK` to indicate a successful link.
    3.  **Error Handling:** If the `id` does not exist when using this minimal payload, the API MUST return `404 Not Found`.
    4.  **Cycle Prevention:** Implementations SHOULD reject links that create a circular reference (e.g., linking a parent as a child of itself).

### 4. Catalog Deletion (`DELETE /catalogs/{id}`)
* **Behavior (Disband):**
    * The Catalog object `{id}` is deleted from the database.
    * All Child Collections **AND Child Sub-Catalogs** linked to this catalog are **Unlinked**.
    * **Adoption:** If an unlinked child (Collection or Catalog) has no other parents, it MUST be automatically adopted by the Root Catalog to ensure data/structure is preserved.
    * **Constraint:** This operation MUST NOT delete Collection or Item data.
 
### 5. Sub-Catalog Unlinking (`DELETE /catalogs/{id}/catalogs/{subId}`)
* **Behavior (Unlink):**
    * The Sub-Catalog `{subId}` is **Unlinked** from the parent Catalog `{id}`.
    * **Safety:** The Sub-Catalog resource itself is **NOT deleted**.
    * **Adoption:** If the Sub-Catalog has no other parents (orphaned), it MUST be automatically adopted by the Root Catalog to ensure it remains discoverable.
    * **Constraint:** This operation only removes the specific hierarchical link between `{id}` and `{subId}`.

### 6. Scoped Collection Creation (`POST /catalogs/{id}/collections`)

This endpoint supports two distinct modes of operation: **Creation** and **Linking**.

#### Mode A: Creation (Full Body)
* **Body:** A full [STAC Collection](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md) JSON object.
* **Condition:** The `id` in the body does not currently exist in the database.
* **Behavior:** Creates a new Collection and links it as a child of `{catalogId}`.

#### Mode B: Linking (Reference Only)
* **Body:** A JSON object containing only the `id`. Example: `{"id": "existing-collection-id"}`.
* **Condition:** The `id` already exists in the database.
* **Behavior:**
    1.  Does **not** overwrite the existing collection metadata.
    2.  **Establishes Reciprocal Links:** Adds `{catalogId}` to the collection's parent list, AND adds a `rel="child"` link to the Catalog pointing to the collection.
    3.  Returns `200 OK` to indicate a successful link (vs `201 Created` for new resources).
    4.  **Error Handling:** If the `id` does not exist when using this minimal payload, the API MUST return `404 Not Found`.

### 7. Scoped Collection Deletion (`DELETE /catalogs/{catalogId}/collections/{collectionId}`)

This operation is strictly an **Unlink** action. It modifies the hierarchy but **never destroys data**.

* **Behavior:**
    1.  Removes `{catalogId}` from the collection's list of parents.
    2.  Removes the `rel="child"` link from the Catalog `{catalogId}`.
    3.  **Adoption Logic:** If the collection has **no other parents** after this operation (i.e., it was only linked to this one catalog), it MUST be automatically linked to the **Root Catalog**. This ensures no data becomes "orphaned" or undiscoverable.
* **Response:** Returns `204 No Content`.
* **Safety Guarantee:** This endpoint **MUST NOT** delete the Collection resource or its Items from the database. To permanently destroy a collection, the client must use the core `DELETE /collections/{collectionId}` endpoint.

## Link Relations

Proper linking is critical for clients to navigate the federation structure.

### 1. The Global Root (`/`)
This is the entry point.
- `rel="catalogs"`: MUST point to the `/catalogs` endpoint (the registry).
- `rel="service-desc"`: Points to the OpenAPI definition.

### 2. The Sub-Catalog (`/catalogs/{catalogId}`)
This resource acts as the Landing Page for the provider or a nested sub-folder.
* `rel="self"`: MUST point to `/catalogs/{catalogId}`.
* `rel="parent"`: MUST point to **exactly one** immediate parent Catalog that this sub-catalog was accessed through. If it is a top-level sub-catalog, it MUST point to the Global Root (`/`).
* `rel="related"`: If the catalog is part of a poly-hierarchy (has multiple parents), the API MUST include this link for all *other* parent catalogs to expose the broader graph.
* `rel="root"`: MUST point to the Global Root (`/`) to maintain a single navigation tree.
* `rel="child"`: MUST point to any immediate Sub-Catalogs (`/catalogs/{subId}`) AND any linked Collections (`/catalogs/{catalogId}/collections/{collectionId}`).

### 3. The Global Collection (`/collections/{collectionId}`)
This resource represents a Collection accessible via the standard STAC core endpoint (flat, global discovery).
* `rel="self"`: MUST point to `/collections/{collectionId}`.
* `rel="parent"`: MUST point to **exactly one** parent, which is the Global Root (`/`).
* `rel="related"`: MUST include a link for every Sub-Catalog that claims this collection as a child, exposing the poly-hierarchy.
* `rel="root"`: MUST point to the Global Root (`/`).

### 4. The Scoped Collection Endpoints (`/catalogs/{catalogId}/collections/{collectionId}/*`)
This resource, and all of its sub-resources, represents a Collection within the context of a specific Catalog. The `rel="parent"` link is locked to the scoped path to preserve contextual breadcrumb navigation.
* `rel="self"`: MUST point to `/catalogs/{catalogId}/collections/{collectionId}`.
* `rel="parent"`: MUST point **exclusively** to `/catalogs/{catalogId}` (the specific parent through which the user navigated). It MUST NOT list other parents as `rel="parent"`, as this breaks UI breadcrumbs.
* `rel="related"`: MAY be included to expose the collection's alternative parents in the poly-hierarchy.
* `rel="alternate"`: MAY be provided to point to the corresponding `/collections/{collectionId}/*` endpoints, and vice versa.

> [!NOTE]
> **Contextual vs. Global Navigation:** The distinction between scoped and global endpoints is critical for STAC Browser and other UI clients. Scoped endpoints lock the breadcrumb trail to a single parent for clarity, while global endpoints expose the full poly-hierarchy graph. Implementations MUST respect this distinction when generating `rel="parent"` and `rel="related"` links.

> [!NOTE]
> All sub-resources can be considered, accordingly to other STAC API extensions that are implemented.
> For example, if the [Filter](https://github.com/stac-api-extensions/filter) extension
> is implemented and supports [Queryables](https://github.com/stac-api-extensions/filter#queryables),
> then `rel="alternate"` links MAY be included in corresponding responses as well:
> - `/catalogs/{catalogId}/collections/{collectionId}/queryables`
> - `/collections/{collectionId}/queryables`
> - `/queryables?collections={collectionId}`

> [!NOTE]
> The `rel="alternate"` link is optional to allow implementation omitting the reference if such endpoint should be protected
> and hidden from clients. Otherwise, it is RECOMMENDED to provide this link for better interoperability and discoverability
> of clients that are typically aware of the core STAC API structure.

## Response Examples

### 1. The Registry List (`GET /catalogs`)

This endpoint returns a JSON object structurally similar to a standard `/collections` response, but it contains a list of **Catalog** objects in a `catalogs` array.

```json
{
  "catalogs": [
    {
      "id": "catalog3",
      "type": "Catalog",
      "title": "Nested Sub-Catalog",
      "description": "A nested catalog under catalog2.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/catalog3" },
        { "rel": "root", "href": "https://api.example.com/" },
        { "rel": "parent", "href": "https://api.example.com/catalogs/catalog2" },
        { "rel": "data", "href": "https://api.example.com/catalogs/catalog3/collections" }
      ]
    },
    {
      "id": "esa-sentinel",
      "type": "Catalog",
      "title": "ESA Sentinel",
      "description": "Sentinel collections provided by ESA.",
      "stac_version": "1.0.0",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/esa-sentinel" },
        { "rel": "root", "href": "https://api.example.com/" },
        { "rel": "data", "href": "https://api.example.com/catalogs/esa-sentinel/collections" }
      ]
    }
  ],
  "links": [
    {
      "rel": "self",
      "href": "https://api.example.com/catalogs"
    },
    {
      "rel": "root",
      "href": "https://api.example.com/"
    }
  ]
}
```

### 2. The Global Root (`GET /`)

The global root remains a standard STAC Landing Page. Note the addition of the `rel="catalogs"` link.

```json
{
  "stac_version": "1.0.0",
  "type": "Catalog",
  "id": "stac-api",
  "title": "Standard STAC API with Multi-Tenancy",
  "description": "A standard STAC API that also supports multi-tenant catalogs.",
  "conformsTo": [
    "https://api.stacspec.org/v1.0.0/core",
    "https://api.stacspec.org/v1.0.0-beta.3/multi-tenant-catalogs",
    "https://api.stacspec.org/v1.0.0-beta.3/multi-tenant-catalogs/transaction"
  ],
  "links": [
    {
      "rel": "self",
      "type": "application/json",
      "href": "https://api.example.com/"
    },
    {
      "rel": "service-desc",
      "type": "application/vnd.oai.openapi+json;version=3.0",
      "href": "https://api.example.com/api"
    },
    {
      "rel": "data",
      "type": "application/json",
      "href": "https://api.example.com/collections",
      "title": "Global Collections List"
    },
    {
      "rel": "catalogs",
      "type": "application/json",
      "href": "https://api.example.com/catalogs",
      "title": "Multi-Tenant Catalogs Registry"
    }
  ]
}
```

### 3. The Children Endpoint (GET /catalogs/{id}/children)

This endpoint returns a list of both child Catalogs and child Collections.

```json
{
  "children": [
    {
      "id": "sub-catalog-1",
      "type": "Catalog",
      "title": "A nested sub-catalog",
      "links": [
        { "rel": "self", "href": "https://api.example.com/catalogs/sub-catalog-1" }
      ]
    },
    {
      "id": "collection-1",
      "type": "Collection",
      "title": "A child collection",
      "links": [
        { "rel": "self", "href": "https://api.example.com/collections/collection-1" }
      ]
    }
  ],
  "links": [
    {
      "rel": "self",
      "href": "https://api.example.com/catalogs/c1/children"
    },
    {
      "rel": "next",
      "href": "https://api.example.com/catalogs/c1/children?token=..."
    }
  ]
}
```

## Optional Capabilities

### Filter Extension & Queryables

This extension reserves the path `/catalogs/{catalogId}/queryables` to support the **STAC Filter Extension** within a sub-catalog context.

However, implementation of this endpoint is **OPTIONAL**.

* A sub-catalog **MUST** only expose this endpoint if it advertises conformance to the Filter Extension URI (e.g., `https://api.stacspec.org/v1.0.0-rc.2/filter`) in the Sub-Catalog Landing Page (`/catalogs/{catalogId}`).
* If implemented, the queryables response must be scoped specifically to that sub-catalog.
