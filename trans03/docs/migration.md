Migration Guide
Step-by-step instructions for upgrading from legacy versions (2025.01) to current.

Version Timeline
Version	Release	Status	Changes
2025.01	Jan 2025	Legacy	Initial release
current	Feb 2026	Active	+4 schemas, normative updates
Breaking Changes
1. Path Structure
text
# OLD (2025.01)
/registry/2025.01/schemas/alert/mewp.schema.json

# NEW (current) - Added /categories/ layer
/registry/current/schemas/alert/categories/mewp.schema.json
Migration Script:

bash
#!/bin/bash
# Update import paths
find src/ -name "*.ts" -o -name "*.js" | xargs sed -i 's|2025.01/|current/|g'
find src/ -name "*.ts" -o -name "*.js" | xargs sed -i 's|/schemas/\([^/]*\)/|\1/categories/|g'
2. New Required Fields
Schema	New Required	Default Value
AlertBase	location	{ lat: 0, lng: 0 }
DeviceBase	serialNumber	UUID generator
Measurement	timestamp	new Date().toISOString()
Non-Breaking Additions
New Schemas (4 added)
text
software/categories/
├── digitalised-tracking-system.schema.json  ✨ NEW
├── vr-training.schema.json                 ✨ NEW
└── e-permit.schema.json                    ✨ NEW (moved from alert-only)
New Categories
Domain	Added
Software	digitalised-tracking-system, vr-training
Shared	e-permit (previously alert-only)
Normative Updates
text
categories.json          +3 new codes
device-types.json        +8 equipment types  
measurement-types.json   +5 sensor units
errors.json              +12 error codes
Migration Steps
Step 1: Update Discovery Endpoint
typescript
// OLD
const BASE_URL = 'https://registry.4s-safety.com/registry/2025.01';

// NEW  
const BASE_URL = 'https://registry.4s-safety.com/registry/current';
const registry = await fetch(`${BASE_URL}/.well-known/schema-registry.json`);
Step 2: Schema Path Resolver
typescript
// Helper for new category structure
const resolveSchemaPath = (domain: string, category: string) => 
  `/registry/current/schemas/${domain}/categories/${category}.schema.json`;

// Usage
const mewpSchema = resolveSchemaPath('alert', 'mewp');
Step 3: Update Validation Pipeline
typescript
// OLD - single schema
const validate = ajv.compile(await fetchSingleSchema('alert/mewp'));

// NEW - base + category composition
const schemas = await Promise.all([
  fetch(`${BASE_URL}/schemas/alert/base.schema.json`),
  fetch(`${BASE_URL}/schemas/alert/categories/mewp.schema.json`)
]);
const combined = { allOf: schemas.map(s => s.json()) };
Step 4: Handle New Required Fields
typescript
function migratePayload(payload: any) {
  return {
    ...payload,
    // Add defaults for new required fields
    location: payload.location || { lat: 0, lng: 0 },
    serialNumber: payload.serialNumber || generateUUID(),
    timestamp: payload.timestamp || new Date().toISOString()
  };
}
