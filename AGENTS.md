# AGENTS.md — net-dns (Makaretu.Dns)

Guidance for agentic coding tools working in this repository.

## Project Overview

A pure C# DNS library targeting multiple .NET runtimes (`netstandard2.0`, `net462`, `net481`, `net8.0`, `net9.0`).
Root namespace: `Makaretu.Dns`. Resolver subsystem: `Makaretu.Dns.Resolving`.

---

## Build, Lint & Test Commands

### Restore dependencies
```bash
dotnet restore
```

### Build
```bash
dotnet build                        # debug, all TFMs
dotnet build --no-restore           # skip restore (CI pattern)
dotnet build -c Release ./src       # release build of library only
```

> **Warnings are errors.** The library project sets `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
> Every compiler warning will break the build; fix them, do not suppress them without a very good reason.

### Run all tests
```bash
dotnet test                         # build then test
dotnet test --no-build --verbosity normal   # CI pattern
```

### Run a single test (MSTest filter syntax)
```bash
# By fully-qualified name (most precise)
dotnet test --filter "FullyQualifiedName=Makaretu.Dns.MessageTest.DecodeQuery"

# By test class
dotnet test --filter "ClassName=Makaretu.Dns.MessageTest"

# By method name (partial, case-insensitive)
dotnet test --filter "Name=Roundtrip"

# Scoped to a specific runtime
dotnet test --framework net9.0 --filter "FullyQualifiedName=Makaretu.Dns.ARecordTest.Roundtrip"

# From the test project directory
dotnet test ./test --filter "Name=DecodeQuery"
```

Combine expressions with `&` (AND) and `|` (OR):
```bash
dotnet test --filter "ClassName=Makaretu.Dns.MessageTest&Name~Truncation"
```

### Code coverage (optional)
```bash
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover
```

---

## Project Layout

```
Dns.sln
src/            # Makaretu.Dns library (Dns.csproj)
  Resolving/    # NameServer, CachedNameServer, SecureNameServer, Catalog, etc.
  *.cs          # One DNS record type / concept per file
test/           # MSTest project (DnsTests.csproj)
  Resolving/    # Tests mirroring src/Resolving/
  *Test.cs      # One test class per source class
doc/            # DocFX documentation project
.github/
  workflows/
    dotnet.yml  # CI: restore → build → test (ubuntu-latest, .NET 9)
    codeql.yml  # Security scanning
```

---

## Code Style Guidelines

### File & Namespace Organisation
- **One class per file**; file name must exactly match the class name (`ARecord.cs` → `class ARecord`).
- Use **traditional braced namespace syntax** — do NOT use file-scoped namespaces (`namespace Foo;`).
- `using` directives go at the top of the file, before the namespace declaration.
- Order `using` directives: `System` first, then `System.*` sub-namespaces alphabetically.
- No third-party dependencies in `src/`; the library is self-contained aside from the BCL.

### Naming Conventions
| Identifier | Style | Example |
|---|---|---|
| Classes, structs, enums | PascalCase | `DomainName`, `ARecord`, `MessageOperation` |
| Interfaces | `I`-prefixed PascalCase | `IResolver`, `IWireSerialiser` |
| Public properties | PascalCase | `Questions`, `TTL`, `AuthorityRecords` |
| Public methods | PascalCase | `ReadData`, `ToCanonical`, `ResolveAsync` |
| Async methods | `Async` suffix | `ResolveAsync`, `FindAnswerAsync` |
| Private fields | camelCase, **no** underscore prefix | `opcode4`, `labels`, `stream` |
| Local variables | camelCase | `opt`, `extendedOpcode` |
| Constants (public) | PascalCase | `MaxLength`, `MinLength` |

### Braces & Formatting
- **Allman brace style**: opening brace on its own line for classes, methods, and control-flow blocks.
- Short single-expression property getters may be on one line:
  ```csharp
  public bool IsQuery { get { return !QR; } }
  ```
- Short guard `if` statements without braces are acceptable:
  ```csharp
  if (opt == null) return (MessageOperation)opcode4;
  ```
- Use object-initializer syntax where it aids readability:
  ```csharp
  new Message { QR = true, Id = 1234 }
  ```

### Types & Generics
- Prefer explicit BCL types over `var` when the type is not immediately obvious from the right-hand side.
- Optional parameters: use default `null` for reference types (`DateTime? from = null`) and
  `default(CancellationToken)` (not `default`) for `CancellationToken` parameters — match existing style.
- Use `ConcurrentDictionary` / `ConcurrentSet<T>` for thread-safe collections in the Resolving subsystem.
- LINQ is used extensively; prefer it over explicit loops when clarity is not compromised.

### XML Documentation
- **All public types and members must have XML doc comments.**
- Standard tags: `<summary>`, `<remarks>`, `<param>`, `<returns>`, `<value>`, `<exception>`.
- Use `<see href="...">` with full IETF URLs when referencing RFCs.
- Use `<b>true</b>` / `<b>false</b>` for inline boolean values.
- Use `/// <inheritdoc />` on overriding members.

