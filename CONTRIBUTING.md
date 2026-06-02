# Contributing to Gosu

Thank you for your interest in contributing to the Gosu programming language! This guide covers everything you need to get started.

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| **Java JDK** | 11 | Set `JAVA_HOME` to point to your JDK 11 installation |
| **Maven** | 3.x | Used for builds, dependency management, and releases |

> **Tip:** Verify your setup with `java -version` (should show 11.x) and `mvn --version`.

## Building

Clone the repository and compile:

```bash
git clone https://github.com/gosu-lang/gosu-lang.git
cd gosu-lang
export JAVA_HOME=/path/to/jdk-11
mvn compile
```

To run the full test suite (~13,900 tests across 12 modules):

```bash
mvn test
```

To run a single test class:

```bash
mvn test -pl gosu-test -Dtest=gw.specification.statements.forEachStatement.ForEachStatementTest
```

To build and verify (compile + test + integration checks):

```bash
mvn clean verify -B -V
```

## `--add-exports` Requirements and IDE Setup

Gosu accesses internal JDK APIs (`com.sun.tools.javac.*`, `sun.reflect.annotation`, `sun.awt`, etc.). The Maven POM (`gosu-parent/pom.xml`) already configures the required `--add-exports` flags for the compiler and surefire plugins. However, your IDE may need manual configuration:

### IntelliJ IDEA

1. Go to **File → Settings → Build, Execution, Deployment → Compiler → Java Compiler**.
2. Under **Additional command line parameters**, add:
   ```
   --add-exports=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.doclint=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.platform=ALL-UNNAMED
   --add-exports=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED
   --add-exports=java.base/sun.reflect.annotation=ALL-UNNAMED
   --add-exports=java.base/sun.security.action=ALL-UNNAMED
   --add-exports=java.desktop/sun.font=ALL-UNNAMED
   --add-exports=java.desktop/sun.swing=ALL-UNNAMED
   --add-exports=java.desktop/sun.swing.icon=ALL-UNNAMED
   --add-exports=java.desktop/sun.awt=ALL-UNNAMED
   --add-exports=java.desktop/sun.awt.shell=ALL-UNNAMED
   ```
3. For test execution, add the same flags under **Run/Debug Configurations → VM Options**.

### Eclipse

Open the project's `.settings/org.eclipse.jdt.core.prefs` or use **Project Properties → Java Build Path → Module Dependencies** to add the same exports.

### VS Code (with Java extensions)

Add to `.vscode/settings.json`:
```json
{
  "java.project.referencedLibraries": [],
  "java.jdt.ls.vmargs": "--add-exports=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED --add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED"
}
```

## Module Overview

The Gosu project is organized into 12 Maven modules:

| Module | Description |
|--------|-------------|
| **gosu-parent** | Parent POM — centralizes dependency versions, plugin configuration, and `--add-exports` flags for all modules. |
| **gosu** | Distribution POM — assembles the final Gosu distribution archive. |
| **gosu-core-api** | Public API — defines interfaces for the type system, reflection, parsing, and compilation (`IType`, `ITypeLoader`, `IGosuClass`, etc.). |
| **gosu-core-api-precompiled** | Precompiled classes — ships pre-built `.class` files for bootstrap types needed before the Gosu compiler itself is available. |
| **gosu-core** | Core implementation — the Gosu compiler, parser, bytecode generator (via ASM), type system internals, and runtime support. |
| **gosu-lab** | Gosu Lab IDE — a Swing-based interactive development environment with an editor, REPL, and experiment management. |
| **gosu-doc** | Gosudoc — a Javadoc-style documentation generator adapted for Gosu source files. |
| **gosu-ant-tools** | Ant integration — provides custom Ant tasks for compiling and documenting Gosu projects. |
| **gosu-maven-compiler** | Maven compiler — a Plexus Compiler component that plugs Gosu compilation into the standard Maven build lifecycle. |
| **gosu-process** | Process utilities — helpers for launching and managing external OS processes from Gosu code. |
| **gosu-test** | Test suite — language-level tests covering the specification, errant-type verification, and regression tests. |
| **gosu-test-api** | Test API — base classes and utilities (`TestClass`, `TestEnvironment`) shared across test modules. |

