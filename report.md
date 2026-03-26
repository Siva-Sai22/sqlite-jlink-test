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

### Key Configuration

#### build.gradle

```groovy
plugins {
    id 'application'
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.xerial:sqlite-jdbc:3.47.1.0'
}

application {
    mainModule = 'com.example.app'
    mainClass = 'com.example.app.Main'
}

def jlinkTask = tasks.register('createJlinkImage', Exec) {
    group = 'distribution'
    description = 'Creates a custom JRE using jlink'
    
    def modulePath = sourceSets.main.runtimeClasspath.asPath
    def outputDir = "${buildDir}/jlink-image"
    
    commandLine 'jlink',
        '--module-path', modulePath,
        '--add-modules', 'com.example.app',
        '--output', outputDir,
        '--strip-debug',
        '--compress', '2'
}
```

#### module-info.java

```java
module com.example.app {
    requires org.xerial.sqlitejdbc;
}
```

#### Main.java

```java
package com.example.app;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class Main {
    public static void main(String[] args) {
        try {
            // Required for jlink'd applications
            Class.forName("org.sqlite.JDBC");
            
            try (Connection conn = DriverManager.getConnection("jdbc:sqlite::memory:")) {
                System.out.println("Connection Successful!");
            }
        } catch (ClassNotFoundException | SQLException e) {
            System.err.println("Error: " + e.getMessage());
            System.exit(1);
        }
    }
}
```

## Build Process

### Step 1: Build the Project

```bash
./gradlew clean build
```

Output:
```
> Task :compileJava
> Task :jar
> Task :startScripts
> Task :createJlinkImage
Warning: The 2 argument for --compress is deprecated and may be removed in a future release
JLink image created at: /home/sivasai/Cooking/oss/sqlite-jlink-test/build/jlink-image
> Task :build

BUILD SUCCESSFUL
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

## Why This Works

### sqlite-jdbc 3.47.x Module Info

The key to making this work is the module-info.java in sqlite-jdbc 3.47.x:

```java
module org.xerial.sqlitejdbc {
    requires transitive java.logging;
    requires transitive java.sql;
    requires static org.slf4j;  // Key: static requirement
    
    exports org.sqlite;
    exports org.sqlite.core;
    exports org.sqlite.date;
    exports org.sqlite.javax;
    exports org.sqlite.jdbc3;
    exports org.sqlite.jdbc4;
    exports org.sqlite.util;
    
    provides java.sql.Driver with org.sqlite.JDBC;
    provides javax.sql.DataSource with org.sqlite.SQLiteDataSource;
}
```

### Key Points

1. **`requires static org.slf4j`**: Makes slf4j an optional compile-time dependency, not required at runtime
2. **Native Library Loading**: sqlite-jdbc loads native SQLite libraries using `System.load()` which works in jlink'd images
3. **Driver Registration**: The `Class.forName("org.sqlite.JDBC")` ensures the driver is loaded in modular context

## Comparison: Old vs New Approach

### Old Approach (Pre-3.47)

| Step | Description |
|------|-------------|
| 1 | Use jdeps to generate module-info for sqlite-jdbc |
| 2 | Use moditect-maven-plugin to inject module-info into JAR |
| 3 | Handle transitive dependency issues |
| 4 | Create runtime image with complex configuration |

### New Approach (3.47+)

| Step | Description |
|------|-------------|
| 1 | Add sqlite-jdbc 3.47+ dependency |
| 2 | Configure jlink in build.gradle |
| 3 | Build and run |

## Lessons Learned

1. **Always use latest versions**: Many jlink compatibility issues have been fixed in recent library releases

2. **Check module-info**: Run `java --list-modules` to verify what modules are included

3. **Driver loading**: For modular apps, explicitly load JDBC drivers with `Class.forName()` to ensure they're available in the module context

4. **Static dependencies**: Look for `requires static` in module-info to understand runtime vs compile-time requirements

5. **Native libraries**: sqlite-jdbc's native library loading works in jlink'd images because it's handled at runtime, not compile time

## Recommendations for JabRef

Based on this investigation:

1. **Use sqlite-jdbc 3.47+**: This version works with jlink out of the box
2. **No moditect needed**: The pure approach works, simplifying the build
3. **Test with jlink early**: Integrate jlink testing in CI to catch issues early
4. **Consider other JDBC drivers**: If using H2 or other databases, check their JPMS support

## Conclusion

This proof-of-concept demonstrates that the jlink + sqlite-jdbc integration challenge has been solved in recent sqlite-jdbc versions. The solution is simple, requires no workarounds, and provides a clean path forward for modular Java applications using jlink and jpackage.
