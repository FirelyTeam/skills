---
name: fhir-validation
description: FHIR validation specialist. Use when validating FHIR resources, profiles, or Implementation Guides against conformance rules. Covers Firely Terminal (.NET, profile-aware, free + paid QC rules), HL7 Java validator (thorough, profile-aware), and Python schema validation. Guides project assessment, runs both profile-aware validators for consistency, and supports CI/CD via FirelyTeam/firely-terminal-pipeline.
---

# FHIR Validation Skill

Expert guidance for validating FHIR resources, profiles, and Implementation Guides. This skill assesses your project setup, selects the right validators, and runs them in the right order.

> **Firely Terminal commands:** Only use commands that exist in Firely Terminal. This [https://docs.fire.ly/projects/Firely-Terminal/Command-Reference.html](https://docs.fire.ly/projects/Firely-Terminal/Command-Reference.html) is the authoritative source — do not invent command names, flags, or parameters. If you are unsure whether a command or flag exists, look it up in that file before suggesting it.

---

## Validation Approaches

There are three approaches. Use profile-aware validators (Firely Terminal and/or Java) for real conformance checking.

| Approach | Tool | Profile-aware | Speed | Requires |
|---|---|---|---|---|
| **Firely Terminal** (.NET SDK) | `fhir check` / `fhir validate` | Yes | Fast | .NET SDK 8 |
| **HL7 Java Validator** | `validator_cli.jar` | Yes | Slower | Java |
| **Python fhir.resources** | `fhir.resources` lib | No — schema only | Fastest | Python |

**Firely Terminal** is built around an internal resource stack. All resource operations (push, validate, snapshot, etc.) act on that stack. It uses the same validation engine as Simplifier.net and Forge.
- Free tier (with Simplifier.net account): full profile-aware validation via `fhir check` or `fhir validate`
- Paid tier: adds QC business rules via `.rules.yaml` on top of `fhir check`
- No account: `fhir validate` loop only

**HL7 Java Validator** is the official HL7 reference validator — thorough but slower. Best run alongside Firely Terminal to cross-check results.

**Python fhir.resources** performs JSON schema validation only. It does **not** check profile constraints, slices, or FHIRPath invariants. Use only for in-app embedded validation where no external tool can be invoked.

---

## Before You Start

Three quick checks before running any validator:

1. **FSH project?** — if `sushi-config.yaml` and `.fsh` files in `input/fsh/` are present, compile first:
    ```powershell
   sushi build    # compiles FSH → fsh-generated/resources/
   ```
   Fix any SUSHI errors before validating. Never edit `fsh-generated/` — it is regenerated on every build.

2. **Check tooling availability** (if unsure):
    ```powershell
   dotnet --version   # .NET SDK 8+ for Firely Terminal
   java -version      # Java for HL7 Java validator
   ```

---

## Phase 1 — Firely Terminal Validation

For any Firely Terminal command not shown in this skill, look it up in `references/firely_terminal_commands.md` before suggesting it — never guess flags or subcommands.

### Install

```powershell
dotnet tool install -g firely.terminal   # first time
dotnet tool update -g firely.terminal    # to update
```
### Install dependencies
Run `fhir restore` (reads `package.json`). Without this, validators may produce spurious "unknown profile" errors.

### Run validation

**Preferred — `fhir check`** (full profile-aware validation; also runs QC rules if `.rules.yaml` files are present):

Always try `fhir check` first. It runs full validation regardless of whether `.rules.yaml` files exist — the paid license only adds the QC rules layer on top.

```powershell
fhir check
```

If not authenticated:

```powershell
fhir login email=you@example.com password=yourpassword
fhir check
```

**Fallback — `fhir validate` loop** (use only when no Simplifier.net account is available):

Use `fhir validate [file] --push` to validate a file and keep the resulting OperationOutcome on the stack for later review. On Windows, **avoid absolute backslash paths** — pass relative paths or use `Push-Location` to cd into the folder first and pass a bare filename.

```powershell
# PowerShell
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

### Review and process results

After validation, the stack contains the original resources interleaved with OperationOutcomes. Bundle and inspect:

```powershell
fhir bundle      # collect all stack items into one Bundle
fhir save validation-results.json --json --pretty # Save to file for large result sets or programmatic processing
```

**Process the saved results with Python:**

```python
import json
from pathlib import Path

def summarize_validation_results(path="validation-results.json"):
    data = json.loads(Path(path).read_text())
    entries = data.get("entry", []) if data.get("resourceType") == "Bundle" else [{"resource": data}]

    issues = []
    for entry in entries:
        resource = entry.get("resource", {})
        if resource.get("resourceType") == "OperationOutcome":
            for issue in resource.get("issue", []):
                issues.append({
                    "severity": issue.get("severity"),
                    "message": (issue.get("details") or {}).get("text") or issue.get("diagnostics", ""),
                    "location": issue.get("expression") or issue.get("location") or [],
                })

    by_severity = {}
    for i in issues:
        by_severity.setdefault(i["severity"], []).append(i)

    for severity in ("error", "warning", "information"):
        items = by_severity.get(severity, [])
        if items:
            print(f"\n{severity.upper()} ({len(items)}):")
            for i in items:
                loc = ", ".join(i["location"]) if isinstance(i["location"], list) else str(i["location"])
                print(f"  {i['message']}" + (f"  [{loc}]" if loc else ""))

summarize_validation_results()
```

---

## Phase 2 — HL7 Java Validator

Run this alongside Phase 1 to cross-check results.

### Setup

Locate an existing `validator_cli.jar` anywhere in the project first:

```powershell
# PowerShell
$jarItem = Get-ChildItem -Recurse -Filter "validator_cli.jar" -ErrorAction SilentlyContinue | Select-Object -First 1
if ($jarItem) {
    $jar = $jarItem.FullName -replace '\\', '/'
} else {
    New-Item -ItemType Directory -Force -Path "./input-cache" | Out-Null
    Invoke-WebRequest -Uri "https://github.com/hapifhir/org.hl7.fhir.core/releases/latest/download/validator_cli.jar" `
      -OutFile "./input-cache/validator_cli.jar"
    $jar = "./input-cache/validator_cli.jar"
}
```

### Run validation

```powershell
# Validate against base FHIR spec
java -jar $jar patient.json -version 4.0.1

# Validate against a specific IG and profile
java -jar $jar patient.json `
    -version 4.0.1 `
    -ig hl7.fhir.us.core#6.1.0 `
    -profile http://hl7.org/fhir/us/core/StructureDefinition/us-core-patient

# Validate multiple files
Get-ChildItem ./input/resources -Filter *.json | ForEach-Object {
    java -jar $jar $_.FullName -version 4.0.1
}
```

### Local conformance resources (`-ig`) guidance

When validating against local project conformance artifacts, **do not use `-ig .`**. Point `-ig` to concrete folders that actually contain IG resources (for example `StructureDefinition-*.json`, `ValueSet-*.json`, `CodeSystem-*.json`).

Typical local folders to include when they exist:
- `./fsh-generated/resources` (compiled FSH output)
- `./input/resources` (hand-authored conformance resources)
- `./package` (IG Publisher package output, if present)

Use one `-ig` per folder or package:

```powershell
java -jar $jar $OUTPUT_FILE `
    -version 4.0.1 `
    -ig ./fsh-generated/resources `
    -ig ./input/resources `
    -ig "nictiz.fhir.nl.r4.zib2020#0.12.0-beta.4" `
    -ig "nictiz.fhir.nl.r4.nl-core#0.12.0-beta.4"
```

PowerShell helper to add only existing local IG folders:

```powershell
$candidateIgDirs = @("./fsh-generated/resources", "./input/resources", "./package")
$igArgs = @()
foreach ($dir in $candidateIgDirs) {
        if (Test-Path $dir) {
                $igArgs += @("-ig", $dir)
        }
}

java -jar $jar $OUTPUT_FILE -version 4.0.1 `
    @igArgs `
    -ig "nictiz.fhir.nl.r4.zib2020#0.12.0-beta.4" `
    -ig "nictiz.fhir.nl.r4.nl-core#0.12.0-beta.4"
```

---

## Cross-Checking Results

After running both validators, compare their output:

- **Issues flagged by both validators** — high confidence, real conformance problems; prioritise fixing these
- **Issues flagged by only one validator** — investigate; may be a validator-specific edge case or tool difference
- Report any discrepancies clearly so the user can decide which to act on

---

## Fixing Errors in FSH Projects

If the project uses FSH (SUSHI), **never edit files in `fsh-generated/`** — that folder is deleted and regenerated on every `sushi build`. All fixes must be made in the `.fsh` source files under `input/fsh/`.

### Tracing an error back to its FSH source

Validation errors reference the generated JSON file (e.g. `fsh-generated/resources/StructureDefinition-MyProfile.json`). The naming pattern is `{ResourceType}-{id}.json`. To find the FSH source:

1. Note the resource `id` from the error (e.g. `MyProfile`)
2. Search for it in `input/fsh/`: `Get-ChildItem input/fsh -Recurse -Filter *.fsh | Select-String 'Id: MyProfile'`
3. The containing `.fsh` file is where the fix goes

---

## Python — In-App / Embedded Validation Only

Use `fhir.resources` only when validation logic must be embedded inside an application without invoking external tools. This is **schema-only** — profiles, slices, and FHIRPath invariants are not checked.

```python
from fhir.resources import get_fhir_model_class

def validate_resource_schema(resource_data: dict) -> bool:
    """Schema-level validation only. Use Firely Terminal or Java validator for full conformance."""
    try:
        resource_type = resource_data.get('resourceType')
        resource_class = get_fhir_model_class(resource_type)
        resource_class(**resource_data)
        return True
    except Exception as e:
        print(f"Validation error: {e}")
        return False
```

---

## CI/CD

Validation can be automated in CI/CD pipelines using the official `FirelyTeam/firely-terminal-pipeline` GitHub Action, which runs Firely Terminal (and optionally the Java validator) on every push or pull request.

See `references/firely_terminal.md` for a full workflow example, or the source documentation at https://github.com/FirelyTeam/firely-terminal-pipeline.

---

## References

- **Authoritative Firely Terminal command reference** (all commands, parameters, and documentation): https://docs.fire.ly/projects/Firely-Terminal/Command-Reference.html
- `references/firely_terminal.md` — workflow guide including, quality control, CI/CD GitHub Actions setup
- Firely Terminal docs: https://docs.fire.ly/projects/Firely-Terminal/
- firely-terminal-pipeline: https://github.com/FirelyTeam/firely-terminal-pipeline