## Test Patterns

### Errant Type Tests (`BaseVerifyErrantTest` + `processErrantType`)

Errant tests verify that the Gosu compiler correctly reports parse/compile errors. The pattern works as follows:

1. **Test class** extends `BaseVerifyErrantTest` and calls `processErrantType(Errant_Foo)`:
   ```gosu
   // File: SomeFeatureTest.gs
   class SomeFeatureTest extends BaseVerifyErrantTest {
     function testErrant_SomeFeatureTest() {
       processErrantType(Errant_SomeFeatureTest)
     }
   }
   ```

2. **Errant type class** (`Errant_SomeFeatureTest.gs`) contains intentionally broken code with special comment annotations:
   ```gosu
   class Errant_SomeFeatureTest {
     var x : int = "hello"  //## issuekeys: MSG_TYPE_MISMATCH
     var y : int = 42       // No annotation → no error expected here
   }
   ```

3. **How `processErrantType` works:**
   - Parses the errant type's source line-by-line
   - For each line with `//## issuekeys: KEY1, KEY2`, it asserts that the compiler produced exactly those errors on that line
   - For lines without annotations, it asserts no errors were produced
   - Lines with `//## KB(JIRA-123)` mark known breaks (skipped when `-Dgw.tests.skip.knownbreak=true`)

4. **Issue keys** correspond to constants in `gw.lang.parser.resources.Res` (e.g., `MSG_TYPE_MISMATCH`, `MSG_NO_SUCH_FUNCTION`).

### Specification Tests (`gw/specification/`)

Specification tests are organized under `gosu-test/src/test/gosu/gw/specification/` and map directly to sections of the Gosu language specification:

```
gw/specification/
├── expressions/           # Arithmetic, comparison, ternary, etc.
├── genericTypesAndMethods/ # Generics, type parameters, variance
├── statements/            # for-each, if/else, switch, try-catch, using
│   ├── assignmentStatements/
│   ├── blockStatements/
│   ├── forEachStatement/
│   ├── returnsExitsAndExceptions/
│   └── ...
├── types/                 # Primitive types, reference types, conversions
│   ├── primitiveTypes/
│   ├── referenceTypes/
│   ├── typeConversion/
│   └── ...
├── typeDynamic/           # Dynamic type behavior
├── structures/            # Structural typing
├── variablesParametersFieldsScope/  # Scoping rules
└── temp/                  # Exploratory/in-progress tests
```

Each test class typically extends either `TestClass` (for functional tests) or `BaseVerifyErrantTest` (for compile-error tests). Test methods follow the convention `testXxx()` or `testErrant_Xxx()`.

## PR Guidelines

1. **Branch from `master`** — create a feature branch for your work.
2. **Keep changes focused** — one logical change per PR.
3. **Add tests** — for new features, add specification tests under `gw/specification/`. For bug fixes that involve incorrect error reporting, add or update errant tests.
4. **Run the full build** before submitting:
   ```bash
   mvn clean verify -B
   ```
5. **Follow existing code style** — Gosu source uses 2-space indentation; Java source follows standard Java conventions.
6. **Update documentation** — if your change affects public API, update the relevant gosudoc comments.
7. **CI must pass** — the GitHub Actions workflow runs tests on Java 11 and Java 17 via a build matrix. Both must pass.
8. **Sign-off** — by submitting a PR, you agree that your contribution is provided under the [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0.txt).

## Useful Links

- [Gosu Language Home](http://gosu-lang.github.io/)
- [Discussion Forum](http://groups.google.com/group/gosu-lang)
- [Issue Tracker](https://github.com/gosu-lang/gosu-lang/issues)
- [Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0.txt)
