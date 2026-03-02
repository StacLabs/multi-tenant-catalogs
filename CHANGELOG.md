# Changelog

All notable changes to the "Multi-Tenant Catalogs" STAC API Extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Optional Transactions Conformance Class:** Explicitly separated the management plane (`POST`, `PUT`, `DELETE`) into an optional `https://api.stacspec.org/v1.0.0-beta.2/multi-tenant-catalogs/transaction` conformance class. This ensures public-facing APIs can safely implement the core extension as read-only by default without violating the specification.

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

[Unreleased]: https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/compare/v1.0.0-beta.2...main
[v1.0.0-beta.2]: https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/compare/v1.0.0-beta.1...v1.0.0-beta.2
[v1.0.0-beta.1]: https://github.com/Healy-Hyperspatial/multi-tenant-catalogs/releases/tag/v1.0.0-beta.1