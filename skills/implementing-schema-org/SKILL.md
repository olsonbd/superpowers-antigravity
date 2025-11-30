---
name: implementing-schema-org
description: Use when defining data models, creating Payload Collections, or structuring API responses - ensures strict compliance with Schema.org standards for interoperability and SEO, preventing ad-hoc naming and flattened structures
---

# Implementing Schema.org Data Models

## Overview

Schema.org is the internet's standard for structured data. Adopting it prevents "bikeshedding" over field names and ensures our data is ready for SEO, AI agents, and external integrations (like calendars) from day one.

**Core Principle:** If Schema.org has a standard for it, use it. Do not invent your own field names.

## When to Use

- **Creating a new Collection** (e.g., Venues, Events, Users)
- **Defining API responses**
- **Mapping data for export**

## The Rules

1.  **Strict Naming**: Use the exact property name from Schema.org (e.g., `postalCode` not `zip`, `givenName` not `firstName`).
2.  **Preserve Structure**: If Schema.org defines a property as a nested Type (like `address` -> `PostalAddress`), you MUST nest it. Do not flatten it for convenience.
3.  **Use Standard Types**: Map your entities to the most specific Schema.org Type available (e.g., `Event` -> `SocialEvent` if applicable).

## Common Mappings

| Our Concept | Schema.org Type | Key Fields                                                                            |
| :---------- | :-------------- | :------------------------------------------------------------------------------------ |
| **Venue**   | `Place`         | `name`, `address` (PostalAddress), `geo` (GeoCoordinates), `publicAccess`             |
| **Event**   | `Event`         | `name`, `startDate`, `endDate`, `location` (Place), `organizer` (Person/Organization) |
| **User**    | `Person`        | `givenName`, `familyName`, `email`, `telephone`                                       |

## Implementation Pattern (Payload CMS)

### ❌ BAD: Flattened & Ad-Hoc

```typescript
// Venues.ts
fields: [
  { name: "street", type: "text" }, // Violation: Flattened
  { name: "zip", type: "text" }, // Violation: Wrong name
  { name: "lat", type: "number" }, // Violation: Wrong name & flattened
];
```

### ✅ GOOD: Nested & Standard

```typescript
// Venues.ts
fields: [
  {
    name: "address", // Schema.org property
    type: "group", // Preserves structure
    fields: [
      { name: "streetAddress", type: "text" },
      { name: "addressLocality", type: "text" }, // City
      { name: "addressRegion", type: "text" }, // State
      { name: "postalCode", type: "text" }, // Zip
      { name: "addressCountry", type: "text" },
    ],
  },
  {
    name: "geo",
    type: "group",
    fields: [
      { name: "latitude", type: "number" },
      { name: "longitude", type: "number" },
    ],
  },
];
```

## JSON-LD Serialization

When exposing this data via API, simply wrap it to produce valid JSON-LD:

```typescript
// Example response structure
{
  "@context": "https://schema.org",
  "@type": "Place",
  "name": venue.name,
  "address": {
    "@type": "PostalAddress",
    ...venue.address // Spread the nested group directly!
  },
  "geo": {
    "@type": "GeoCoordinates",
    ...venue.geo
  }
}
```

**Benefit:** Because we preserved the structure in the database, serialization is just a spread operator (`...venue.address`). No complex mapping code required.

## Red Flags - STOP and Fix

- "I'll just call it `zip` because it's shorter." -> **STOP**. Use `postalCode`.
- "Nested groups are annoying to query." -> **STOP**. Structure matters more than convenience.
- "I'll add a `customField`." -> **Check Schema.org first**. Is there really no standard field?

## References

- [Schema.org/Place](https://schema.org/Place)
- [Schema.org/Event](https://schema.org/Event)
- [Schema.org/Person](https://schema.org/Person)
