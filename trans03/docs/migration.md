Migration Guide — CIC SSSS Schema Registry

This migration guide explains how to introduce new registry releases (breaking and non‑breaking changes), how consumers locate and validate new schemas, and how teams should test, publish and roll back changes safely.

Scope

    Applies to the schema registry layout in this repository (discovery -> catalogue -> base OpenAPI/JSON Schema documents).
    Intended for schema maintainers (publishers) and integrators (consumers) migrating between registry releases (for example: registry/2026-01 -> registry/2027-01).

Contents

    Quick discovery recap
    Migration principles & versioning policy
    Publisher (author) workflow — creating a new release
    Consumer workflow — migrating clients
    Compatibility testing, diffs and CI checks
    Release checklist and rollback plan
    Tools & references

Quick discovery recap

Clients find the registry via the discovery document:

    Discovery: /.well-known/schema-registry.json — contains "latest" (e.g. "/registry/v1") and versions.
    Catalogue for a release: /registry/<release>/catalogue.json — maps to top-level base schema files (alerts, devices, measurements, software, basic-info).
    Example (fetch discovery and catalogue):

# Get discovery
curl -sS https://api.example.com/.well-known/schema-registry.json | jq .

# Follow the latest to the catalogue (example in this repo)
curl -sS https://api.example.com/registry/2026-01/catalogue.json | jq .

Notes:

    Catalogue $ref values are relative paths (e.g. ./schemas/alert/base.json). Construct absolute URLs using the release base path.
    The repository contains date-stamped release directories (e.g. registry/2026-01). registry/current should point to the active release in production.

