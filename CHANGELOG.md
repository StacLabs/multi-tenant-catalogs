# Changelog

All notable changes to the "Multi-Tenant Catalogs" STAC API Extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [v1.0.0-beta.3] - 2026-03-05

### Added
- **Optional Transactions Conformance Class:** Explicitly separated the management plane (`POST`, `PUT`, `DELETE`) into an optional `https://api.stacspec.org/v1.0.0-beta.2/multi-tenant-catalogs/transaction` conformance class. This ensures public-facing APIs can safely implement the core extension as read-only by default without violating the specification.

### Changed
- **STAC Browser Compatibility:** Updated Link Relations section to explicitly support poly-hierarchy with multiple `rel="parent"` links for catalogs with multiple parents. Clarified that top-level sub-catalogs without other parents MUST point to the Global Root.
- **Dynamic Linking Specification:** Enhanced the "Safety-First Architecture" note to provide explicit implementation guidance: maintain a backend mapping of `parent_ids` and dynamically generate `rel="parent"` and `rel="child"` HATEOAS links at runtime, rather than persisting static link objects in the database.
- **Example JSON Response:** Updated the Registry List example to demonstrate nested catalog structure with proper parent-child relationships, showing what STAC Browser and other clients should expect when traversing true Directed Acyclic Graphs (DAGs).

## [v1.0.0-beta.2] - 2026-03-02

### Changed
- **Linking Logic:** Updated `POST /catalogs/{id}/collections` to support a **"Link by Reference"** payload (ID only). This allows clients to link existing collections to new catalogs without re-uploading the full metadata body.
- **Deletion Safety:** Explicitly defined `DELETE` operations on `/catalogs/...` endpoints as **"Unlink"** actions. Added normative language guaranteeing that these endpoints must never delete the underlying Collection or Item data from the database.
- **Linking Logic:** Updated `POST /catalogs/{id}/catalogs` to support the same "Link by Reference" pattern for sub-catalogs.

## [v1.0.0-beta.1] - 2026-01-11

### Added
- **Recursive Hierarchy:** Defined the `/catalogs/{id}/catalogs` endpoint to support infinite nesting of sub-catalogs.
- **Poly-Hierarchy:** Explicitly defined support for Collections belonging to multiple Catalogs simultaneously.
- **Transactional API:** Added `POST`, `PUT`, and `DELETE` endpoints for managing the catalog structure.
- **Safety Logic:** Defined "Adoption" rules where orphaned resources are automatically moved to the Root Catalog to prevent data loss.

### Fixed
- Updated Conformance Class URIs to match the new versioning.

[Unreleased]: https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/compare/v1.0.0-beta.3...main
[v1.0.0-beta.3]:
https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/compare/v1.0.0-beta.2...v1.0.0-beta.3
[v1.0.0-beta.2]: https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/compare/v1.0.0-beta.1...v1.0.0-beta.2
[v1.0.0-beta.1]: https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/releases/tag/v1.0.0-beta.1