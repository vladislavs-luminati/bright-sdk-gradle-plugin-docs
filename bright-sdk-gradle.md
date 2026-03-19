# bright-sdk-gradle – Full Documentation

## Overview

`bright-sdk-gradle` is a Gradle plugin (ID: `com.brightdata.brightsdk-gradle`) that automates downloading, extracting, and injecting the Bright SDK AAR into Android projects. It supports caching, Maven Local publishing, version pinning, and multi-project setups.

- **Plugin ID:** `com.brightdata.brightsdk-gradle`
- **Maven coordinates:** `com.brightdata:bright-sdk-gradle:1+`
- **Current version:** 1.2.5
- **Java source:** `src/main/java/com/brightdata/BrightSdkPlugin.java`
- **GitHub:** https://github.com/BrightSDK/gradle-plugin
- **Javadoc:** https://vladislavs-luminati.github.io/bright-sdk-gradle-plugin-docs/com/brightdata/package-summary.html
- **Guide:** https://vladislavs-luminati.github.io/bright-sdk-gradle-plugin-guide
- **Maven Central:** https://central.sonatype.com/artifact/com.brightdata/bright-sdk-gradle

---

## Prerequisites

- **JDK:** 17+ (plugin is compiled targeting Java 8 compatibility)
- **Gradle:** 6.7+
- **Internet access:** Required unless using `useLocalCache=true` with a pre-populated cache

---

## Installation

### Step 1 – Create `brightsdk.gradle`

Place in your `app/` module or top-level project:

```gradle
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.brightdata:bright-sdk-gradle:1+'
    }
}

ext.brightsdk = [
    dir:              "libs",
    version:          "latest",
    consumerProjects: "app",
    useLocalCache:    false,
    useMavenLocal:    false,
]

apply plugin: 'com.brightdata.brightsdk-gradle'

tasks.named("preBuild").configure {
    dependsOn installBrightSdk
}
```

### Step 2 – Apply in `build.gradle`

```gradle
apply from: 'brightsdk.gradle'
```

### Step 3 – Run the build

```bash
./gradlew build
```

The plugin will automatically download the SDK (or use cached copy), extract the AAR into `libs/`, and inject dependencies before `preBuild`.

---

## Configuration

All options can be set in `gradle.properties`, in `ext.brightsdk[]`, or via `-P` CLI arguments.

| Property (gradle.properties) | CLI (`-P`) | Description | Default |
|---|---|---|---|
| `brightsdk.version` | `-PsdkVersion` | SDK version to use. Use `latest` for auto-resolve | `latest` |
| `brightsdk.dir` | `-Pdir` | Output directory for the extracted AAR | `libs` |
| `brightsdk.consumerProjects` | `-PconsumerProjects` | Comma-separated module names to receive `mavenLocal()` | `app` |
| `brightsdk.useLocalCache` | `-PuseLocalCache` | Store cached downloads in project dir (`.brightsdk/`) instead of Gradle cache | `false` |
| `brightsdk.useMavenLocal` | `-PuseMavenLocal` | Publish AAR to Maven Local and auto-inject as `implementation` dependency | `false` |
| `brightsdk.projectDir` | `-PprojectDir` | Override the project directory (default: Gradle project dir) | `.` |

### Example `gradle.properties`

```properties
brightsdk.version=latest
brightsdk.dir=libs
brightsdk.useLocalCache=false
brightsdk.useMavenLocal=false
brightsdk.consumerProjects=app
```

### Example with pinned version

```properties
brightsdk.version=1.482.706
```

---

## Gradle Tasks

| Task | Depends On | Description |
|---|---|---|
| `downloadBrightSdk` | — | Downloads `bright_sdk_android-{version}.tar.gz` from CDN to cache dir. Skipped if file already in cache |
| `extractBrightSdk` | `downloadBrightSdk` | Unpacks the `.tar.gz` and copies `bright_sdk.aar` (or `bright_sdk-{version}.aar`) into the configured `dir` |
| `updateBrightSdk` | `extractBrightSdk` | Convenience task: download + extract. Prints "Updated BrightSDK to version X" |
| `publishBrightSdkToMavenLocal` | `extractBrightSdk` | Publishes AAR to `~/.m2/repository/com/bright/sdk/{version}/`. No-op if already published or `useMavenLocal=false` |
| `installBrightSdk` | `publishBrightSdkToMavenLocal` | Full pipeline: download → extract → publish to Maven Local |
| `checkBrightSdkInMavenLocal` | — | Diagnostic task: checks whether the AAR + POM are in `~/.m2/repository/com/bright/sdk/{version}/` |

### Task flow (standard)

