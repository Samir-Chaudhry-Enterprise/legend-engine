# Java 17 Migration Strategy for Legend Engine Monorepo

## Executive Summary

This document provides a comprehensive dependency analysis and migration strategy for upgrading the Legend Engine monorepo from Java 8 to Java 17. The analysis identifies the module dependency hierarchy, framework compatibility requirements, and a phased migration approach.

**Current State:**
- Target: Java 8 (`maven.compiler.source=1.8`, `maven.compiler.target=1.8`, `maven.compiler.release=8`)
- Build Requirement: Java 11 (`maven.enforcer.requireJavaVersion=[11.0.10,12)`)
- CI/CD: Java 11 (Zulu distribution)
- Total Modules: ~575 pom.xml files across 85+ top-level modules

---

## Migration Progress Tracking

| Phase | Status | Modules | PR |
|-------|--------|---------|-----|
| Phase 0: Preparation | COMPLETED | CI/CD updated to Java 17, Maven enforcer updated | [PR #4](https://github.com/Samir-Chaudhry-Enterprise/legend-engine/pull/4) |
| Phase 1: Foundation Layer | COMPLETED | legend-engine-shared-structures, legend-engine-protocol, legend-engine-shared-extensions | [PR #4](https://github.com/Samir-Chaudhry-Enterprise/legend-engine/pull/4) |
| Phase 2: Core Shared Layer | IN PROGRESS | legend-engine-protocol-pure, legend-engine-shared-core, legend-engine-shared-javaCompiler, legend-engine-shared-vault (all sub-modules) | Current PR |
| Phase 3: Language & Compilation | PENDING | | |
| Phase 4: Execution Layer | PENDING | | |
| Phase 5: Pure Runtime | PENDING | | |
| Phase 6: External Formats | PENDING | | |
| Phase 7: Store Extensions | PENDING | | |
| Phase 8: Function Activators | PENDING | | |
| Phase 9: Server & Integration | PENDING | | |
| Phase 10: Final Validation | PENDING | | |

---

## 1. Complete Dependency Graph

### 1.1 Module Hierarchy Overview

```
legend-engine (root)
├── legend-engine-core/                          [CORE SYSTEM - Migrate First]
│   ├── legend-engine-core-shared/               [FOUNDATION LAYER]
│   │   ├── legend-engine-shared-structures      [LEAF - No internal deps]
│   │   ├── legend-engine-shared-extensions      [Depends on: shared-structures]
│   │   ├── legend-engine-shared-core            [Depends on: shared-extensions, protocol, protocol-pure, identity-core]
│   │   ├── legend-engine-shared-javaCompiler    [Depends on: shared-core]
│   │   └── legend-engine-shared-vault/          [Depends on: shared-core]
│   │
│   ├── legend-engine-core-identity/             [ALREADY JAVA 11]
│   │   └── legend-engine-identity-core          [ALREADY JAVA 11]
│   │
│   ├── legend-engine-core-base/                 [LANGUAGE & EXECUTION LAYER]
│   │   ├── legend-engine-core-language-pure/
│   │   │   ├── legend-engine-protocol           [LEAF - Only Jackson deps]
│   │   │   ├── legend-engine-protocol-pure      [Depends on: protocol]
│   │   │   ├── legend-engine-language-pure-grammar    [Depends on: shared-extensions, protocol, protocol-pure, shared-core]
│   │   │   ├── legend-engine-language-pure-compiler   [Depends on: grammar, shared-core, pure-code-compiled-core]
│   │   │   └── legend-engine-language-pure-modelManager [Depends on: compiler, grammar]
│   │   │
│   │   ├── legend-engine-core-executionPlan-execution/
│   │   │   ├── legend-engine-executionPlan-dependencies [Depends on: protocol-pure]
│   │   │   ├── legend-engine-executionPlan-execution    [Depends on: shared-core, dependencies, javaCompiler]
│   │   │   └── legend-engine-executionPlan-execution-store-inMemory [Depends on: execution]
│   │   │
│   │   └── legend-engine-core-executionPlan-generation/
│   │       └── legend-engine-executionPlan-generation   [Depends on: compiler, execution]
│   │
│   ├── legend-engine-core-pure/                 [PURE RUNTIME LAYER]
│   │   ├── legend-engine-pure-code-compiled-core [Depends on: legend-pure]
│   │   ├── legend-engine-pure-platform-modular-generation/
│   │   │   └── legend-engine-pure-platform-java [Depends on: legend-pure]
│   │   └── legend-engine-pure-ide/
│   │       └── legend-engine-pure-ide-light-http-server [Depends on: server-http-server]
│   │
│   ├── legend-engine-core-external-format/      [EXTERNAL FORMAT LAYER]
│   │   ├── legend-engine-external-format-core   [Depends on: compiler, grammar]
│   │   └── legend-engine-external-shared-format-runtime [Depends on: execution]
│   │
│   └── legend-engine-core-testable/
│       ├── legend-engine-testable              [Depends on: compiler]
│       └── legend-engine-test-mft              [ALREADY JAVA 11]
│
├── legend-engine-xts-relationalStore/           [RELATIONAL STORE EXTENSIONS]
│   ├── legend-engine-xt-relationalStore-generation/
│   │   ├── legend-engine-xt-relationalStore-pure
│   │   └── legend-engine-xt-relationalStore-grammar
│   ├── legend-engine-xt-relationalStore-execution/
│   │   └── legend-engine-xt-relationalStore-executionPlan
│   ├── legend-engine-xt-relationalStore-dbExtension/
│   │   ├── legend-engine-xt-relationalStore-h2/
│   │   │   └── legend-engine-xt-relationalStore-h2-execution-2.1.214 [ALREADY JAVA 11]
│   │   ├── legend-engine-xt-relationalStore-postgres/
│   │   ├── legend-engine-xt-relationalStore-snowflake/
│   │   ├── legend-engine-xt-relationalStore-databricks/
│   │   ├── legend-engine-xt-relationalStore-bigquery/
│   │   └── [14+ more database adapters]
│   └── legend-engine-xt-relationalStore-PCT/    [PCT Testing Framework]
│
├── legend-engine-xts-serviceStore/              [SERVICE STORE EXTENSIONS]
│   ├── legend-engine-xt-serviceStore-pure
│   ├── legend-engine-xt-serviceStore-grammar
│   ├── legend-engine-xt-serviceStore-protocol
│   └── legend-engine-xt-serviceStore-executionPlan
│
├── legend-engine-xts-mongodb/                   [MONGODB EXTENSIONS]
├── legend-engine-xts-elasticsearch/             [ELASTICSEARCH EXTENSIONS]
├── legend-engine-xts-deephaven/                 [DEEPHAVEN - PARTIALLY JAVA 11]
│   └── legend-engine-xt-deephaven-executionPlan [ALREADY JAVA 11]
│
├── legend-engine-xts-json/                      [JSON FORMAT]
├── legend-engine-xts-xml/                       [XML FORMAT]
├── legend-engine-xts-protobuf/                  [PROTOBUF FORMAT]
├── legend-engine-xts-flatdata/                  [FLATDATA FORMAT]
├── legend-engine-xts-avro/                      [AVRO FORMAT]
│
├── legend-engine-xts-functionActivator/         [FUNCTION ACTIVATORS]
├── legend-engine-xts-snowflake/                 [SNOWFLAKE FUNCTIONS]
│   └── legend-engine-xt-snowflake-pure-test     [ALREADY JAVA 11]
├── legend-engine-xts-bigqueryFunction/          [BIGQUERY FUNCTIONS]
├── legend-engine-xts-memsqlFunction/            [MEMSQL FUNCTIONS]
│
├── legend-engine-xts-dataquality/               [DATA QUALITY - ALL JAVA 11]
│   ├── legend-engine-xt-dataquality-api         [ALREADY JAVA 11]
│   ├── legend-engine-xt-dataquality-compiler    [ALREADY JAVA 11]
│   ├── legend-engine-xt-dataquality-generation  [ALREADY JAVA 11]
│   ├── legend-engine-xt-dataquality-grammar     [ALREADY JAVA 11]
│   ├── legend-engine-xt-dataquality-protocol    [ALREADY JAVA 11]
│   ├── legend-engine-xt-dataquality-pure        [ALREADY JAVA 11]
│   └── legend-engine-xt-dataquality-pure-test   [ALREADY JAVA 11]
│
├── legend-engine-xts-identity/                  [IDENTITY - ALL JAVA 11]
│   ├── legend-engine-xt-identity-pac4j          [ALREADY JAVA 11]
│   ├── legend-engine-xt-identity-kerberos       [ALREADY JAVA 11]
│   ├── legend-engine-xt-identity-oauth          [ALREADY JAVA 11]
│   ├── legend-engine-xt-identity-gcp            [ALREADY JAVA 11]
│   ├── legend-engine-xt-identity-apiToken       [ALREADY JAVA 11]
│   ├── legend-engine-xt-identity-privateKey     [ALREADY JAVA 11]
│   ├── legend-engine-xt-identity-middletier     [ALREADY JAVA 11]
│   └── legend-engine-xt-identity-plainTextUserPassword [ALREADY JAVA 11]
│
├── legend-engine-xts-analytics/                 [ANALYTICS]
│   ├── legend-engine-xts-analytics-lineage/
│   ├── legend-engine-xts-analytics-mapping/
│   ├── legend-engine-xts-analytics-search/
│   └── legend-engine-xts-analytics-quality/     [ALREADY JAVA 11]
│
├── legend-engine-xts-generation/
│   └── legend-engine-xts-deployment-model       [ALREADY JAVA 11]
│
└── legend-engine-config/                        [SERVER CONFIGURATION]
    ├── legend-engine-server/
    │   ├── legend-engine-server-support-core    [ALREADY JAVA 11]
    │   ├── legend-engine-server-http-server     [Depends on: ALL extensions]
    │   └── legend-engine-server-integration-tests
    └── legend-engine-repl/
```

### 1.2 Leaf Modules (No Internal Dependencies)

These modules should be migrated first as they have no dependencies on other Legend Engine modules:

| Module | Current Java | Dependencies |
|--------|-------------|--------------|
| `legend-engine-shared-structures` | Java 8 | Eclipse Collections only |
| `legend-engine-protocol` | Java 8 | Jackson only |
| `legend-engine-identity-core` | **Java 11** | External libs only |

---

## 2. Modules Already Targeting Java 11

The following 27+ modules already target Java 11 and serve as a foundation for the migration:

### Core Identity (All Java 11)
- `legend-engine-core-identity`
- `legend-engine-identity-core`

### Identity Extensions (All Java 11)
- `legend-engine-xts-identity` (parent)
- `legend-engine-xt-identity-pac4j`
- `legend-engine-xt-identity-kerberos`
- `legend-engine-xt-identity-oauth`
- `legend-engine-xt-identity-gcp`
- `legend-engine-xt-identity-apiToken`
- `legend-engine-xt-identity-privateKey`
- `legend-engine-xt-identity-middletier`
- `legend-engine-xt-identity-plainTextUserPassword`

### Data Quality (All Java 11)
- `legend-engine-xts-dataquality` (parent)
- `legend-engine-xt-dataquality-api`
- `legend-engine-xt-dataquality-compiler`
- `legend-engine-xt-dataquality-generation`
- `legend-engine-xt-dataquality-grammar`
- `legend-engine-xt-dataquality-protocol`
- `legend-engine-xt-dataquality-pure`
- `legend-engine-xt-dataquality-pure-test`

### Server & Testing
- `legend-engine-server-support-core`
- `legend-engine-test-mft`

### Database Extensions
- `legend-engine-xt-relationalStore-h2-execution-2.1.214`

### Analytics
- `legend-engine-xts-analytics-quality`
- `legend-engine-xt-analytics-quality-pure`

### Other
- `legend-engine-xts-deployment-model`
- `legend-engine-xt-deephaven-executionPlan`
- `legend-engine-xt-snowflake-pure-test`

---

## 3. Framework Compatibility Matrix

### 3.1 Current Versions vs Java 17 Requirements

| Framework | Current Version | Java 17 Compatible | Recommended Version | Notes |
|-----------|----------------|-------------------|---------------------|-------|
| **Jackson** | 2.10.5 | Partial | 2.15.x+ | 2.10.x works but has limitations; upgrade recommended |
| **Eclipse Collections** | 10.2.0 | Yes | 11.1.x | Current version works; upgrade for better performance |
| **DropWizard** | 1.3.29 | **No** | 2.1.x+ | **CRITICAL**: Must upgrade; 1.x doesn't support Java 17 |
| **Jetty** | 9.4.44 | Partial | 11.x or 12.x | 9.4.x has limited Java 17 support; upgrade recommended |
| **Jersey** | 2.25.1 | **No** | 2.39+ or 3.x | **CRITICAL**: Must upgrade for Java 17 |
| **Guava** | 30.0-jre | Yes | 32.x+ | Current version works; upgrade recommended |
| **SLF4J** | 1.7.36 | Yes | 2.0.x | Current version works; upgrade for better features |
| **Logback** | 1.2.3 | Partial | 1.4.x+ | Upgrade recommended for full Java 17 support |
| **ANTLR** | 4.8-1 | Yes | 4.13.x | Current version works; upgrade recommended |
| **Pac4j** | 4.5.8 | Partial | 5.x+ | Upgrade recommended for better Java 17 support |
| **Netty** | 4.1.100 | Yes | 4.1.100+ | Current version is compatible |
| **gRPC** | 1.56.0 | Yes | 1.56.0+ | Current version is compatible |
| **Arrow** | 13.0.0 | Yes | 13.0.0+ | Current version is compatible |
| **Protobuf** | 3.25.3 | Yes | 3.25.x | Current version is compatible |

### 3.2 Database Driver Compatibility

| Driver | Current Version | Java 17 Compatible | Notes |
|--------|----------------|-------------------|-------|
| H2 | 2.1.214 | Yes | Compatible |
| PostgreSQL | 42.7.4 | Yes | Compatible |
| Snowflake | 3.13.5 | Yes | Compatible |
| Databricks | 2.6.27 | Yes | Compatible |
| DuckDB | 1.3.0.0 | Yes | Compatible |
| MariaDB | 3.0.6 | Yes | Compatible |
| MongoDB | 5.3.1 | Yes | Compatible |

### 3.3 Critical Upgrades Required

**Must Upgrade Before Java 17:**

1. **DropWizard 1.3.29 → 2.1.x+**
   - DropWizard 1.x uses Jetty 9.x and Jersey 2.25.x which have Java 17 issues
   - DropWizard 2.x uses Jetty 10/11 and Jersey 2.35+ with full Java 17 support
   - This is a **breaking change** requiring code modifications

2. **Jersey 2.25.1 → 2.39+**
   - Jersey 2.25.x has reflection issues with Java 17
   - Upgrade to 2.39+ or consider Jakarta EE 9+ (Jersey 3.x)

3. **Jetty 9.4.44 → 11.x or 12.x**
   - Jetty 9.4.x has limited Java 17 support
   - Jetty 11.x is the recommended upgrade path (Jakarta EE 9)
   - Jetty 12.x is the latest (Jakarta EE 10)

4. **Jackson 2.10.5 → 2.15.x+**
   - While 2.10.x technically works, upgrading provides better Java 17 support
   - Includes fixes for module system and reflection access

---

## 4. Phased Migration Plan

### Phase 0: Preparation (Pre-Migration)

**Duration:** 2-3 weeks

**Tasks:**
1. Update CI/CD to use Java 17 for testing (parallel to Java 11)
2. Add Java 17 compatibility tests to the build matrix
3. Create feature branch for migration work
4. Document all deprecated API usage that will break in Java 17

**Files to Update:**
- `.github/workflows/actions/before-test/action.yml` (line 38: change `java-version: 11` to `java-version: 17`)
- Root `pom.xml` (lines 140-143: update compiler properties)

### Phase 1: Foundation Layer (Leaf Modules)

**Duration:** 1-2 weeks

**Modules to Migrate:**
1. `legend-engine-shared-structures` (true leaf, no internal deps)
2. `legend-engine-protocol` (true leaf, only Jackson deps)
3. `legend-engine-shared-extensions` (depends on structures)

**Changes Required:**
```xml
<!-- In each module's pom.xml -->
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <maven.compiler.release>17</maven.compiler.release>
</properties>
```

**Validation:**
- Run unit tests
- Verify no reflection warnings
- Check for deprecated API usage

### Phase 2: Core Shared Layer

**Duration:** 2-3 weeks

**Modules to Migrate:**
1. `legend-engine-protocol-pure`
2. `legend-engine-shared-core`
3. `legend-engine-shared-javaCompiler`
4. `legend-engine-shared-vault` (all sub-modules)

**Dependencies to Verify:**
- Jackson serialization/deserialization
- Reflection-based code
- Dynamic class loading

### Phase 3: Language & Compilation Layer

**Duration:** 3-4 weeks

**Modules to Migrate:**
1. `legend-engine-language-pure-grammar`
2. `legend-engine-language-pure-compiler`
3. `legend-engine-language-pure-compiler-http-api`
4. `legend-engine-language-pure-modelManager`
5. `legend-engine-language-pure-modelManager-sdlc`

**Key Considerations:**
- ANTLR-generated code compatibility
- Pure language compilation
- Metadata handling

### Phase 4: Execution Layer

**Duration:** 3-4 weeks

**Modules to Migrate:**
1. `legend-engine-executionPlan-dependencies`
2. `legend-engine-executionPlan-execution`
3. `legend-engine-executionPlan-execution-store-inMemory`
4. `legend-engine-executionPlan-execution-http-api`
5. `legend-engine-executionPlan-generation`

**Key Considerations:**
- Dynamic code generation
- Java compiler integration
- Template processing (Freemarker)

### Phase 5: Pure Runtime Layer

**Duration:** 2-3 weeks

**Modules to Migrate:**
1. `legend-engine-pure-code-compiled-core`
2. `legend-engine-pure-platform-java`
3. All `legend-engine-pure-platform-dsl-*` modules
4. All `legend-engine-pure-code-functions-*` modules

**Key Considerations:**
- Generated Pure code compatibility
- Runtime class loading
- Reflection-based Pure execution

### Phase 6: External Format Layer

**Duration:** 2-3 weeks

**Modules to Migrate:**
1. `legend-engine-external-format-core`
2. `legend-engine-external-shared-format-runtime`
3. `legend-engine-xts-json` (all sub-modules)
4. `legend-engine-xts-xml` (all sub-modules)
5. `legend-engine-xts-flatdata` (all sub-modules)
6. `legend-engine-xts-protobuf` (all sub-modules)
7. `legend-engine-xts-avro` (all sub-modules)

### Phase 7: Store Extensions

**Duration:** 4-6 weeks

**Modules to Migrate:**
1. **Relational Store** (largest extension)
   - `legend-engine-xt-relationalStore-pure`
   - `legend-engine-xt-relationalStore-grammar`
   - `legend-engine-xt-relationalStore-executionPlan`
   - All database-specific modules (H2, PostgreSQL, Snowflake, etc.)
   - PCT testing framework

2. **Service Store**
   - All `legend-engine-xt-serviceStore-*` modules

3. **MongoDB**
   - All `legend-engine-xt-nonrelationalStore-mongodb-*` modules

4. **Elasticsearch**
   - All `legend-engine-xt-elasticsearch-*` modules

### Phase 8: Function Activators & Analytics

**Duration:** 2-3 weeks

**Modules to Migrate:**
1. `legend-engine-xts-functionActivator` (all sub-modules)
2. `legend-engine-xts-snowflake` (remaining modules)
3. `legend-engine-xts-bigqueryFunction` (all sub-modules)
4. `legend-engine-xts-memsqlFunction` (all sub-modules)
5. `legend-engine-xts-analytics` (remaining modules)

### Phase 9: Server & Integration

**Duration:** 3-4 weeks

**Critical Framework Upgrades Required First:**
- DropWizard 1.3.29 → 2.1.x
- Jersey 2.25.1 → 2.39+
- Jetty 9.4.44 → 11.x

**Modules to Migrate:**
1. `legend-engine-server-http-server`
2. `legend-engine-server-integration-tests`
3. `legend-engine-pure-ide-light-http-server`
4. `legend-engine-repl`

### Phase 10: Final Validation & Cleanup

**Duration:** 2-3 weeks

**Tasks:**
1. Update root `pom.xml` to set Java 17 as default
2. Update CI/CD to require Java 17
3. Run full test suite
4. Run PCT tests against all databases
5. Performance benchmarking
6. Documentation updates

---

## 5. Potential Blocking Issues

### 5.1 Circular Dependencies

No circular dependencies were identified in the module structure. The dependency graph is acyclic.

### 5.2 Known Issues

1. **Reflection Access Warnings**
   - Java 17 enforces strong encapsulation
   - May need `--add-opens` JVM arguments for some libraries
   - Example: `--add-opens java.base/java.lang=ALL-UNNAMED`

2. **Removed APIs**
   - `javax.xml.bind` (JAXB) removed in Java 11+
   - `javax.annotation` removed in Java 11+
   - Need to add explicit dependencies if used

3. **DropWizard Migration**
   - DropWizard 2.x has breaking API changes
   - Configuration class changes
   - Jersey injection changes

4. **Legend Pure Dependency**
   - Legend Engine depends on `legend-pure` (version 5.72.1)
   - Must verify `legend-pure` Java 17 compatibility
   - May need coordinated upgrade

### 5.3 Risk Mitigation

1. **Incremental Migration**
   - Migrate modules one at a time
   - Maintain backward compatibility during transition
   - Use multi-release JARs if needed

2. **Testing Strategy**
   - Run existing tests after each phase
   - Add Java 17-specific tests
   - Use PCT framework for database compatibility

3. **Rollback Plan**
   - Keep Java 8/11 compatibility branches
   - Document all changes for potential rollback
   - Maintain CI/CD for both versions during transition

---

## 6. Recommended Timeline

| Phase | Duration | Cumulative |
|-------|----------|------------|
| Phase 0: Preparation | 2-3 weeks | 2-3 weeks |
| Phase 1: Foundation | 1-2 weeks | 3-5 weeks |
| Phase 2: Core Shared | 2-3 weeks | 5-8 weeks |
| Phase 3: Language & Compilation | 3-4 weeks | 8-12 weeks |
| Phase 4: Execution | 3-4 weeks | 11-16 weeks |
| Phase 5: Pure Runtime | 2-3 weeks | 13-19 weeks |
| Phase 6: External Formats | 2-3 weeks | 15-22 weeks |
| Phase 7: Store Extensions | 4-6 weeks | 19-28 weeks |
| Phase 8: Function Activators | 2-3 weeks | 21-31 weeks |
| Phase 9: Server & Integration | 3-4 weeks | 24-35 weeks |
| Phase 10: Final Validation | 2-3 weeks | 26-38 weeks |

**Total Estimated Duration: 6-9 months**

---

## 7. Quick Reference: Module Migration Order

### Priority 1 (Migrate First - No Internal Dependencies)
1. `legend-engine-shared-structures`
2. `legend-engine-protocol`

### Priority 2 (Core Infrastructure)
3. `legend-engine-shared-extensions`
4. `legend-engine-protocol-pure`
5. `legend-engine-shared-core`
6. `legend-engine-shared-javaCompiler`

### Priority 3 (Language Layer)
7. `legend-engine-language-pure-grammar`
8. `legend-engine-language-pure-compiler`
9. `legend-engine-language-pure-modelManager`

### Priority 4 (Execution Layer)
10. `legend-engine-executionPlan-dependencies`
11. `legend-engine-executionPlan-execution`
12. `legend-engine-executionPlan-generation`

### Priority 5 (Pure Runtime)
13. `legend-engine-pure-code-compiled-core`
14. `legend-engine-pure-platform-java`

### Priority 6 (External Formats)
15. `legend-engine-external-format-core`
16. `legend-engine-xts-json`
17. `legend-engine-xts-xml`
18. `legend-engine-xts-flatdata`

### Priority 7 (Store Extensions)
19. `legend-engine-xts-relationalStore`
20. `legend-engine-xts-serviceStore`
21. `legend-engine-xts-mongodb`
22. `legend-engine-xts-elasticsearch`

### Priority 8 (Function Activators)
23. `legend-engine-xts-functionActivator`
24. `legend-engine-xts-snowflake`
25. `legend-engine-xts-bigqueryFunction`

### Priority 9 (Server - Requires Framework Upgrades)
26. `legend-engine-server-http-server`
27. `legend-engine-pure-ide-light-http-server`

---

## 8. Appendix: Build Configuration Changes

### Root pom.xml Changes

```xml
<!-- Current (lines 140-143) -->
<maven.compiler.source>1.8</maven.compiler.source>
<maven.compiler.target>1.8</maven.compiler.target>
<maven.compiler.release>8</maven.compiler.release>
<maven.enforcer.requireJavaVersion>[11.0.10,12)</maven.enforcer.requireJavaVersion>

<!-- Target -->
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>
<maven.compiler.release>17</maven.compiler.release>
<maven.enforcer.requireJavaVersion>[17,18)</maven.enforcer.requireJavaVersion>
```

### CI/CD Changes

```yaml
# .github/workflows/actions/before-test/action.yml (line 38)
# Current
java-version: 11

# Target
java-version: 17
```

### JVM Arguments for Reflection Access

```xml
<!-- Add to surefire plugin configuration if needed -->
<argLine>
    --add-opens java.base/java.lang=ALL-UNNAMED
    --add-opens java.base/java.util=ALL-UNNAMED
    --add-opens java.base/java.lang.reflect=ALL-UNNAMED
</argLine>
```

---

*Document generated: January 2026*
*Legend Engine Version: 4.118.2-SNAPSHOT*
