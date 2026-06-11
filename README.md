# ig-publisher-npm-alias-repro

Minimal FHIR IG to reproduce a bug in [HL7/fhir-ig-publisher](https://github.com/HL7/fhir-ig-publisher): multi-version npm alias dependencies fail with `Publishing Content Failed: Name '...' already exists` when the template sets `multilanguage-format: true`.

See issue: <!-- link to be added once filed -->

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

**Expected**: build succeeds, `package.json` contains both alias entries.  
**Actual**: `Publishing Content Failed: Name 'hl7.fhir.us.core' already exists (value = "7.0.0")`

## How to reproduce

```bash
npm install -g fsh-sushi
sushi .
curl -L https://github.com/HL7/fhir-ig-publisher/releases/latest/download/publisher.jar \
  -o input-cache/publisher.jar
java -jar input-cache/publisher.jar -ig ig.ini
```

The CI workflow reproduces the failure automatically: see [Actions](.github/workflows/build.yml).

## Proposed fix

> This is a suggested approach — please see [`issue-fhir-ig-publisher.md`](issue-fhir-ig-publisher.md) for the full analysis.

The root cause is that `copyToLanguage()` (called when `multilanguage-format: true`) creates a copy of the `ImplementationGuide` that loses the `IG_DEP_ALIASED` `UserData` flag set on alias dependency elements. Without this flag, `NPMPackageGenerator.buildPackageJson()` adds both alias entries using the plain package name, causing a duplicate key error in the JSON object.

**Suggested fix in `PublisherIGLoader.java`**: after each `copyToLanguage()` / `publishedIg.copy()` used for NPM variant generation, re-propagate the flag from `pf.publishedIg` to the copy:

```java
private void reapplyAliasFlagsTo(ImplementationGuide vig) {
  List<ImplementationGuide.ImplementationGuideDependsOnComponent> origDeps =
      pf.publishedIg.getDependsOn();
  List<ImplementationGuide.ImplementationGuideDependsOnComponent> vigDeps =
      vig.getDependsOn();
  for (int i = 0; i < Math.min(origDeps.size(), vigDeps.size()); i++) {
    if (origDeps.get(i).getPackageIdElement()
        .hasUserData(UserDataNames.IG_DEP_ALIASED)) {
      vigDeps.get(i).getPackageIdElement()
          .setUserData(UserDataNames.IG_DEP_ALIASED, true);
    }
  }
}
```

**Workaround**: set `"multilanguage-format": false` in the template's `config.json`.
