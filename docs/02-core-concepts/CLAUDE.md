# CLAUDE.md - Core Concepts Section

This file provides guidance for working with the Core Concepts documentation section.

## Section Purpose

This section documents the fundamental building blocks of ThingsBoard:

- **Entities**: Device, Asset, Tenant, Customer, Dashboard, Alarm, Relations
- **Data Model**: Telemetry, Attributes, RPC, Calculated Fields
- **Identity**: Entity IDs, UUID addressing
- **Device Lifecycle**: Provisioning, Profiles, Claiming, OTA Updates

## File Structure

```
02-core-concepts/
├── README.md                 # Section overview and navigation
├── entities/                 # Entity type documentation
│   ├── README.md
│   ├── device.md
│   ├── asset.md
│   ├── tenant.md
│   ├── customer.md
│   ├── dashboard.md
│   ├── alarm.md
│   └── relations.md
├── data-model/               # Data types documentation
│   ├── README.md
│   ├── telemetry.md
│   ├── attributes.md
│   ├── rpc.md
│   └── calculated-fields.md
├── identity/                 # Identity and addressing
│   ├── README.md
│   └── entity-ids.md
├── device-provisioning.md    # Device registration
├── device-profiles.md        # Profile configuration
├── device-claiming.md        # Ownership transfer
└── ota-updates.md            # Firmware distribution
```

## Writing Guidelines

### Audience

Junior developers learning ThingsBoard concepts. Explain the "what" and "why" before the "how".

### Content Pattern

Each entity/concept document should include:

1. **Overview** - What it is and why it matters
2. **Key Properties** - Core fields and their purpose
3. **Relationships** - How it connects to other entities
4. **Common Operations** - Typical usage patterns
5. **Common Pitfalls** - What to avoid
6. **See Also** - Cross-references

### Terminology

- Use "Device" not "thing" or "endpoint"
- Use "Telemetry" for time-series data, "Attributes" for static/config data
- Use "Tenant" for organization/account level
- Use "Customer" for sub-organization level

### Diagrams

Use Mermaid diagrams to show:

- Entity relationships (`graph TB`)
- State machines (`stateDiagram-v2`)
- Data flow (`sequenceDiagram`)

### Technology-Agnostic Rule

Focus on concepts and behaviors, not implementation:

**DO**: "Devices store telemetry as time-series data with millisecond precision"
**DON'T**: "TelemetryDao persists TsKvEntry to PostgreSQL ts_kv table"

**DO**: "Entity IDs combine type and UUID for unique identification"
**DON'T**: "EntityId class extends UUIDBased with EntityType enum"

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard.github.io-master/docs/user-guide/` - Official user guides
- `~/work/viaanix/thingsboard-master/common/data/` - Entity data models
- `~/work/viaanix/thingsboard-master/dao/` - Data access patterns

## Related Sections

- `01-architecture/` - System design context
- `04-rule-engine/` - Processing entity data
- `07-data-persistence/` - Storage layer
- `09-security/` - Authorization and access control

## Common Tasks

### Adding a New Entity Type

1. Create `entities/{entity-name}.md`
2. Follow the entity template pattern
3. Update `entities/README.md` contents table
4. Add to `entities/entity-types-overview.md` if applicable
5. Update `../GLOSSARY.md` with new terms

### Updating Data Model

1. Verify changes against source code in `~/work/viaanix/thingsboard-master/`
2. Update relevant file in `data-model/`
3. Check for impacts on related concepts (e.g., if telemetry changes, check rule-engine docs)

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/02-core-concepts/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **technical-writer** | `/technical-writer` | Creating clear documentation, API references, user guides |
| **docs-write** | `/docs-write` | User-focused writing style, editing for clarity |
| **iot-engineer** | `/iot-engineer` | IoT concepts (devices, telemetry, protocols, provisioning) |

### When to Use Each Skill

- **Creating new entity documentation**: Use `/technical-writer` for structure, `/iot-engineer` for IoT-specific accuracy
- **Editing existing docs for clarity**: Use `/docs-write` for conversational style
- **Documenting device lifecycle**: Use `/iot-engineer` for provisioning, OTA, protocols
- **Writing data model docs**: Use `/technical-writer` + `/iot-engineer` for telemetry/attributes

## Helpful Paths

- local-skillz : `~/Projects/barf/repo/SKILLS/README.md`