Migration principles & versioning policy

    Breaking changes MUST produce a new release directory (e.g. registry/2027-01).
        Never change or mutate an already-published release directory (immutable).
        Update catalogue.json and add new/modified schema files in the new release directory.

    Non-breaking additions
        Add optional fields, new enum values, or extend category-specific properties in a backwards-compatible way whenever possible.
        Consider a new release for significant additions to keep explicit versioning.

    Discovery / latest pointer
        After a release is validated and staged, update .well-known/schema-registry.json's latest (or registry/current) to point to the new release path.
        Coordinate updates with downstream integrators (announce the date/time and compatibility notes).

    Normative lists (enums)
        Add new enumeration entries in registry/<release>/normative/*.json in the same release.
        Consumers should fetch normative lists from the same release version they validate against.

Publisher (schema author) workflow — creating a new release

High-level steps to publish a new registry release (breaking change example):

    Create a new release directory
        Copy the previous release directory as a starting point:

        cp -r registry/2026-01 registry/2027-01

        Update or replace modified schema files under registry/2027-01/schemas/....

    Update catalogue.json
        Edit registry/2027-01/catalogue.json if base schema paths changed.
        Ensure $ref paths are correct and relative to the catalogue file.

    Update normative lists
        Add/modify registry/2027-01/normative/*.json as needed.

    Add release notes & migration notes
        Create registry/2027-01/RELEASE_NOTES.md summarizing breaking changes, affected endpoints and required client changes.
        Add sample diff snippets and representative payloads.

    Validate the release locally
        Run schema validator and OpenAPI linter on each modified OpenAPI/JSON Schema file.
        Example: use ajv-cli to validate example payloads against extracted schema components.

    CI / integration tests (see CI checklist below)
        Run contract tests against a staging vendor/CMP mock.

    Publish: update discovery pointer
        Update .well-known/schema-registry.json to include the new latest path or change registry/current to point to the new release.
        Alternatively, publish the new release and leave latest unchanged until integration testing completes — coordinate with integrators.

Example: update discovery to point to the new release

# Edit .well-known/schema-registry.json
# {
#   "latest": "/registry/2027-01",
#   "versions": ["v1", "v2", "v3"]
# }

Suggested PR title/template: [registry/2027-01] Add release 2027-01 — breaking changes: <short-list>
Consumer workflow — migrating clients

When a new release is available, follow these steps:

    Discover the new release

curl -sS https://api.example.com/.well-known/schema-registry.json | jq .latest
# => "/registry/2027-01"
curl -sS https://api.example.com/registry/2027-01/catalogue.json | jq .

    Resolve and fetch the relevant base schema

    Compose the absolute URL from registry base + catalogue $ref.

Example: fetch alert base schema

BASE="https://api.example.com/registry/2027-01"
CATALOGUE_URL="$BASE/catalogue.json"
SCHEMA_REF=$(curl -sS $CATALOGUE_URL | jq -r '.alerts["$ref"]')   # e.g. "./schemas/alert/base.json"
SCHEMA_URL="$BASE/$(echo $SCHEMA_REF | sed 's|^\./||')"
curl -sS "$SCHEMA_URL" -o alert-base.json

    Extract the component schema (OpenAPI -> JSON Schema) if needed

    Use jq to extract a component, for example BaseAlert:

jq '.components.schemas.BaseAlert' alert-base.json > base-alert.schema.json

    Validate your payloads

    Example using ajv-cli:

npm i -g ajv-cli
ajv validate -s base-alert.schema.json -d my-example-alert.json --strict=false

    Or use a small Node.js snippet (as in docs/implementation-guide.md) to compile and run validations that resolve references.

    Schema diff & impact analysis

    Use a schema diff tool to find incompatible changes:
        json-schema-diff (npm) or schemastore tools or a custom script that compares components.schemas objects between releases.
    Example (using a hypothetical json-schema-diff CLI):

npm i -g json-schema-diff
json-schema-diff registry/2026-01/schemas/alert/base.json registry/2027-01/schemas/alert/base.json --component BaseAlert

(If this tool isn't available, compare jq outputs and run a textual diff.)

    Run integration tests against a staging endpoint

    Use consumer tests to exercise:
        Request/response validation
        Authentication and paging semantics
        Category-specific behaviors

    Backwards compatibility runbook

    If client must support both old and new schemas, implement runtime detection:
        Fetch discovery once and cache release path(s) per site/environment.
        Validate incoming data against the declared schema version (include registry version metadata in messages where possible).

Compatibility testing & automated checks

Automate the following checks in CI for any registry change PR:

    Static validations:
        OpenAPI linting (e.g. Spectral)
        JSON Schema validity (ajv)
    Schema diffs:
        Run a component-level diff between the old and new schemas for each modified base (alert, device, measurement, software, basic-info).
        Fail the PR if changes are breaking unless the author marked the PR as a new release (breaking) and added migration notes.
    Example payload validation:
        Validate a sample set of upstream and downstream payloads with the new schema.
    Contract tests
        Run consumer contract tests (mock server) to ensure the consumers continue to work (or entry points are updated).

Sample CI snippet (bash) — run in PR against main:

# Assumes registry/releaseA and registry/releaseB are available in the repo workspace
OLD=registry/2026-01
NEW=registry/2027-01

# 1. Validate JSON schema syntax
for f in $(find $NEW/schemas -name '*.json'); do
  ajv validate -s $f -d '{}' --errors=text --strict=false || true
done

# 2. Diff key components (example: BaseAlert)
jq '.components.schemas.BaseAlert' $OLD/schemas/alert/base.json > /tmp/old.BaseAlert.json
jq '.components.schemas.BaseAlert' $NEW/schemas/alert/base.json > /tmp/new.BaseAlert.json
json-diff /tmp/old.BaseAlert.json /tmp/new.BaseAlert.json || echo "diff output above"

# 3. Validate example payloads against new schema
ajv validate -s $NEW/schemas/alert/base.json -d tests/examples/alert/example1.json --strict=false || exit 1

Breaking change classification (examples)

Mark a change as breaking if any of the following occur:

    Removing a required property from a schema.
    Changing a property's type (e.g. string -> integer).
    Tightening enum sets (removing enum members).
    Changing request/response envelopes (root structure, authentication requirements).
    Changing endpoint semantics (path, HTTP method, required query params).

Non-breaking:

    Adding optional properties.
    Adding new enum values (consumers should ignore unknown values).
    Adding additional response fields.

Release checklist (publisher)

Before publishing a new release directory:

    All modified OpenAPI/JSON Schema files validated (ajv, OpenAPI linter).
    Schema diffs documented in RELEASE_NOTES.md.
    Representative payload examples updated and validated.
    Normative lists updated in registry/<release>/normative/.
    Integration/contract tests pass in staging.
    Announce migration plan and deprecation schedule to integrators.
    Update .well-known/schema-registry.json or registry/current according to release plan.

Sample PR description (required):

    Release name (registry/2027-01)
    What changed (list files)
    Breaking? (yes/no). If yes, include migration steps and example diffs
    Risks and roll-back plan

Rollback and emergency hotfixes

If a published release introduces a regression:

    Quick rollback:
        Revert .well-known/schema-registry.json or registry/current to point to the previous release.
        If consumers already fetched the new schema, notify them to revert or to use the stable release until fix is applied.
    Hotfix policy:
        Do not patch an immutable release directory. Instead:
            Create a new patch release (e.g. registry/2027-01-patch1 or registry/2027-02).
            Update discovery to point to the corrected release after validation.
    Communication:
        Post incident notes with affected endpoints, remediation actions and expected timelines.

Consumer & publisher best practices (summary)

    Always pin to a release while validating (do not mix schemas from different release directories).
    Cache discovery and catalogue but refresh periodically and on startup/rollout.
    Use component-level schema extraction for validation (use jq to extract OpenAPI components).
    Automate diffs and contract tests in CI; require REVIEW and migration notes for breaking changes.
    Document migration steps and examples in RELEASE_NOTES.md for each release.

Recommended tools & references

    curl, jq — lightweight discovery & extraction
    ajv / ajv-cli — JSON Schema validation (Node.js)
    json-schema-diff (npm) or custom jq+git-diff — schema diffs
    Spectral — OpenAPI linting
    Contract test frameworks — Pact, JSON Schema based contract tests or custom integration test suites

Useful commands (recap)

# Discover latest release
curl -sS https://api.example.com/.well-known/schema-registry.json | jq .latest

# Get catalogue and resolve schema URL
curl -sS https://api.example.com/registry/2026-01/catalogue.json | jq .

# Extract a schema component
jq '.components.schemas.BaseAlert' registry/2026-01/schemas/alert/base.json > base-alert.schema.json

# Validate local example
ajv validate -s base-alert.schema.json -d tests/examples/alert/example.json --strict=false

Appendix — Example migration scenario

Scenario: you need to add a new required field deviceModel to device registration which will be a breaking change. Recommended flow:

    Create registry/2027-01 copied from registry/2026-01.
    Edit registry/2027-01/schemas/device/base.json and update DeviceBase or specific category schema, adding "deviceModel" in required.
    Add migration notes to registry/2027-01/RELEASE_NOTES.md that show:
        Example old payloads and new payloads
        Suggest client code change (validate presence of deviceModel)
    Run CI that diffs the old/new schema and flags breaking changes.
    Publish registry/2027-01, update .well-known/schema-registry.json to point latest to /registry/2027-01 once integrators are ready.
    Coordinate a communication window that allows clients to adapt.

Reference — key paths in this repo

    Discovery: .well-known/schema-registry.json
    Catalogue (example): registry/2026-01/catalogue.json
    Base schemas: registry/2026-01/schemas/{alert,device,measurement,software}/base.json
    Basic info schemas: registry/2026-01/schemas/basic-info/orgInfo.schema.json, zoneInfo.schema.json
    Normative lists: registry/2026-01/normative/*.json
    Local implementation guide: docs/implementation-guide.md
    (Add per-release) registry//RELEASE_NOTES.md — put migration steps here

