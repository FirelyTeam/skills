# Firely Terminal Reference

Firely Terminal (`fhir`) is a cross-platform .NET CLI tool for FHIR resource validation, package management, Simplifier.net synchronization, and IG authoring workflows. It uses the same Firely .NET SDK validator engine as Simplifier.net and Forge — providing  **profile-aware** validation, not just schema checking.

Official docs: https://docs.fire.ly/projects/Firely-Terminal/

Firely Terminal was build with an internal stack, to allow you to combine multiple actions on one or more resources. All resource operations in Firely Terminal are performed on that stack.
The validate command validates the resource on the top of the stack, using the current Scope. Validation needs a lot of data: from StructureDefinitions on the resource itself, to data types, Extensions and ValueSets. It will look into your project (current folder) and any packages, wether dependencies or depenencies of dependencies (etc.) to find all these assets.

---

## Installation

Requires **.NET SDK 8** (not just the runtime):

```bash
# Windows (winget)
winget install Microsoft.DotNet.SDK.8

# Or download from https://dotnet.microsoft.com/download

# Install Firely Terminal
dotnet tool install -g firely.terminal

# Update to latest
dotnet tool update -g firely.terminal

# Verify
fhir help
fhir --version
```

---

## Validation (Core Feature)

`fhir validate` runs the Firely .NET SDK validator — identical to Simplifier.net's validator. It checks:
- Base FHIR conformance (cardinality, data types, references)
- Profile constraints (slicing, invariants, Must Support)
- Terminology bindings (ValueSet membership)
- StructureDefinition invariants (FHIRPath expressions)

Use `fhir validate [file] --push` to validate a file and keep the resulting OperationOutcome on the stack for later review. On Windows, **avoid absolute backslash paths** — pass relative paths or use `Push-Location` to cd into the folder first and pass a bare filename.

```bash
# Validate a single resource
fhir validate patient.json

# Validate all resources in a folder — loop
for f in ./input/resources/*.json; do
    fhir validate "$f" --push    # --push keeps OperationOutcome on stack for review
done
```

On **Windows (PowerShell)**, use `Push-Location` to cd into each folder and pass only the bare filename — absolute Windows backslash paths fail:

```powershell
fhir clear
$folders = @("./input/resources", "./input/examples", "./fsh-generated/resources")
foreach ($folder in $folders) {
    if (Test-Path $folder) {
        Push-Location $folder
        Get-ChildItem -Filter "*.json" | ForEach-Object {
            fhir validate $_.Name --push
        }
        Pop-Location
    }
}
```

```bash
# Set FHIR version context before validating
fhir spec r4

# Install deps from package.json first (required for profile resolution)
fhir restore

# Output formats
fhir validate --output SUMMARY   # markdown summary
fhir validate --output RAW       # full OperationOutcome JSON
```

**Note:** Python `fhir.resources` / Pydantic validation only checks JSON structure — it does **not** validate against profiles, slices, or FHIRPath invariants. Use `fhir validate` for real conformance checking.

---


## Quality Control (QC)

Quality Control runs FHIRPath-based business rules across an entire specification. Requires a Simplifier account.

Rules are defined in `.rules.yaml` files, here some examples:

```yaml
- action: parse
  name: parse-fhir-resources 
  status: "Checking if all FHIR Resource files can be parsed"
  files:
    - /**/*.xml
    - /**/*.json
    - "!package.json"
    - "!feed.config.json"
    - "!**/node_modules/**"
    - "!validator/advisor.json"
    - "!fhirpkg.lock.json"
    - "!fsh-generated/data/**"

- name: id-mandatory
  error: https://acme.com/fhir/qa/errors|id-mandatory
  status: "Checking if all resources have an id"
  predicate: id.exists()
  error-message: "Resource must have an id"
 
- name: id-valid
  status: "Checking if all resources have valid ids"
  predicate: id.matches('^[A-Za-z0-9\\-\\.]{1,64}$')
  error-message: The resource must have a valid id
 
- name: id-unique-profiles
  error: https://acme.com/fhir/qa/errors|id-unique-profiles
  status: "Checking if all profiles resources have a unique id"
  filter: StructureDefinition.exists()
  unique: id
 
- name: id-unique-examples
  error: https://acme.com/fhir/qa/errors|id-unique-examples
  status: "Checking if all examples have a unique id"
  category: example
  unique: id
 
- name: id-format-starts-with-ht
  error: https://acme.com/fhir/qa/errors|id-format-starts-with-ht
  status: "Checking if all profiles start with 'ht-'"
  category: conformance
  predicate: id.startsWith('ht-')
  files: 
    - "!guides/firely-sample-ig/ImplementationGuide.json"

- name: example-codeableconcept-display-text
  error: https://acme.com/fhir/qa/errors|example-codeableconcept-display-text
  error-message: "CodeableConcept does not have a .coding.display or a .text value or an extension." 
  severity: warning
  status: "Checking if all CodeableConcept elements contain either a coding.display or a text value if no Coding exists or has an extension (e.g. a nullFlavor or data-absent-reason extension)."
  category: example
  predicate: descendants().where($this.is(CodeableConcept))
                .all(
                (coding.display.exists() or (coding.exists().not() and text.exists())) or extension.exists())

```

Run QC locally via Firely Terminal (requires paid Simplifier.net license for `.rules.yaml`):

```bash
fhir restore    # install deps
fhir check customrules    # runs QC ruleset + full validation in one pass
```

---


## Reviewing Validation Results

After validating multiple files with `--push`, the stack contains the original resources interleaved with OperationOutcomes. Bundle everything and inspect:

```bash
fhir bundle      # collect all stack items into one Bundle
fhir show        # display the bundle for inspection

# Save to file if output is too large for the console
fhir save validation-results.json --json --pretty
```

---

## Package Management

```bash
# Install a specific FHIR package
fhir install hl7.fhir.r4.core 4.0.1
fhir install hl7.fhir.us.core 6.1.0
fhir install hl7.fhir.uv.ips 1.1.0

# Install all dependencies declared in package.json
fhir restore

# Show full resolved dependency tree
fhir scope

# Set active FHIR version
fhir spec r4
fhir spec r5
```

---

## Resource Stack Operations

Firely Terminal maintains an in-memory "stack" of resources for interactive workflows:

```bash
fhir push patient.json          # load resource onto stack
fhir push ./resources/          # push all resources in a folder
fhir stack                      # show contents of the stack
fhir show                       # display top resource on stack
fhir pop                        # remove top resource from stack

fhir save output.json           # export top of stack to JSON
fhir save output.xml            # export to XML

fhir snapshot                   # generate StructureDefinition snapshot from stack
fhir expand                     # expand ValueSet on top of stack

fhir bundle                     # create Bundle from all stack resources
fhir split bundle.json          # unpack Bundle → individual resource files
```

---

## FHIR Server Interactions

```bash
fhir server add myfhir https://server.fire.ly
fhir server add local http://localhost:8080/fhir

fhir search myfhir Patient name=Smith
fhir read myfhir Patient/123
fhir post myfhir patient.json
fhir put myfhir Patient/123 patient-updated.json
```

---

## Simplifier.net Integration

Simplifier.net is a FHIR specification development, collaboration, and publishing platform. Firely Terminal integrates directly for project sync and IG management.

### Authentication

```bash
fhir login email=you@example.com password=yourpassword
fhir logout
```

### Project Management

```bash
fhir projects                                  # list all your Simplifier projects
fhir project clone <project-name>              # clone project to a subfolder
fhir project link <project-urlkey>             # link current folder to a Simplifier project
fhir project sync                              # sync current folder ↔ Simplifier
```

### File Sync

```bash
fhir project push               # push local files → Simplifier project
fhir project pull               # pull Simplifier project → local
fhir project sync               # push all local changes (deduces project from folder name)
fhir pack                       # create a deployable FHIR package from current project
```

### Recommended Workflow

```bash
# Initial setup
fhir login email=you@example.com password=yourpassword
fhir project clone my-ig-project

# Daily development loop
cd my-ig-project
# ... make edits to resources, FSH files, etc.
fhir restore                       # install/update dependencies
fhir check                         # validate + QC rules (paid), or use fhir validate loop
fhir project sync                  # push to Simplifier for review
```

---


## CI/CD: GitHub Actions Pipeline

Use the official Firely Terminal GitHub Action for automated validation:

```yaml
# .github/workflows/fhir-validate.yml
name: FHIR Validation
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: FirelyTeam/firely-terminal-pipeline@v1
        with:
          PATH_TO_CONFORMANCE_RESOURCES: 'input/resources'
          DOTNET_VALIDATION_ENABLED: 'true'
          JAVA_VALIDATION_ENABLED: 'false'       # optional HL7 Java validator
          SUSHI_ENABLED: 'false'                 # set true if using FSH
          PATH_TO_QUALITY_CONTROL_RULES: '.rules'
          SIMPLIFIER_USERNAME: ${{ secrets.SIMPLIFIER_USERNAME }}
          SIMPLIFIER_PASSWORD: ${{ secrets.SIMPLIFIER_PASSWORD }}
```

**Key configuration options:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `PATH_TO_CONFORMANCE_RESOURCES` | required | Folder(s) with StructureDefinitions, ValueSets, CodeSystems |
| `DOTNET_VALIDATION_ENABLED` | `true` | Run Firely .NET SDK validator |
| `JAVA_VALIDATION_ENABLED` | `false` | Also run official HL7 Java validator |
| `SUSHI_ENABLED` | `false` | Compile FSH before validating |
| `PATH_TO_QUALITY_CONTROL_RULES` | — | Path to `.rules.yaml` files (omit `.rules.yaml` extension) |
| `CLOSE_SLICING_FOR_VALIDATION` | `false` | Treat all slicing as closed (stricter) |
| `EXPECTED_FAILS` | — | Mark specific steps as expected failures |

---

## Simplifier.net IG Editor Widgets

When authoring IG pages (Markdown) in Simplifier's IG editor, embed resource views with:

```
{{tree:http://example.org/StructureDefinition/MyProfile}}
{{json:http://example.org/StructureDefinition/MyProfile}}
{{xml:http://example.org/StructureDefinition/MyProfile}}
{{table:http://example.org/StructureDefinition/MyProfile}}
{{render:http://example.org/StructureDefinition/MyProfile}}
```

FQL tables for dynamic content:

```
<fql​>
from StructureDefinition
    where url in ('http://example.org/fhir/StructureDefinition/xxx' | 'http://example.org/fhir/StructureDefinition/yyy')
select
    Name: name,
    Description: description,
    Version: version,
    Status: status,
    URL: url
</fql​>
```

---

## FQL (Firely Query Language)

FQL is a query language that allows you to retrieve, filter and project data from any data source containing FHIR Resources. It brings the power of three existing languages: SQL, JSON and FhirPath. SQL-like queries across a FHIR project :

```bash
# Run FQL query on all resources in scope
fhir query 'from StructureDefinition select name, url' --output json

# Common queries
fhir query 'from StructureDefinition where status = "active" select name, url'
fhir query 'from ValueSet select name, url, status'
```