### Error Handling
- Use standard BCL exceptions — do **not** introduce custom exception types.
  - Malformed wire/presentation data → `InvalidDataException`
  - Unexpected end of stream → `EndOfStreamException`
  - Invalid arguments → `ArgumentException` / `ArgumentNullException`
- Use `#pragma warning disable/restore` only for specific, named warning IDs (e.g., `IDE0041`),
  never to suppress broad categories. Add a comment explaining why.

### Async
- All resolver and I/O methods that may block must be `async Task<T>` with `CancellationToken cancel`.
- Use `Task.FromResult(...)` for synchronous fast-paths that must conform to an async interface.
- Do **not** call `.Result` or `.Wait()` on tasks; propagate `async`/`await` up the call stack.

### Object Model
- Inherit from `DnsObject` (implements `IWireSerialiser`, `ICloneable`) for all DNS objects.
- Record types inherit: `DnsObject` → `ResourceRecord` → specific record (e.g., `ARecord`).
- Override `ReadData(WireReader, int)` and `WriteData(WireWriter)` for binary serialisation.
- Override `ReadData(PresentationReader)` and `WriteData(PresentationWriter)` for master-file format.

---

## Test Guidelines

### Test File Conventions
- Each source file `Foo.cs` has a corresponding `FooTest.cs` in the same relative path under `test/`.
- Test classes use the same namespace as the production code (e.g., `namespace Makaretu.Dns`).
- Decorate test classes with `[TestClass]` and test methods with `[TestMethod]`.
- Async test methods: `public async Task MethodName()` — do not use `.Result` in tests.

### Test Naming
- PascalCase method names; use underscores to separate scenario from condition:
  ```
  Roundtrip()
  Roundtrip_Master()
  Missing_Name()
  MultipleQuestions_AnswerAll()
  Truncation_NotRequired()
  Issue_42()        // regression test for GitHub issue #42
  ```

### Common Test Patterns
- **Roundtrip**: serialise to bytes → deserialise → assert all properties match. Write one for
  every new record type or serialisable object.
- **Exception assertion**: use the provided `ExceptionAssert.Throws<T>(Action, string?)` helper
  (defined in `test/ExceptionAssert.cs`, namespace `Makaretu`).
- Prefer `Assert.AreEqual`, `Assert.IsTrue`, `Assert.IsInstanceOfType` (MSTest assertions).
- Use `MemoryStream` + `WireWriter`/`WireReader` for low-level binary roundtrip tests.

---

## CI

The GitHub Actions workflow (`.github/workflows/dotnet.yml`) runs on every push and PR to `master`:
1. `dotnet restore`
2. `dotnet build --no-restore`
3. `dotnet test --no-build --verbosity normal`

All three steps must pass. CodeQL security scanning also runs on push/PR to `master`.
