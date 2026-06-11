# ig-publisher-npm-alias-repro

Minimal FHIR IG to reproduce a bug in [HL7/fhir-ig-publisher](https://github.com/HL7/fhir-ig-publisher): multi-version npm alias dependencies fail with `Publishing Content Failed: Name '...' already exists` when the template sets `multilanguage-format: true`.

See issue: <!-- link to be added -->

## What this reproduces

The IG declares two versions of `hl7.fhir.us.core` using the npm alias syntax:

```yaml
dependencies:
  uscore700@npm:hl7.fhir.us.core:
    id: uscore700
    version: 7.0.0
  uscore610@npm:hl7.fhir.us.core:
    id: uscore610
    version: 6.1.0
```

**Expected**: build succeeds, `package.json` contains both entries with their alias keys.  
**Actual**: `Publishing Content Failed: Name 'hl7.fhir.us.core' already exists`

## How to reproduce

```bash
npm install -g fsh-sushi
curl -L https://github.com/HL7/fhir-ig-publisher/releases/latest/download/publisher.jar -o input-cache/publisher.jar
java -jar input-cache/publisher.jar -ig ig.ini
```

See the [CI run](.github/workflows/build.yml) for an automated reproduction.
