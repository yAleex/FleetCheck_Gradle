# FleetCheck – Build Systems Lab

This project is intentionally incomplete. Follow the worksheet in the order given.

Expected final application output:

FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km

Do not copy the solution POM. The objective is to observe how each build change alters the result.

---

## Evidence

### Evidence 1 – Missing dependency (Maven)

Command: `mvn clean package` (JDK 21, Maven 3.9.12)

[ERROR] /home/alex/IdeaProjects/FleetCheck_Starter/src/main/java/pt/upt/fleetcheck/App.java:[4,38] package com.fasterxml.jackson.databind does not exist

Cause: the import `com.fasterxml.jackson.databind.ObjectMapper` (line 4 of App.java) fails because pom.xml does not declare the jackson-databind dependency.

### Evidence 4 – What the Shade plugin changed

Before (default JAR): `java -jar target/fleetcheck-1.0.0.jar` fails with
"no main manifest attribute". The default JAR contains only the project's own
classes and a manifest without Main-Class; Jackson is not included.

After (Shade): `target/fleetcheck-1.0.0-all.jar` is a fat JAR. Shade writes
Main-Class: pt.upt.fleetcheck.App into the manifest and copies the classes of
jackson-databind, jackson-core and jackson-annotations into the JAR, so it
runs standalone with `java -jar`. The normal JAR is kept alongside it.

### Evidence 8.1 – Missing dependency (Gradle)

Command: `gradle clean build` (Gradle 9.8.0, JDK 21)

/home/alex/IdeaProjects/FleetCheck_Gradle/src/main/java/pt/upt/fleetcheck/App.java:4: error: package com.fasterxml.jackson.databind does not exist
import com.fasterxml.jackson.databind.ObjectMapper;

Cause: the import `com.fasterxml.jackson.databind.ObjectMapper` (line 4 of
App.java) fails because build.gradle does not declare the jackson-databind
dependency yet — same missing dependency as in the Maven build (Evidence 1),
just reported by a different build tool.

### Evidence 8.2 – Dependency graph (Gradle) vs Maven

`gradle dependencies --configuration runtimeClasspath`:

jackson-databind:2.22.2 (direct)
├── jackson-annotations:2.22 (transitive)
└── jackson-core:2.22.2 (transitive)

This matches `mvn dependency:tree` (Step 3) exactly: jackson-databind is the
only direct dependency in both builds, and jackson-core / jackson-annotations
are pulled in transitively, with the same versions. The jackson-bom entries
in the Gradle output are just version constraints from Jackson's own BOM,
not extra dependencies.

Changing the build system (Maven → Gradle) did not change the application's
dependencies — both resolve the same artifacts and versions. What changed
is only how each tool declares dependencies and reports the graph.

### Evidence 8.3 – What changed in the JAR after including runtime dependencies (Gradle)

Before: `java -jar build/libs/fleetcheck-1.0.0.jar` failed with
"no main manifest attribute". The default Gradle jar contained only the
project's own classes, with no Main-Class entry in the manifest and no
Jackson classes.

After: adding the `application` plugin with `mainClass` and configuring the
`jar` task to (1) write `Main-Class: pt.upt.fleetcheck.App` into the manifest
and (2) merge the contents of `runtimeClasspath` (jackson-databind,
jackson-core, jackson-annotations) into the JAR turned it into a fat/uber JAR.
It now runs standalone with `java -jar`, producing:

FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 1
Average mileage: 37000 km

This is the same behaviour observed with the Maven Shade-built JAR
(fleetcheck-1.0.0-all.jar): both report 1 vehicle requiring service, not 2
as stated in the worksheet's "Expected application output". Since the exact
same src/ was used for both builds, this confirms the discrepancy comes from
the application logic (FleetService), not from Maven or Gradle.