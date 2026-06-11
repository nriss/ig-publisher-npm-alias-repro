# Bug: NPM alias multi-version dependencies fail with `Name '...' already exists` when `multilanguage-format: true`

## Summary

When an IG declares two versions of the same FHIR package using the npm alias syntax (supported since SUSHI [#1463](https://github.com/FHIR/sushi/issues/1463)), the publisher throws:

```
Publishing Content Failed: Name 'hl7.fhir.us.core' already exists (value = "7.0.0")
org.hl7.fhir.utilities.json.JsonException: Name 'hl7.fhir.us.core' already exists (value = "7.0.0")
```

This happens on **all tested publisher versions** (2.0.29 → 2.2.8) when the IG template sets `"multilanguage-format": true`.

## Reproduction

Minimal reproduction repository: https://github.com/nriss/ig-publisher-npm-alias-repro

`sushi-config.yaml`:
```yaml
dependencies:
  uscore700@npm:hl7.fhir.us.core:
    id: uscore700
    version: 7.0.0
  uscore610@npm:hl7.fhir.us.core:
    id: uscore610
    version: 6.1.0
```

`ig.ini`:
```ini
[IG]
ig = fsh-generated/resources/ImplementationGuide-org.example.npm-alias-repro.json
template = https://github.com/HL7/ig-template-base2
```

```bash
sushi .
java -jar publisher.jar -ig ig.ini
```

## Expected behavior

Build succeeds. The generated `package.json` contains both alias entries:
```json
"dependencies": {
  "uscore700@npm:hl7.fhir.us.core": "7.0.0",
  "uscore610@npm:hl7.fhir.us.core": "6.1.0"
}
```

## Actual behavior

```
Publishing Content Failed: Name 'hl7.fhir.us.core' already exists (value = "7.0.0")
org.hl7.fhir.utilities.json.JsonException: Name 'hl7.fhir.us.core' already exists (value = "7.0.0")
    at org.hl7.fhir.utilities.json.model.JsonObject.add(JsonObject.java:34)
    at org.hl7.fhir.r5.utils.NPMPackageGenerator.buildPackageJson(NPMPackageGenerator.java:273)
    at org.hl7.fhir.r5.utils.NPMPackageGenerator.<init>(NPMPackageGenerator.java:150)
    at org.hl7.fhir.igtools.publisher.PublisherIGLoader.load(PublisherIGLoader.java:2271)
```

## Root cause analysis *(by Claude Sonnet 4.6)*

The publisher correctly handles alias dependencies in `PublisherIGLoader.load()` by:
1. Detecting `@npm:` in the `packageId`
2. Stripping the alias prefix from `packageId`
3. Setting an `IG_DEP_ALIASED` user data flag on the `packageIdElement`

This works for the **main** `NPMPackageGenerator` call (line 2260), which uses `pf.publishedIg` directly and has the flags set.

However, when `multilanguage-format: true` is set in the template, the publisher generates language-variant packages via a second loop (line 2267–2272). This loop creates a copy of the IG using `copyToLanguage()`:

```java
// PublisherIGLoader.java ~line 2267
if (isNewML()) {
  for (String l : allLangs()) {
    ImplementationGuide vig = (ImplementationGuide) pf.langUtils.copyToLanguage(
        pf.publishedIg, l, true, pf.defaultTranslationLang, igf.getErrors());
    pf.lnpms.put(l, new NPMPackageGenerator(..., vig, ...)); // ← fails here
  }
}
```

`copyToLanguage()` — like `copy()` — does **not preserve `UserData`** (it is `transient` by default in the HAPI FHIR R5 model, only copied when `Base.isCopyUserData()` returns `true`). The copy therefore has no `IG_DEP_ALIASED` flag, so `NPMPackageGenerator.buildPackageJson()` falls into the plain `dep.add(packageId, version)` branch for both alias entries, causing the duplicate key error.

## Affected versions

All tested publisher versions: **2.0.29, 2.1.2, 2.2.6, 2.2.7, 2.2.8**.

The feature was introduced in `hapifhir/org.hl7.fhir.core` at commit `331bcab` (2025-05-17, "Support for NPM Aliases") and the `IG_DEP_ALIASED` flag-setting code has been present in the publisher since at least 2.0.29 — but the language-pack copy path has never propagated the flag.

## Suggested fix *(by Claude Sonnet 4.6, to be reviewed)*

> ⚠️ The following fix was suggested by an AI assistant based on the analysis above. I am not a Java developer — please review and adjust as needed.

After each `copyToLanguage()` (and `publishedIg.copy()` in the `generateVersions` loop), re-apply the `IG_DEP_ALIASED` flag from the original `pf.publishedIg` to the copy, since the flag is positional (the deps list order is preserved by both copy methods):

```java
// Re-apply IG_DEP_ALIASED flags lost during copy (UserData is transient)
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

Then call `reapplyAliasFlagsTo(vig)` after each copy:

```java
// language-pack loop (~line 2269):
ImplementationGuide vig = (ImplementationGuide) pf.langUtils.copyToLanguage(...);
reapplyAliasFlagsTo(vig);
pf.lnpms.put(l, new NPMPackageGenerator(..., vig, ...));

// version-variant loop (~line 2287):
ImplementationGuide vig = pf.publishedIg.copy();
reapplyAliasFlagsTo(vig);
checkIgDeps(vig, v);
pf.vnpms.put(v, new NPMPackageGenerator(..., vig, ...));
```

## Workaround

Setting `"multilanguage-format": false` in the template's `config.json` disables the language-pack generation code path and avoids the error. The `i18n-default-lang` parameter continues to work for locale-based narrative rendering.
