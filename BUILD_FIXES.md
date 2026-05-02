# Modifications nécessaires pour compiler RINexus

> **Auteure du document** : Emeric Laberge
> **Date** : 2 mai 2026
> **Repo original** : [`major-lab/RINexus`](https://github.com/major-lab/RINexus) (François Major, IRIC, Université de Montréal)
> **Fork fonctionnel** : [`EmericLaberge/RINexus`](https://github.com/EmericLaberge/RINexus)

---

## Contexte

Le repo `major-lab/RINexus` contient le code source Java pour RIMap-RISC et RINets, des outils de prédiction d'interactions RNA. Cependant, **le repo ne compile pas tel quel** après un `git clone`. Ce document détaille chaque problème rencontré et la correction appliquée.

---

## État initial du repo (après `git clone`)

```
RINexus/
├── .gitignore
├── LICENSE
├── README.md          # Contient seulement "# RINexus\nRepository for RIMaps and RINets"
└── src/
    └── main/java/
        └── ca/iric/major/
            ├── common/         # 70+ fichiers Java (coeur du projet)
            └── rinexus/rimaprisc/  # 15 fichiers Java (API web + scan)
```

- **85 fichiers `.java`**
- **Aucun `pom.xml`** à la racine
- **Aucun système de build** fonctionnel

---

## Erreur #1 — Aucun fichier de build (`pom.xml` absent)

### Symptôme

```
$ mvn compile
[ERROR] The goal you specified requires a project to execute but there
is no POM in this directory. Please verify you invoked Maven from the
correct directory.
```

### Cause

Le repo original ne contient aucun `pom.xml`. L'historique Git montre qu'un `common/pom.xml` existait dans un commit antérieur, mais il a été supprimé lors de la réorganisation du repo. Ce POM original référençait un parent Maven inaccessible :

```xml
<parent>
    <groupId>ca.iric.major</groupId>
    <artifactId>projects</artifactId>
    <version>1.0</version>
</parent>
```

Le POM parent `ca.iric.major:projects:1.0` est un POM local de François Major qui n'est pas publié sur Maven Central ni sur aucun dépôt public. Il est donc **impossible de compiler le projet avec le POM original**, même si on le récupère depuis l'historique Git.

### Correction

Créer un `pom.xml` autonome, sans parent inaccessible. Le POM original listait déjà certaines dépendances (`jackson-databind`, `commons-csv`, `commons-io`, `commons-codec`), mais le code Java utilise aussi **Spring Boot** et **Lombok** (non listés dans le POM original).

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
        <!-- Dépendances du POM original -->
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
        <!-- Spring Boot (nécessaire pour rimaprisc/) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- Lombok (nécessaire pour RISCScanRequest.java) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.30</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

**Changements clés par rapport au POM original :**

| Élément | POM original (inaccessible) | Nouveau POM |
|---------|---------------------------|-------------|
| Parent | `ca.iric.major:projects:1.0` (introuvable) | `spring-boot-starter-parent:3.2.0` |
| `artifactId` | `common` (sous-module) | `RINexus` (projet standalone) |
| `packaging` | non spécifié | `jar` |
| Spring Boot | absent | `spring-boot-starter-web` |
| Lombok | absent | `1.18.30` (provided) |
| `<build>` | absent | hérité de Spring Boot parent |

---

## Erreur #2 — Dépendances Spring Boot manquantes

### Symptôme

Même après création d'un `pom.xml` minimal (sans Spring Boot), la compilation échoue avec **~40 erreurs** :

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

### Cause

Les fichiers suivants dans `rimaprisc/` utilisent Spring Boot mais cette dépendance n'était pas dans le POM original :

| Fichier | Annotations Spring utilisées |
|---------|------------------------------|
| `RIMapRISC.java` | `@SpringBootApplication` |
| `RISCScanController.java` | `@RestController`, `@RequestMapping`, `@PostMapping`, `@RequestBody`, `@CrossOrigin`, `ResponseEntity` |
| `TranscriptController.java` | `@RestController`, `@GetMapping` |
| `WebConfig.java` | `@Configuration`, `@Bean`, `WebMvcConfigurer`, `CorsRegistry` |

### Correction

Ajout de `spring-boot-starter-web` comme dépendance dans le `pom.xml` (voir correction #1).

---

## Erreur #3 — Lombok manquant

### Symptôme

```
[ERROR] package lombok does not exist
[ERROR] cannot find symbol: class Getter
[ERROR] cannot find symbol: class Setter
[ERROR] cannot find symbol: class NoArgsConstructor
```

### Cause

`RISCScanRequest.java` utilise les annotations Lombok (`@Getter`, `@Setter`, `@NoArgsConstructor`) pour générer automatiquement les getters/setters/constructeur, mais Lombok n'est pas listé comme dépendance.

### Correction

Ajout de Lombok en scope `provided` dans le `pom.xml` :

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>
```

---

## Erreur #4 — Classes `ca.iric.major.tools` manquantes

### Symptôme

```
[ERROR] package ca.iric.major.tools does not exist
[ERROR] cannot find symbol: variable SeedDuplexCache
[ERROR] cannot find symbol: variable SeedToEightMerCache
```

### Cause

Deux classes sont référencées par le code existant mais **absentes du repo GitHub** :

- `EightMerDuplex.java` importe `ca.iric.major.tools.SeedDuplexCache`
- `SeedDuplex.java` importe `ca.iric.major.tools.SeedToEightMerCache`

Le package `ca.iric.major.tools` n'existe pas dans le repo.

### Correction

Créer le répertoire et les deux classes :

```
src/main/java/ca/iric/major/tools/
├── SeedDuplexCache.java
└── SeedToEightMerCache.java
```

**`SeedToEightMerCache.java`** — Encode/décode des 7-mers RNA en binaire compact (2 bits/base) :

```java
package ca.iric.major.tools;

public class SeedToEightMerCache {
    // A=0, C=1, G=2, U=3 — 7mer = 14 bits, tient dans un int
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

**`SeedDuplexCache.java`** — Décode des 8-mers :

```java
package ca.iric.major.tools;

public class SeedDuplexCache {
    // A=0, C=1, G=2, U=3 — 8mer = 16 bits, tient dans un int
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

## Résumé des modifications

| # | Problème | Fichiers concernés | Action |
|---|----------|--------------------|--------|
| 1 | Aucun `pom.xml` / parent POM inaccessible | — | Créer un `pom.xml` standalone avec `spring-boot-starter-parent` |
| 2 | Spring Boot non résolu | `RIMapRISC`, `RISCScanController`, `TranscriptController`, `WebConfig` | Ajouter `spring-boot-starter-web` |
| 3 | Lombok non résolu | `RISCScanRequest` | Ajouter `lombok` en scope `provided` |
| 4 | Package `tools` manquant | `EightMerDuplex`, `SeedDuplex` | Créer `SeedToEightMerCache.java` et `SeedDuplexCache.java` |

**Total : 1 fichier modifié (`pom.xml` créé) + 2 fichiers créés. Aucun fichier `.java` existant n'a été modifié.**

---

## Vérification

Après application des corrections :

```bash
$ mvn compile
[INFO] BUILD SUCCESS
[INFO] Compiling 87 source files

$ mvn exec:java -Dexec.mainClass="ca.iric.major.common.CommonKmers"
# → Exécute correctement, affiche les k-mers communs
```

---

## Fork fonctionnel

Le repo forké avec toutes les corrections appliquées est disponible ici :
**https://github.com/EmericLaberge/RINexus**
