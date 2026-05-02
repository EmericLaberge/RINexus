# Modifications Required to Compile RINexus

> **Author** : Emeric Laberge
> **Date** : May 2, 2026
> **Original repo** : [`major-lab/RINexus`](https://github.com/major-lab/RINexus) (François Major, IRIC, Université de Montréal)
> **Working fork** : [`EmericLaberge/RINexus`](https://github.com/EmericLaberge/RINexus)

---

## Context

The `major-lab/RINexus` repository contains the Java source code for RIMap-RISC and RINets, RNA interaction prediction tools. However, **the repository does not compile as-is** after a `git clone`. This document details every issue encountered and the fix applied.

---

## Initial Repository State (after `git clone`)

```
RINexus/
├── .gitignore
├── LICENSE
├── README.md          # Only contains "# RINexus\nRepository for RIMaps and RINets"
└── src/
    └── main/java/
        └── ca/iric/major/
            ├── common/             # 70+ Java files (core project)
            └── rinexus/rimaprisc/  # 15 Java files (web API + scan)
```

- **85 `.java` files**
- **No `pom.xml`** at the root
- **No functional build system**

---

## Issue #1 — No Build File (`pom.xml` missing)

### Symptom

```
$ mvn compile
[ERROR] The goal you specified requires a project to execute but there
is no POM in this directory. Please verify you invoked Maven from the
correct directory.
```

### Root Cause

The original repository does not contain any `pom.xml`. The Git history shows that a `common/pom.xml` existed in an earlier commit, but it was deleted during a repository restructuring. This original POM referenced an unreachable Maven parent:

```xml
<parent>
    <groupId>ca.iric.major</groupId>
    <artifactId>projects</artifactId>
    <version>1.0</version>
</parent>
```

The parent POM `ca.iric.major:projects:1.0` is a local POM belonging to François Major that is not published on Maven Central or any public repository. It is therefore **impossible to compile the project with the original POM**, even if recovered from the Git history.

### Fix

Create a standalone `pom.xml` with no unreachable parent. The original POM already listed some dependencies (`jackson-databind`, `commons-csv`, `commons-io`, `commons-codec`), but the Java code also uses **Spring Boot** and **Lombok** (not listed in the original POM).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>ca.iric.major</groupId>
    <artifactId>RINexus</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>RINexus</name>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Dependencies from the original POM -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-csv</artifactId>
            <version>1.9.0</version>
        </dependency>
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.16.1</version>
        </dependency>
        <dependency>
            <groupId>commons-codec</groupId>
            <artifactId>commons-codec</artifactId>
            <version>1.17.1</version>
        </dependency>
        <!-- Spring Boot (required by rimaprisc/) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- Lombok (required by RISCScanRequest.java) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.30</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

**Key changes from the original POM:**

| Element | Original POM (unreachable) | New POM |
|---------|---------------------------|---------|
| Parent | `ca.iric.major:projects:1.0` (unresolvable) | `spring-boot-starter-parent:3.2.0` |
| `artifactId` | `common` (sub-module) | `RINexus` (standalone project) |
| `packaging` | not specified | `jar` |
| Spring Boot | absent | `spring-boot-starter-web` |
| Lombok | absent | `1.18.30` (provided) |
| `<build>` | absent | inherited from Spring Boot parent |

---

## Issue #2 — Missing Spring Boot Dependencies

### Symptom

Even after creating a minimal `pom.xml` (without Spring Boot), compilation fails with **~40 errors**:

```
[ERROR] package org.springframework.boot does not exist
[ERROR] package org.springframework.boot.autoconfigure does not exist
[ERROR] package org.springframework.web.bind.annotation does not exist
[ERROR] package org.springframework.http does not exist
[ERROR] package org.springframework.context.annotation does not exist
[ERROR] package org.springframework.web.servlet.config.annotation does not exist
[ERROR] cannot find symbol: class SpringBootApplication
[ERROR] cannot find symbol: class RestController
[ERROR] cannot find symbol: class RequestMapping
[ERROR] cannot find symbol: class ResponseEntity
[ERROR] cannot find symbol: class WebMvcConfigurer
...
```

### Root Cause

The following files in `rimaprisc/` use Spring Boot, but this dependency was not in the original POM:

| File | Spring annotations used |
|------|-------------------------|
| `RIMapRISC.java` | `@SpringBootApplication` |
| `RISCScanController.java` | `@RestController`, `@RequestMapping`, `@PostMapping`, `@RequestBody`, `@CrossOrigin`, `ResponseEntity` |
| `TranscriptController.java` | `@RestController`, `@GetMapping` |
| `WebConfig.java` | `@Configuration`, `@Bean`, `WebMvcConfigurer`, `CorsRegistry` |

### Fix

Add `spring-boot-starter-web` as a dependency in `pom.xml` (see fix #1).

---

## Issue #3 — Missing Lombok Dependency

### Symptom

```
[ERROR] package lombok does not exist
[ERROR] cannot find symbol: class Getter
[ERROR] cannot find symbol: class Setter
[ERROR] cannot find symbol: class NoArgsConstructor
```

### Root Cause

`RISCScanRequest.java` uses Lombok annotations (`@Getter`, `@Setter`, `@NoArgsConstructor`) to auto-generate getters, setters, and a no-arg constructor, but Lombok is not listed as a dependency.

### Fix

Add Lombok with `provided` scope in `pom.xml`:

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>
```

---

## Issue #4 — Missing `ca.iric.major.tools` Classes

### Symptom

```
[ERROR] package ca.iric.major.tools does not exist
[ERROR] cannot find symbol: variable SeedDuplexCache
[ERROR] cannot find symbol: variable SeedToEightMerCache
```

### Root Cause

Two classes are referenced by existing code but **absent from the GitHub repository**:

- `EightMerDuplex.java` imports `ca.iric.major.tools.SeedDuplexCache`
- `SeedDuplex.java` imports `ca.iric.major.tools.SeedToEightMerCache`

The `ca.iric.major.tools` package does not exist in the repository.

### Fix

Create the directory and both classes:

```
src/main/java/ca/iric/major/tools/
├── SeedDuplexCache.java
└── SeedToEightMerCache.java
```

**`SeedToEightMerCache.java`** — Encodes/decodes RNA 7-mers into compact binary (2 bits/base):

```java
package ca.iric.major.tools;

public class SeedToEightMerCache {
    // A=0, C=1, G=2, U=3 — 7mer = 14 bits, fits in an int
    public static int encode7mer(String seed) {
        int encoded = 0;
        for (char c : seed.toCharArray()) {
            encoded <<= 2;
            switch (c) {
                case 'A': encoded |= 0; break;
                case 'C': encoded |= 1; break;
                case 'G': encoded |= 2; break;
                case 'U': encoded |= 3; break;
            }
        }
        return encoded;
    }

    public static String decode7mer(int encoded) {
        StringBuilder sb = new StringBuilder(7);
        for (int i = 6; i >= 0; i--) {
            int bits = (encoded >> (i * 2)) & 3;
            switch (bits) {
                case 0: sb.append('A'); break;
                case 1: sb.append('C'); break;
                case 2: sb.append('G'); break;
                case 3: sb.append('U'); break;
            }
        }
        return sb.toString();
    }
}
```

**`SeedDuplexCache.java`** — Decodes RNA 8-mers:

```java
package ca.iric.major.tools;

public class SeedDuplexCache {
    // A=0, C=1, G=2, U=3 — 8mer = 16 bits, fits in an int
    public static String decode8mer(int encoded) {
        StringBuilder sb = new StringBuilder(8);
        for (int i = 7; i >= 0; i--) {
            int bits = (encoded >> (i * 2)) & 3;
            switch (bits) {
                case 0: sb.append('A'); break;
                case 1: sb.append('C'); break;
                case 2: sb.append('G'); break;
                case 3: sb.append('U'); break;
            }
        }
        return sb.toString();
    }
}
```

---

## Summary of Changes

| # | Issue | Files Affected | Fix |
|---|-------|----------------|-----|
| 1 | No `pom.xml` / unreachable parent POM | — | Create standalone `pom.xml` with `spring-boot-starter-parent` |
| 2 | Spring Boot unresolved | `RIMapRISC`, `RISCScanController`, `TranscriptController`, `WebConfig` | Add `spring-boot-starter-web` |
| 3 | Lombok unresolved | `RISCScanRequest` | Add `lombok` with `provided` scope |
| 4 | Missing `tools` package | `EightMerDuplex`, `SeedDuplex` | Create `SeedToEightMerCache.java` and `SeedDuplexCache.java` |

**Total: 1 file modified (`pom.xml` created) + 2 files created. No existing `.java` files were modified.**

---

## Verification

After applying all fixes:

```bash
$ mvn compile
[INFO] BUILD SUCCESS
[INFO] Compiling 87 source files

$ mvn exec:java -Dexec.mainClass="ca.iric.major.common.CommonKmers"
# → Runs correctly, displays common k-mers
```

---

## Working Fork

The forked repository with all fixes applied is available at:
**https://github.com/EmericLaberge/RINexus**

---

*Version française : [`BUILD_FIXES.md`](BUILD_FIXES.md)*
