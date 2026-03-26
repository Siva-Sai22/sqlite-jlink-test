# JLink + SQLite-JDBC Integration Report

## Executive Summary

This project demonstrates that bundling sqlite-jdbc with jlink in a modular Java 21 application works without requiring complex workarounds like moditect or shadow-jar plugins. The key insight is that using the latest sqlite-jdbc version (3.47.1.0+) resolves the historical jlink compatibility issues.

## Problem Statement

Java 9+ Modules (JPMS) and jlink have known issues with third-party database drivers like sqlite-jdbc because:

1. **Automatic Modules**: Many JDBC drivers were "automatic modules" - JARs without explicit `module-info.java` that derive module names from the JAR filename
2. **jlink Limitation**: jlink cannot include automatic modules in custom runtime images
3. **Transitive Dependencies**: Even when the main library supports modules, transitive dependencies may not

### Historical Issues with sqlite-jdbc

| Version | Issue |
|---------|-------|
| < 3.39.4 | No module-info.java, treated as automatic module |
| 3.39.4+ | Has module-info.java but slf4j was a hard requirement |
| 3.43.2.1+ | Downgraded slf4j to 1.7.x which broke jlink |
| 3.47.0+ | slf4j 2.x with `requires static` - **works with jlink** |

## Solution Approach

### Technology Stack

| Component | Version |
|-----------|---------|
| Java | 21 (OpenJDK) |
| Gradle | 8.10.2 |
| sqlite-jdbc | 3.47.1.0 |

### Project Structure

```
sqlite-jlink-test/
├── build.gradle              # Gradle build configuration
├── settings.gradle           # Gradle settings
├── gradlew                   # Gradle wrapper
├── gradlew.bat               # Windows Gradle wrapper
├── src/main/java/
│   ├── module-info.java      # Java module descriptor
│   └── com/example/app/
│       └── Main.java         # Main application class
└── build/
    └── jlink-image/          # Generated jlink image
        ├── bin/java           # Custom JRE executable
        └── lib/               # Modular JARs
```

## Build Process

### Step 1: Build the Project

```bash
./gradlew clean build
```

### Step 2: Verify Modules in Image

```bash
./build/jlink-image/bin/java --list-modules | grep -E "(sqlite|example)"
```

Output:
```
com.example.app
org.xerial.sqlitejdbc@3.47.1.0
```

### Step 3: Run the Bundled Application

```bash
./build/jlink-image/bin/java --module com.example.app/com.example.app.Main
```

Output:
```
Connection Successful!
```

## Conclusion

This proof-of-concept demonstrates that the jlink + sqlite-jdbc integration challenge has been solved in recent sqlite-jdbc versions. The solution is simple, requires no workarounds, and provides a clean path forward for modular Java applications using jlink and jpackage.