```
preBuild → installBrightSdk → publishBrightSdkToMavenLocal → extractBrightSdk → downloadBrightSdk
```

---

## Internal Architecture

### `BrightSdkPlugin.java` – `apply(Project project)`

The main lifecycle method. On application:

1. Reads and validates plugin parameters (`parseBrightSdkArgs()`)
2. If version is `latest`: fetches `https://bright-sdk.com/sdk_api/sdk/versions` → gets `android` field
3. Caches versions JSON to `{cacheDir}/sdk_versions.json` (expires after 24 hours)
4. Configures Maven Local publishing (if `useMavenLocal=true`)
5. Registers all tasks (`downloadBrightSdk`, `extractBrightSdk`, `updateBrightSdk`, `installBrightSdk`, `checkBrightSdkInMavenLocal`)
6. In `afterEvaluate`: adds `mavenLocal()` to consumer projects; imports SDK + dependencies

### Version resolution

```
sdkVersion = "latest"  →  GET https://bright-sdk.com/sdk_api/sdk/versions
                          →  response["android"]  →  actual version string
sdkVersion = "1.x.y"   →  used as-is (fixed version mode skips fetch)
```

### Download URL pattern

```
https://cdn.bright-sdk.com/static/bright_sdk_android-{sdkVersion}.tar.gz
```

### Cache directory

| `useLocalCache` | Cache location |
|---|---|
| `false` (default) | `~/.gradle/caches/.brightsdk/` |
| `true` | `{projectDir}/.brightsdk/` |

### Maven Local artifact path

```
~/.m2/repository/com/bright/sdk/{sdkVersion}/sdk-{sdkVersion}.aar
~/.m2/repository/com/bright/sdk/{sdkVersion}/sdk-{sdkVersion}.pom
```

Maven coordinates published: `com.bright:sdk:{sdkVersion}` (packaging: `aar`)

### Dependency injection

After `afterEvaluate`, the plugin adds these `implementation` dependencies to consumer projects (loaded from bundled `dependencies.json`):

| Group | Artifact | Version |
|---|---|---|
| `com.google.android.gms` | `play-services-ads-identifier` | `18.0.1` |
| `androidx.core` | `core` | `1.9.0` |
| `androidx.constraintlayout` | `constraintlayout` | `2.1.4` |

When `useMavenLocal=true`, also adds: `com.bright:sdk:{sdkVersion}`

Each dependency is only added once (deduplication check against existing `implementation` configuration).

### Multi-project support

When `useMavenLocal=true`, the plugin scans all sub-projects and adds `mavenLocal()` only to modules listed in `consumerProjects` (comma-separated). Safe to set to multiple consumers, e.g. `consumerProjects=app,react-native-bright-sdk`.

---

## AAR file naming

| Condition | AAR filename |
|---|---|
| `sdkVersion = "latest"` | `bright_sdk.aar` |
| `sdkVersion = "1.2.3"` (specific) | `bright_sdk-1.2.3.aar` |

---

## Publishing (for plugin maintainers)

The plugin itself is published to Maven Central with these coordinates:
- **groupId:** `com.brightdata`
- **artifactId:** `bright-sdk-gradle`
- **version:** set in `build.gradle` (`1.2.5` as of this writing)

Publishing pipeline uses `tech.yanand.maven-central-publish` with GPG signing. Requires `gpgSign.keyFile`, `gpgSign.password`, `ossrh.username`, `ossrh.password` properties.

```bash
./gradlew publishToMaven
```

---

## Quick Reference Card

```gradle
// Minimal setup (auto-resolve latest, use file AAR in libs/)
ext.brightsdk = [version: "latest", dir: "libs"]
apply plugin: 'com.brightdata.brightsdk-gradle'

// Pin a specific version, use Maven Local dependency injection
ext.brightsdk = [version: "1.482.706", useMavenLocal: true, consumerProjects: "app"]
apply plugin: 'com.brightdata.brightsdk-gradle'
```

```bash
# Check if SDK in Maven Local
./gradlew checkBrightSdkInMavenLocal

# Force re-download and update the SDK
./gradlew updateBrightSdk --refresh-dependencies

# Full install (download → extract → Maven Local → inject)
./gradlew installBrightSdk
```

---

## Links

- GitHub: https://github.com/BrightSDK/gradle-plugin
- Javadoc: https://vladislavs-luminati.github.io/bright-sdk-gradle-plugin-docs/com/brightdata/package-summary.html
- Guide: https://vladislavs-luminati.github.io/bright-sdk-gradle-plugin-guide
- Maven Central: https://central.sonatype.com/artifact/com.brightdata/bright-sdk-gradle
- Support: sdk@brightdata.com
