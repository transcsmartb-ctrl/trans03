schemas/
├── index.html                    # Landing page
├── docs/
│   ├── implementation-guide.md   # API usage
│   └── migration.md              # Version upgrades
├── registry/
│   ├── 2025.01/                 # Previous version
│   └── current/                  # Active version
│       ├── catalogue.json        # Master index
│       ├── normative/            # Code lists
│       │   ├── alert-types.json
│       │   ├── categories.json
│       │   ├── device-types.json
│       │   ├── errors.json
│       │   └── measurement-types.json
│       └── schemas/
│           ├── category4s.schema.json
│           ├── alert/             # 12 category schemas
│           ├── device/            # 10 category schemas  
│           ├── measurement/       # 10 category schemas
│           ├── software/          # 4 category schemas
│           └── basic-info/        # 2 schemas
└── .well-known/
    ├── schema-registry.json      # Discovery endpoint
    └── 4s-openapi.json           # Swagger spec

| Endpoint                          | Purpose                     | Format      |
| --------------------------------- | --------------------------- | ----------- |
| /.well-known/schema-registry.json | Master catalog + path index | YAML/JSON   |
| /.well-known/4s-openapi.json      | Interactive API spec        | OpenAPI 3.1 |
| /registry/current/catalogue.json  | Complete schema inventory   | JSON        |


Core Components
Normative Registries (5)
Standardized code lists for interoperability:

alert-types.json - Alert severity codes

categories.json - Domain categories

device-types.json - Equipment classifications

errors.json - Error code registry

measurement-types.json - Sensor units/types

Domain Schemas (45 total)

| Domain      | Base Schema                  | Categories | Total |
| ----------- | ---------------------------- | ---------- | ----- |
| Alert       | alert/base.schema.json       | 12         | 13    |
| Device      | device/base.schema.json      | 10         | 11    |
| Measurement | measurement/base.schema.json | 10         | 11    |
| Software    | software/base.schema.json    | 4          | 5     |
| Basic Info  | -                            | 2          | 2     |
| Category    | category4s.schema.json       | -          | 1     |

Shared Categories (across alert/device/measurement): ai-monitoring, confined-space, lock-key, mewp, mobile-plant-operation, others, scaffold-coupler, smart-monitoring-device, tower-crane

Software Only: digitalised-tracking-system, e-permit, vr-training

Usage
1. Client Integration

# Fetch discovery document
curl https://registry.4s-safety.com/.well-known/schema-registry.json

# Get MEWP alert schema
curl https://registry.4s-safety.com/registry/current/schemas/alert/categories/mewp.schema.json

# Validate payload
npm install ajv @your-org/4s-schema-registry

2. API Base Paths

https://registry.4s-safety.com/schemas/
├── /.well-known/schema-registry.json
├── /.well-known/4s-openapi.json
├── /registry/current/catalogue.json
├── /registry/current/normative/alert-types.json
└── /registry/current/schemas/alert/categories/tower-crane.schema.json


3. Swagger UI
Load /.well-known/4s-openapi.json in Swagger Editor for interactive browsing/testing.

Schema Validation Pipeline
text
Payload → Category Resolution → Base Schema + Category Schema → Normative Validation → Response
Example: Tower crane alert uses alert/base.schema.json + alert/categories/tower-crane.schema.json + normative alert-types.json

