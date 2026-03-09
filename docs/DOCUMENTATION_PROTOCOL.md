# Technical Documentation Generation Protocol

## Purpose

This document defines the protocol for generating and maintaining technical documentation markdown files for the distributed architecture components. It ensures consistency, completeness, and quality across all component documentation.

## Scope

This protocol applies to all distributed architecture components:
- Multi App
- Remote Controller
- DC-DR Support

## Documentation Structure

Each component must have a `PROVISIONING_STEPS.md` file in its respective directory under `docs/`. The directory layout must follow this structure:

```
docs/
├── README.md                              # Documentation index
├── architecture-overview.md               # High-level architecture
├── DOCUMENTATION_PROTOCOL.md              # This protocol
├── multi-app/
│   └── PROVISIONING_STEPS.md
├── remote-controller/
│   └── PROVISIONING_STEPS.md
└── dc-dr/
    └── PROVISIONING_STEPS.md
```

## Required Sections in PROVISIONING_STEPS.md

Every `PROVISIONING_STEPS.md` file must include the following sections in order:

### 1. Component Header

| Field | Description |
|-------|-------------|
| **Component Name** | Official name of the component |
| **Purpose** | One-paragraph description of what the component does |
| **Version** | Current version of the component |

### 2. Prerequisites

- **Dependencies** — List of software, services, and components that must be installed before provisioning this component. Each dependency must include the minimum version.
- **Hardware Requirements** — Minimum CPU, RAM, and disk specifications.
- **Network Requirements** — Required ports, protocols, and firewall rules.

### 3. Network Ports and Firewall Restrictions

A table documenting every network port used by the component:

| Column | Description |
|--------|-------------|
| Port | Port number or range |
| Protocol | TCP / UDP |
| Direction | Inbound / Outbound |
| Source | Where traffic originates |
| Destination | Where traffic is directed |
| Purpose | What the port is used for |
| Required | Whether the port is mandatory |

### 4. Provisioning Steps

Step-by-step instructions to deploy the component. Each step must include:
- **Step number and title**
- **Action description** — What to do
- **Commands** — Exact commands to run (in fenced code blocks)
- **Expected outcome** — What the operator should observe upon success

### 5. Validation

Post-provisioning checks to confirm the component is working correctly:
- **Validation class files** — References to Java validation classes with brief descriptions of what each validates
- **Manual checks** — Steps an operator can perform to verify functionality
- **Automated checks** — Commands or scripts that can verify the deployment

### 6. Troubleshooting

Common issues and their resolutions, presented as a table or list.

### 7. Screenshots

Links or placeholders for screenshots that illustrate key provisioning steps or successful deployment states.

## Markdown Formatting Standards

### General Rules

1. Use ATX-style headings (`#`, `##`, `###`).
2. Use fenced code blocks with language identifiers for all commands and code snippets.
3. Use tables for structured data (ports, dependencies, configuration parameters).
4. Use ordered lists for sequential steps.
5. Use unordered lists for non-sequential items.
6. Include blank lines before and after headings, code blocks, and tables.

### Code Blocks

Use language-specific fencing:

````
```bash
# Shell commands
systemctl start postgresql
```

```sql
-- SQL statements
CREATE EXTENSION cstore_fdw;
```

```java
// Java code
ValidationResult result = validator.validate();
```
````

### Placeholders

When exact values depend on the deployment environment, use angle-bracket placeholders:

```
<hostname>        — Target machine hostname
<port>            — Service port number
<db_name>         — Database name
<workspace_id>    — Workspace identifier
<install_dir>     — Installation directory path
```

### Cross-References

Link to other documentation files using relative paths:

```markdown
See [Architecture Overview](../architecture-overview.md) for the high-level design.
```

## Documentation Generation Workflow

### For New Components

1. Create a new directory under `docs/` matching the component name (lowercase, hyphen-separated).
2. Create `PROVISIONING_STEPS.md` following the required sections above.
3. Update `docs/README.md` to add the new component to the index table.
4. Update `docs/architecture-overview.md` if the component introduces new architectural elements.

### For Existing Components

1. Identify the sections that need updates.
2. Preserve the existing structure and section ordering.
3. Add change notes at the top of the file if the update is significant.
4. Verify all cross-references and links remain valid.

### Validation Checklist

Before merging documentation changes, verify:

- [ ] All required sections are present in `PROVISIONING_STEPS.md`
- [ ] Network ports table is complete with all ports used by the component
- [ ] Dependencies list includes minimum version requirements
- [ ] All provisioning steps include commands and expected outcomes
- [ ] Validation section references applicable validation classes
- [ ] Cross-references and links are valid
- [ ] Markdown renders correctly (no broken formatting)
- [ ] Placeholder values use consistent angle-bracket notation
- [ ] Code blocks specify the correct language identifier
