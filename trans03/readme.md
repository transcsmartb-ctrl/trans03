Construction Safety API Schemas

This repository is the canonical schema registry and API surface for a construction safety platform (CIC SSSS). It contains OpenAPI/JSON Schema definitions, normative registries (enums), examples and discovery metadata used by CMPs, solution vendors and integrators.

Quick highlights

    Discover available schema versions with: .well-known/schema-registry.json
    Master catalog: registry/{version}/catalogue.json — links to the base schemas
    Canonical schemas for: devices, measurements, alerts, basic-info (org/zone), normative lists (alert types)
    Validate requests/responses against the JSON Schemas before sending or accepting them

Table of contents

    Quick discovery (how to find the registry)
    Catalog & schema resolution
    Quickstart (curl + validation examples)
    Representative payload examples (copy-paste)
    API summary (major endpoints described in schemas)
    Versioning & release policy
    Contributing and governance
    Reference (key files & paths)

Quick discovery

    Find the registry entrypoint:
        GET the discovery document:

        curl -sS https://api.example.com/.well-known/schema-registry.json

        In this repo the discovery file is: .well-known/schema-registry.json Example content:

        {
          "latest": "/registry/v1",
          "versions": ["v1"]
        }

    Obtain the latest registry catalogue:
        Use the latest path from .well-known to find the catalogue:

        # using the repo layout, replace host with your API domain
        curl -sS https://api.example.com/registry/2026-01/catalogue.json | jq

        The catalogue references base schemas:

        {
          "version": "v1",
          "devices": { "$ref": "./schemas/devices/base.json" },
          "software": { "$ref": "./schemas/software/base.json" },
          "measurements": { "$ref": "./schemas/measurements/base.json" },
          "alerts": { "$ref": "./schemas/alerts/base.json" }
        }

    Resolve a specific schema:
        Compose the URL from the registry base + referenced path. Example (device base schema):

        curl -sS https://api.example.com/registry/2026-01/schemas/device/base.json | jq .

Notes

    All catalogue $ref values are relative to the catalogue file location. Use the registry base path to construct absolute URLs.
    The repo contains registry/2026-01 as a release directory. registry/current is expected to point to the latest stable version in production.

Catalog & schema resolution (quick reference)

    Discovery: /.well-known/schema-registry.json -> returns latest path (/registry/<version>).
    Catalogue: /registry//catalogue.json -> lists top-level schema entrypoints for devices, measurements, alerts, software.
    Schema files: /registry//schemas/... -> full OpenAPI/JSON Schema documents.
    Normative lists (enums): /registry//normative/*.json (e.g. alert types).

Best practice

    Clients should cache schema URIs and periodically refresh discovery/catalogue (or follow server cache headers).
    Validate payloads using the exact schema published for the registry version you support — do not mix schemas from different registry versions.

Quickstart — fetch and validate (examples)

Prerequisites

    curl, jq
    Node.js or ajv-cli for JSON Schema validation

A. Fetch the latest registry path:

curl -sS https://api.example.com/.well-known/schema-registry.json | jq .latest
# => "/registry/v1"  (or "/registry/2026-01")

B. Download the alert base schema (example):

curl -sS https://api.example.com/registry/2026-01/schemas/alert/base.json -o alert-base.json

C. Validate a payload with ajv-cli (install: npm i -g ajv-cli)

    Save a payload to alert.json then:

ajv validate -s alert-base.json -d alert.json --strict=false

Note: Many of the repo files are OpenAPI documents that embed JSON Schema components. You may need to extract the specific component (e.g. #/components/schemas/BaseAlert) or use a validation script that resolves OpenAPI references. For general JSON Schema validation use the schema component file (or a flattened schema) that corresponds to the entity.

D. Minimal Node.js validation snippet (using ajv)

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

Representative payload examples (copy-paste)

A. Delete device (DeviceDeleteRequest)

{
  "deviceSn": "SN-8655078237",
  "siteId": "1693279120646",
  "category4S": "smart-monitoring-device"
}

B. Push a single alert (matches components.schemas.BaseAlert)

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

C. Push a measurement (example shape — category-specific measurement schemas live under measurements/*)

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

D. OrgInfo (Org/site/contract example)

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

E. ZoneInfo (single zone detail)

{
  "zoneId": "floor-05",
  "siteId": "SITE-001",
  "name": { "en-HK": "Level 5", "zh-HK": "5樓" },
  "zoneType": "floor",
  "capacity": { "maxPersonnel": 20, "maxEquipment": 5 },
  "hazards": ["fall", "confined"],
  "enableStatus": true
}

API summary (as defined in base schemas)

Major API surfaces are defined in the schema files under registry/<version>/schemas/:

    Devices
        POST /devices — add/register device (Device-001)
        PATCH /devices — update device (Device-002)
        DELETE /devices — delete device (Device-003)
        GET /devices — query devices (Device-004)
        Base schema file: registry/2026-01/schemas/device/base.json

    Measurements
        POST /measurements — push streaming measurements (Measurement-001)
        GET /measurements — query/paginate historical measurements (Measurement-002)
        Base schema file: registry/2026-01/schemas/measurement/base.json

    Alerts
        POST /alerts — push alerts (Alert-001)
        PATCH /alerts — update alerts (Alert-002)
        POST /alerts/media — upload alert media (Alert-003)
        GET /alerts — query alerts (Alert-004)
        POST /alerts/status — push alert status (Alert-005)
        Base schema file: registry/2026-01/schemas/alert/base.json

    Basic Info
        GET /org-info — query site/org hierarchy
        GET /org-info/{siteId}/zones — query zones for a site
        Schema files: registry/2026-01/schemas/basic-info/orgInfo.schema.json and zoneInfo.schema.json

Refer to each base schema for parameter definitions, security schemes (bearerAuth/JWT), request/response envelopes and pagination patterns.
Normative registries (enums)

    alert-types: registry/2026-01/normative/alert-types.json (OpenAPI document enumerating alertType codes by category)
    categories / device-types / measurement-types: registry/2026-01/normative/*.json

Clients should use these normative lists to populate type fields, enums and to drive UI selections. Normative lists may be published as separate APIs (e.g. /alert-types).
Versioning & compatibility

    registry/current should point to the latest stable registry release (e.g. /registry/2026-01).
    Breaking changes MUST create a new registry version directory (e.g. /registry/2027-01) with a new catalogue.json; old versions remain immutable.
    Non-breaking additions (new optional fields) may be added in-place depending on governance — prefer minor-version entries or patch notes.

Contributing

To propose a change:

    Open a pull request describing the change, the affected version, and if the change is breaking.
    For new categories:
        Add an entry in the appropriate registry/<version>/normative/*.json file (category, device-type, etc.).
        Add the new schema file under registry/<version>/schemas/.../categories/ or appropriate folder.
        Add or update example payloads under api/ if present.
    For fixes or clarifications:
        Edit the schema file or documentation file and include schema diffs and rationales in the PR.
    Maintain backward compatibility where possible; mark breaking changes clearly.

Reference — key files & paths

    Discovery: .well-known/schema-registry.json
    Registry catalogue: registry/2026-01/catalogue.json
    Device base schema: registry/2026-01/schemas/device/base.json
    Measurement base schema: registry/2026-01/schemas/measurement/base.json
    Alert base schema: registry/2026-01/schemas/alert/base.json
    Org/Zone schemas: registry/2026-01/schemas/basic-info/orgInfo.schema.json, zoneInfo.schema.json
    Normative lists: registry/2026-01/normative/*

Local docs: docs/index.html and docs/implementation-guide.md (see repo docs/ for more human-friendly guidance).
