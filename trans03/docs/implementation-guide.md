CIC SSSS — Implementation Guide & Schema Registry README

This repository is the canonical schema registry and API surface for the CIC SSSS (Construction Safety) platform. It contains OpenAPI / JSON Schema documents, normative registries (enums), examples and discovery metadata used by CMPs, solution vendors and integrators.

This guide explains how to discover the registry, resolve schema URLs, extract component schemas from OpenAPI documents, validate payloads, and follow versioning & contribution practices.
Quick summary

    Discovery entrypoint: /.well-known/schema-registry.json
    Master catalogue (per-release): /registry/<release>/catalogue.json — contains relative $ref pointers to base schemas
    Key base schemas: registry/<release>/schemas/{alert,device,measurement,software, basic-info}/base.json
    Normative lists (enums): registry/<release>/normative/*.json (e.g. alert-types)

Files referenced in this repository (examples):

    .well-known/schema-registry.json
    registry/2026-01/catalogue.json
    registry/2026-01/schemas/alert/base.json
    registry/2026-01/schemas/device/base.json
    registry/2026-01/schemas/measurement/base.json
    registry/2026-01/schemas/software/base.json
    registry/2026-01/schemas/basic-info/orgInfo.schema.json
    registry/2026-01/schemas/basic-info/zoneInfo.schema.json
    registry/2026-01/normative/alert-types.json

Table of contents

    Quick discovery
    Catalogue & schema resolution
    Extracting component schemas (jq)
    Validate payloads (ajv-cli, Node.js, Python)
    Representative payload examples
    API surface lookup (endpoints & base schema locations)
    Versioning & release policy
    Contributing checklist
    Troubleshooting & recommended tools
    Reference — key files & paths

Quick discovery

Find the registry entrypoint to learn which registry path to use:

Example: fetch discovery document

curl -sS https://api.example.com/.well-known/schema-registry.json | jq

Typical discovery doc (this repo's file: .well-known/schema-registry.json):

{
  "latest": "/registry/v1",
  "versions": ["v1"]
}

From latest, follow to the release directory. In this repo the actual catalogue file you use is: /registry/2026-01/catalogue.json

Fetch the catalogue:

curl -sS https://api.example.com/registry/2026-01/catalogue.json | jq

Example catalogue (repo):

{
  "version": "v1",
  "devices": { "$ref": "./schemas/devices/base.json" },
  "software": { "$ref": "./schemas/software/base.json" },
  "measurements": { "$ref": "./schemas/measurements/base.json" },
  "alerts": { "$ref": "./schemas/alerts/base.json" }
}

Notes:

    Catalogue $ref values are relative to the catalogue file location. Use the registry base path to assemble absolute URLs.
    registry/current should point to the latest stable path in production (e.g. /registry/2026-01). If registry/current is empty in the repo, your deployment should set it to the active release.

Catalogue & schema resolution

Given the discovery path and catalogue, construct schema URLs by resolving relative $ref:

Example:

    Discovery says latest: "/registry/v1"
    Suppose /registry/v1 maps to /registry/2026-01 on the server
    Catalogue $ref for alerts: ./schemas/alerts/base.json

Absolute URL:

https://api.example.com/registry/2026-01/schemas/alert/base.json

Tip: When automating, read latest, then fetch catalogue.json and resolve relative refs using the catalogue's directory as base.
Extracting component schemas from OpenAPI files

Base schemas are OpenAPI documents. To validate just a component (e.g., components.schemas.BaseAlert), extract it with jq.

Example: extract BaseAlert into a standalone JSON Schema file

# download OpenAPI file
curl -sS https://api.example.com/registry/2026-01/schemas/alert/base.json -o alert-base-openapi.json

# extract the component schema object (BaseAlert)
jq '.components.schemas.BaseAlert' alert-base-openapi.json > base-alert.schema.json

If the component references other components (e.g., #/components/schemas/AlertProperties), you'll need to either:

    Extract and inline dependencies manually
    Use a resolver tool to bundle/flatten references (tools below)

Recommended tools for resolving $ref:

    json-schema-ref-parser (npm package @apidevtools/json-schema-ref-parser) — can dereference and bundle schemas
    swagger-cli (npm i -g @apidevtools/swagger-cli) — can bundle OpenAPI files
    openapi-cli tools for OpenAPI-specific workflows

Example bundling with json-schema-ref-parser (Node):

const $RefParser = require("@apidevtools/json-schema-ref-parser");

(async () => {
  const schema = await $RefParser.bundle("alert-base-openapi.json");
  // schema.components.schemas.BaseAlert will now have resolved references (where possible)
})();

Validation & Quickstart

This section shows how to validate payloads against schemas using common tools.

Prerequisites:

    curl, jq
    For Node examples: Node.js (>=14), ajv, node-fetch or cross-fetch
    For CLI validation: ajv-cli (npm)
    For Python: requests, jsonschema

A. Using ajv-cli (fast CLI approach)

    Download OpenAPI file and extract the JSON Schema component:

curl -sS https://api.example.com/registry/2026-01/schemas/alert/base.json -o alert-openapi.json
jq '.components.schemas.BaseAlert' alert-openapi.json > base-alert.schema.json

    Save a payload to alert.json (see Representative payloads below).

    Validate with ajv-cli:

# install if needed
npm i -g ajv-cli

ajv validate -s base-alert.schema.json -d alert.json --strict=false

Notes:

    --strict=false can be helpful if schema uses OpenAPI-style features or if there are minor differences.
    If the extracted component references other components, ensure they’re inlined/bundled first.

B. Node.js example (AJV + fetch)

This snippet fetches the OpenAPI file and validates a sample payload using the BaseAlert component:

// install: npm i ajv node-fetch
const Ajv = require("ajv");
const fetch = require("node-fetch");

(async () => {
  const schemaUrl = "https://api.example.com/registry/2026-01/schemas/alert/base.json";
  const resp = await fetch(schemaUrl);
  const openapi = await resp.json();

  // extract the component schema for BaseAlert
  const baseAlert = openapi.components.schemas.BaseAlert;

  const ajv = new Ajv({strict:false});
  const validate = ajv.compile(baseAlert);

  const payload = {
    "category4S":"smart-monitoring-device",
    "siteId":"1693279120646",
    "supplierId":"SUPPLIER-001",
    "deviceSn":"DEVICE-001",
    "pushAt":"2025-01-06T14:40:16+08:00",
    "alertProperties": {
      "alertId":"ALERT-001",
      "alertType":"fall-detection",
      "alertLevel":2,
      "alertStatus":0,
      "alertMessage":{"en-HK":"Fall detected"},
      "alertTime":"2025-01-06T14:40:16+08:00",
      "alertNum":85.5
    }
  };

  const valid = validate(payload);
  console.log(valid ? "valid" : validate.errors);
})();

If baseAlert references other components, use a bundler (json-schema-ref-parser) to dereference first.
C. Python example (requests + jsonschema)

# pip install requests jsonschema
import requests
from jsonschema import validate, Draft7Validator, RefResolver
import json

# fetch OpenAPI doc
schema_url = "https://api.example.com/registry/2026-01/schemas/alert/base.json"
r = requests.get(schema_url)
openapi = r.json()

# extract BaseAlert (may contain $ref)
base_alert = openapi['components']['schemas']['BaseAlert']

# If base_alert contains $ref to other components, create a resolver using the full OpenAPI doc as store
store = {}
# build a mapping of component id -> object for resolver (optional strategy)
for name, comp in openapi['components']['schemas'].items():
    store[f"#/components/schemas/{name}"] = comp

resolver = RefResolver(base_uri=schema_url, referrer=base_alert, store=store)

payload = {
    "category4S":"smart-monitoring-device",
    "siteId":"1693279120646",
    "supplierId":"SUPPLIER-001",
    "deviceSn":"DEVICE-001",
    "pushAt":"2025-01-06T14:40:16+08:00",
    "alertProperties": {
      "alertId":"ALERT-001",
      "alertType":"fall-detection",
      "alertLevel":2,
      "alertStatus":0,
      "alertMessage":{"en-HK":"Fall detected"},
      "alertTime":"2025-01-06T14:40:16+08:00",
      "alertNum":85.5
    }
}

validator = Draft7Validator(base_alert, resolver=resolver)
errors = sorted(validator.iter_errors(payload), key=lambda e: e.path)
if errors:
    for err in errors:
        print(err.message, "at", list(err.path))
else:
    print("valid")

If you get resolution errors in Python, use a bundler to inline $ref or pass a proper store mapping.
Representative payload examples

(A) Device Delete (DeviceDeleteRequest)

{
  "deviceSn": "SN-8655078237",
  "siteId": "1693279120646",
  "category4S": "smart-monitoring-device"
}

(B) Push a single Alert (matches components.schemas.BaseAlert)

{
  "category4S": "smart-monitoring-device",
  "siteId": "1693279120646",
  "supplierId": "SUPPLIER-001",
  "deviceSn": "DEVICE-001",
  "pushAt": "2025-01-06T14:40:16+08:00",
  "alertProperties": {
    "alertId": "ALERT-001",
    "alertType": "fall-detection",
    "alertLevel": 2,
    "alertStatus": 0,
    "alertMessage": {
      "en-HK": "Fall detected",
      "zh-HK": "偵測到跌倒"
    },
    "alertTime": "2025-01-06T14:40:16+08:00",
    "alertNum": 85.5
  }
}

(C) Push a Measurement (category-specific measurement schema)

{
  "category4S": "confined-space",
  "siteId": "1693279120646",
  "supplierId": "SUPPLIER-001",
  "deviceSn": "DEVICE-CO2-01",
  "pushAt": "2025-01-06T14:45:00+08:00",
  "measurements": [
    { "measurementType": "co2", "value": 415.2, "unit": "ppm" }
  ]
}

(D) OrgInfo (Org/site/contract example)

{
  "orgId": "ORG-123",
  "fullName": { "en-HK": "ACME Construction", "zh-HK": "恩笙建築" },
  "contracts": [
    {
      "cicSiteId": 1001,
      "siteId": "SITE-001",
      "siteCode": "Project-A",
      "name": { "en-HK": "Project A" },
      "zones": [
        { "zoneId": "floor-05", "name": { "en-HK":"5/F"}, "zoneType":"floor" }
      ],
      "enableStatus": true
    }
  ]
}

(E) ZoneInfo (single zone detail)

{
  "zoneId": "floor-05",
  "siteId": "SITE-001",
  "name": { "en-HK": "Level 5", "zh-HK": "5樓" },
  "zoneType": "floor",
  "capacity": { "maxPersonnel": 20, "maxEquipment": 5 },
  "hazards": ["fall", "confined"],
  "enableStatus": true
}

API surface summary (where to find definitions)

Base OpenAPI files in this repo define the major endpoints and payload shapes:

    Alerts
        Path: registry/2026-01/schemas/alert/base.json
        Endpoints:
            POST /alerts — push alerts (Alert-001)
            PATCH /alerts — update alerts (Alert-002)
            POST /alerts/media — upload media (Alert-003)
            GET /alerts — query alerts (Alert-004)
            POST /alerts/status — push alert status (Alert-005)
        Component to extract: components.schemas.BaseAlert

    Devices
        Path: registry/2026-01/schemas/device/base.json
        Endpoints:
            POST /devices — add/register device (Device-001)
            PATCH /devices — update device (Device-002)
            DELETE /devices — delete device (Device-003)
            GET /devices — query devices (Device-004)
        Component to extract: components.schemas.DeviceBase

    Measurements
        Path: registry/2026-01/schemas/measurement/base.json
        Endpoints:
            POST /measurements — push streaming measurements (Measurement-001)
            GET /measurements — query historical measurements (Measurement-002)
        Component: components.schemas.MeasurementBase

    Software
        Path: registry/2026-01/schemas/software/base.json
        Endpoints:
            POST /software — add software (Software-001)
            PATCH /software — update (Software-002)
            DELETE /software — delete (Software-003)
            GET /software — query (Software-004)

    Basic Info (org/zone)
        Paths: registry/2026-01/schemas/basic-info/orgInfo.schema.json, .../zoneInfo.schema.json
        Endpoints:
            GET /org-info
            GET /org-info/{siteId}/zones
            GET /zones
            GET /zones/{zoneId}
            GET /zones/{zoneId}/capacity

    Normative lists (enums)
        registry/2026-01/normative/alert-types.json — YAML OpenAPI doc listing alertType enums by category (useful for UI enums and allowed values)

Versioning & release policy

    The registry exposes stable releases by directory (e.g. registry/2026-01). Each release is immutable once published.
    registry/current is expected to point to the latest stable release. In your deployed environment, ensure registry/current is a redirect or directory that maps to the actual release.
    Breaking changes MUST create a new release directory (e.g. registry/2027-01) and update the discovery latest.
    Non-breaking additions (optional fields) may be added in a patch release depending on governance — prefer new release directories for clarity.
    Clients SHOULD:
        Pin the registry path you support (e.g., /registry/2026-01) in integration settings and deploy CI checks to upgrade deliberately.
        Cache discovery/catalogue URIs and refresh periodically or follow server-provided cache headers.

Contributing checklist (how to propose schema changes)

When proposing changes:

    Create a PR with:
        A clear title and description that explains whether the change is breaking or non-breaking.
        Schema diffs (before/after) and rationale.
    For new categories:
        Add the category to registry/<release>/normative/categories.json (if applicable).
        Add new schema under registry/<release>/schemas/.../categories/.
        Update the base schema discriminator.mapping (if adding a new discriminator value) in a new release if the change is breaking.
    For updates/fixes:
        Edit the schema in the release that is being maintained (for non-breaking) or prepare a new release directory for breaking changes.
    Add/update example payloads and tests where applicable.
    Provide migration notes in docs/migration.md (or a PR description) for breaking changes.

Troubleshooting & recommended tools

Common issues:

    "Ref resolution failed" — Use bundlers to dereference; ensure relative refs are resolved against catalogue path.
    "Schema expects additional component" — extract or bundle the OpenAPI doc so that referenced components are present.
    “Validation pass locally but not in production” — verify the exact registry version/URL used in production; ensure client is pinning correct release.

Recommended tooling:

    CLI: ajv-cli (validation), jq (JSON extraction)
    Bundlers: @apidevtools/json-schema-ref-parser, @apidevtools/swagger-cli
    Local dev: node, python + jsonschema, requests
    CI: include a validation job that fetches the target release catalogue and validates example payloads and API responses against the extracted/bundled schemas.

Reference — key files & paths in this repository

    Discovery: .well-known/schema-registry.json
    Registry releases:
        registry/2026-01/catalogue.json
        registry/current (should point to latest release in deployment)
    Base schemas:
        registry/2026-01/schemas/alert/base.json
        registry/2026-01/schemas/device/base.json
        registry/2026-01/schemas/measurement/base.json
        registry/2026-01/schemas/software/base.json
        registry/2026-01/schemas/basic-info/orgInfo.schema.json
        registry/2026-01/schemas/basic-info/zoneInfo.schema.json
    Normative lists:
        registry/2026-01/normative/alert-types.json
        registry/2026-01/normative/* (other enumerations)

Local documentation:

    docs/implementation-guide.md
    docs/index.html

