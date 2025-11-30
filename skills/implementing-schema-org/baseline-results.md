# Baseline Results: Schema.org Compliance

**Scenario:** Create `Venues` collection compliant with `Schema.org/Place`.

**Observed Failures (Simulated):**

1.  **Flattened Address**: Agents tend to put `street`, `city`, `state`, `zip` at the root level.
    - _Schema.org Requirement_: `address` should be a `PostalAddress` object (nested).
2.  **Non-Standard Naming**: Agents use `zip` instead of `postalCode`, or `lat`/`lng` instead of `latitude`/`longitude`.
    - _Schema.org Requirement_: Strict property names (`postalCode`, `latitude`, `longitude`).
3.  **Geo Flattening**: Agents put `latitude` and `longitude` at the root.
    - _Schema.org Requirement_: `geo` property containing a `GeoCoordinates` object.
4.  **Missing Context**: Agents fail to add `@context` and `@type` fields (or equivalent metadata) to make it valid JSON-LD when exported.

**Rationalizations:**

- "It's simpler to query flattened fields."
- "Payload doesn't natively support nested JSON-LD structures easily."
- "Zip is the common name in the US."

**Conclusion:** Without a skill, agents prioritize convenience over strict Schema.org compliance, breaking the "standardization" goal.
