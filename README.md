# Directory.Packages.props: Central Package Management (CPM) in .NET

`Directory.Packages.props` is the file NuGet uses for **Central Package Management (CPM)**. It lets you declare every NuGet package version for a whole repository or solution in **one place**, instead of repeating `Version="..."` in every `.csproj`.

> A fully commented sample covering every option is in [`Directory.Packages.props`](./Directory.Packages.props) next to this README.
> Package names and version numbers in the sample are illustrative. Check the versions you actually want.

---

## Table of contents

1. [Why use it](#1-why-use-it)
2. [Requirements](#2-requirements)
3. [How it works](#3-how-it-works)
4. [Quick start](#4-quick-start)
5. [Reference: items and properties](#5-reference-items-and-properties)
6. [Use cases](#6-use-cases)
7. [File discovery, nesting and Directory.Build.props](#7-file-discovery-nesting-and-directorybuildprops)
8. [Tooling: CLI, Visual Studio, Dependabot, Renovate](#8-tooling)
9. [Migrating an existing solution](#9-migrating-an-existing-solution)
10. [Common errors and warnings](#10-common-errors-and-warnings)
11. [Best practices](#11-best-practices)
12. [FAQ](#12-faq)

---

## 1. Why use it

Without CPM, every project declares its own versions:

```xml
<!-- Api.csproj -->
<PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
<!-- Worker.csproj -->
<PackageReference Include="Newtonsoft.Json" Version="13.0.1" />   <!-- drifted -->
```

Problems this causes in multi-project solutions:

- **Version drift**: different projects end up on different versions of the same package.
- **Noisy upgrades**: bumping one package touches many `.csproj` files, which means big diffs and merge conflicts.
- **Hard audits**: you can't see the whole dependency list in one place.
- **Inconsistent transitive resolution**: different projects can resolve different transitive versions.

With CPM:

- Versions live in **one file**; project files only say *which* packages they use.
- Upgrades are a **one-line change**.
- Diffs are small and reviewable.
- You can **pin transitive dependencies** (for example to fix a vulnerable transitive package) for the entire repo.

---

## 2. Requirements

| Requirement | Minimum |
|---|---|
| NuGet | 6.2+ |
| .NET SDK | 6.0.300+ (any modern SDK, including .NET 8/9/10, is fine) |
| Visual Studio | 2022 17.2+ |
| Project style | SDK-style projects (`Microsoft.NET.Sdk`, `.Web`, `.Worker`, and so on) |

`GlobalPackageReference` requires NuGet 5.11+ / SDK 5.0.300+, so it is covered by the above.

CPM works with `PackageReference` only. It does not apply to legacy `packages.config` projects.

---

## 3. How it works

CPM splits package information into two places:

| Where | Item | Contains |
|---|---|---|
| `Directory.Packages.props` | `<PackageVersion Include="X" Version="1.2.3" />` | The **version** of package X |
| Each `.csproj` | `<PackageReference Include="X" />` | The **fact that this project uses** X (no version) |

At restore time NuGet looks up the version for each `PackageReference` in the central file.

The file is found by MSBuild's usual "walk up the directory tree" lookup: the **nearest** `Directory.Packages.props` above the project wins. It is imported late in evaluation, so properties set in the project body (for example `TargetFramework`) are available in conditions.

---

## 4. Quick start

**1. Create `Directory.Packages.props` at the repo/solution root**

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageVersion Include="Serilog" Version="4.2.0" />
  </ItemGroup>
</Project>
```

**2. Remove `Version` from every `PackageReference`**

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" />
  <PackageReference Include="Serilog" />
</ItemGroup>
```

**3. Restore and build**

```bash
dotnet restore
dotnet build
```

If a project still has `Version="..."` on a `PackageReference`, restore fails with **NU1008** (see [errors](#10-common-errors-and-warnings)).

---

## 5. Reference: items and properties

### 5.1 Items

| Item | Where | Purpose |
|---|---|---|
| `PackageVersion` | `Directory.Packages.props` | Declares the central version of a package. |
| `GlobalPackageReference` | `Directory.Packages.props` | Adds a package reference to **every** project automatically (analyzers, source link, build tools). Carries its own `Version`. |
| `PackageReference` | `.csproj` | Uses a package. No `Version` allowed (unless `VersionOverride`). |
| `PackageReference ... VersionOverride` | `.csproj` | Overrides the central version for that one project. |

### 5.2 Properties

| Property | Default | Meaning |
|---|---|---|
| `ManagePackageVersionsCentrally` | `false` | Master switch. `true` turns CPM on. Can be set to `false` in a single project to opt out. |
| `CentralPackageVersionOverrideEnabled` | `true` | Allows `VersionOverride` in projects. Set `false` to forbid overrides repo-wide. |
| `CentralPackageTransitivePinningEnabled` | `false` | When `true`, a `PackageVersion` also applies to **transitive** dependencies of that package id. |
| `CentralPackageFloatingVersionsEnabled` | `false` | Allows floating versions (`1.*`, `2.0.*-*`) in `PackageVersion`. Otherwise restore fails with NU1011. |

### 5.3 `PackageVersion` details

```xml
<!-- Add an entry. Condition is optional. -->
<PackageVersion Include="Package.Id" Version="1.2.3" Condition="..." />

<!-- Change an entry that was already added (for example from an imported parent file). -->
<PackageVersion Update="Package.Id" Version="1.2.4" />
```

- `Include` adds an entry; `Update` changes an entry that was already added (useful in nested files).
- `Version` follows normal NuGet version syntax: `1.2.3` (means "1.2.3 or higher", resolved to the lowest applicable), `[1.2.3]` (exact), `[1.0,2.0)` (range), and `1.*` (floating, needs the property above).
- The same package id can appear several times **only** if the entries are mutually exclusive through `Condition`.

### 5.4 `GlobalPackageReference` details

```xml
<GlobalPackageReference Include="StyleCop.Analyzers" Version="1.2.0-beta.556" PrivateAssets="all" />
```

- Applied to every project that imports the file, with no need to edit any `.csproj`.
- `Version` goes **on this item**. Do **not** also add a `PackageVersion` for the same id.
- Use `PrivateAssets="all"` (and optionally `IncludeAssets`/`ExcludeAssets`) so build-time tools don't flow to consumers of your packages.
- Because every project gets it, reserve it for things you genuinely want everywhere. A project that must not have one should live under a different `Directory.Packages.props` (see [nesting](#7-file-discovery-nesting-and-directorybuildprops)).

---

## 6. Use cases

Every example below is implemented in the sample file.

### 6.1 Basic central versions
One `PackageVersion` per package. The default and most common case.

### 6.2 Shared version for a package family (version properties)
Packages that must move together (for example `Microsoft.Extensions.*`, EF Core, or Azure Functions worker packages) can share an MSBuild property:

```xml
<PropertyGroup>
  <MicrosoftExtensionsVersion>10.0.0</MicrosoftExtensionsVersion>
</PropertyGroup>
<ItemGroup>
  <PackageVersion Include="Microsoft.Extensions.Logging"       Version="$(MicrosoftExtensionsVersion)" />
  <PackageVersion Include="Microsoft.Extensions.Configuration" Version="$(MicrosoftExtensionsVersion)" />
</ItemGroup>
```

### 6.3 Different versions per target framework
Multi-targeting solutions often need different package lines per TFM:

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="8.0.11" />
</ItemGroup>
<ItemGroup Condition="'$(TargetFramework)' == 'net10.0'">
  <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
</ItemGroup>
```

### 6.4 Per-project exception with `VersionOverride`
A single project needs a different version (legacy integration, temporary pin):

```xml
<!-- Legacy.csproj -->
<PackageReference Include="Newtonsoft.Json" VersionOverride="12.0.3" />
```

Use sparingly and add a comment explaining why. Disable the feature entirely with `CentralPackageVersionOverrideEnabled=false` if you want strict uniformity.

### 6.5 Global analyzers and build tools
`GlobalPackageReference` for code analyzers, style rules, SourceLink, versioning tools (for example Nerdbank.GitVersioning) so every project gets them with zero per-project configuration.

### 6.6 Pin transitive dependencies (security fixes)
A vulnerable package arrives indirectly (for example `System.Text.Json` via some library):

```xml
<PropertyGroup>
  <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
</PropertyGroup>
<ItemGroup>
  <PackageVersion Include="System.Text.Json" Version="9.0.0" />
</ItemGroup>
```

No project needs to reference `System.Text.Json` directly. The pinned version is used wherever it appears in the graph.

### 6.7 Floating versions
Allowed only when explicitly enabled. Handy for internal feeds or CI-only prerelease flows, but it makes builds non-deterministic.

```xml
<CentralPackageFloatingVersionsEnabled>true</CentralPackageFloatingVersionsEnabled>
...
<PackageVersion Include="Contoso.Internal.Sdk" Version="3.*" />
```

### 6.8 Test-only versions and per-project conditions
Conditions can use any property or item visible at evaluation time, such as `$(MSBuildProjectName)`, or a property you set in your own `Directory.Build.props`:

```xml
<PackageVersion Include="Moq" Version="4.20.72" Condition="$(MSBuildProjectName.EndsWith('.Tests'))" />
```

### 6.9 Opt a project out of CPM
In a project that cannot use CPM (for example a sample or a legacy project):

```xml
<PropertyGroup>
  <ManagePackageVersionsCentrally>false</ManagePackageVersionsCentrally>
</PropertyGroup>
```

That project goes back to ordinary `Version="..."` references.

### 6.10 Different rules in a subfolder (nested files)
The nearest file wins, and files **are not merged automatically**. To inherit from a parent, import it explicitly (see next section).

### 6.11 Related security/restore settings often kept in this file
`NuGetAudit`, `NuGetAuditMode`, `NuGetAuditLevel`, `RestorePackagesWithLockFile`, and `RestoreLockedMode` are not CPM features, but they pair naturally with central versions and are shown in the sample.

---

## 7. File discovery, nesting and Directory.Build.props

- MSBuild picks the **closest** `Directory.Packages.props` walking **up** from each project. Only one is used.
- To layer files (a repo-wide base plus a folder-specific extension), import the parent from the child:

```xml
<!-- src/legacy/Directory.Packages.props -->
<Project>
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Packages.props', '$(MSBuildThisFileDirectory)../'))" />

  <ItemGroup>
    <!-- Update changes a version that the parent already declared -->
    <PackageVersion Update="Newtonsoft.Json" Version="12.0.3" />
  </ItemGroup>
</Project>
```

- Don't confuse it with `Directory.Build.props`/`.targets`: those set general MSBuild properties and targets for every project below them. `Directory.Packages.props` is only for package versions (and the CPM switches). They are independent and are commonly used together at the repo root.
- Keep it at the **repo root** (next to the `.sln`/`.slnx`) so every project, test and tool is covered.

---

## 8. Tooling

- **dotnet CLI**: `dotnet add package Foo` with CPM enabled adds a `PackageVersion` to `Directory.Packages.props` and a version-less `PackageReference` to the project. `dotnet list package --outdated` and `--vulnerable` read the central versions.
- **Visual Studio**: the NuGet Package Manager UI understands CPM and edits the central file when you install or update.
- **Dependabot / Renovate**: both support `Directory.Packages.props`, so updates arrive as a single-file PR.
- **Other IDEs** (Rider, VS Code C# Dev Kit): supported through NuGet/MSBuild.

---

## 9. Migrating an existing solution

Manual approach:

1. Create `Directory.Packages.props` with `ManagePackageVersionsCentrally=true`.
2. List every distinct package with its chosen version as `PackageVersion`. Where projects disagree, pick one version, and use `VersionOverride` only where that is truly impossible.
3. Strip `Version` from all `PackageReference` items.
4. Restore, build, and run tests.

Automated approach: community tools such as **CentralisedPackageConverter** (`dotnet tool install -g CentralisedPackageConverter`) can do steps 2 and 3 for you. Review its output before committing.

Tip: do it in one dedicated PR, so version-alignment changes aren't mixed with feature work.

---

## 10. Common errors and warnings

| Code | Message (short) | Cause / fix |
|---|---|---|
| **NU1008** | Projects using CPM must not define `Version` on `PackageReference` | Remove `Version`, or use `VersionOverride` if an override is intended. |
| **NU1010** | `PackageReference` has no corresponding `PackageVersion` | Add `<PackageVersion Include="..." Version="..." />` to the central file. |
| **NU1011** | Floating versions in `PackageVersion` not allowed | Pin the version, or set `CentralPackageFloatingVersionsEnabled=true`. |
| **NU1507** | Multiple package sources and no source mapping | Warning when several feeds are configured. Add [package source mapping](https://learn.microsoft.com/nuget/consume-packages/package-source-mapping) in `nuget.config`. Recommended with CPM. |
| Override not applied | `VersionOverride` ignored/blocked | Check `CentralPackageVersionOverrideEnabled` isn't `false`. |
| Condition not matching | Version missing for one TFM | Make sure every TFM you build has a matching `PackageVersion` (or an unconditioned fallback). |
| Duplicate entries | Same id declared twice | Remove the duplicate or make the entries mutually exclusive with `Condition`. |

---

## 11. Best practices

1. **One file at the repo root.** Avoid nested files unless you really need layered rules.
2. **Group and comment.** Sort `PackageVersion` entries alphabetically or by purpose (runtime, Azure, testing, tooling), and explain unusual pins.
3. **Use version properties for families** that must move together.
4. **Use `GlobalPackageReference` for analyzers and build tools**, with `PrivateAssets="all"`.
5. **Prefer exact, deterministic versions**; avoid floating versions on main branches.
6. **Treat `VersionOverride` as technical debt**: comment it, and track its removal.
7. **Enable transitive pinning** to fix vulnerable transitive packages repo-wide, and review the pins periodically so they don't linger.
8. **Combine with NuGet Audit, lock files, and source mapping** for secure and reproducible restores.
9. **Let a bot propose updates** (Dependabot or Renovate) and rely on CI to catch breakage.

---

## 12. FAQ

**Does it change what ends up in my NuGet package's `.nuspec`?**
Packing resolves the central version into the generated dependency list, so consumers see the concrete version range as usual.

**Can I use it with `dotnet new` / `.slnx` / Directory.Build.props?**
Yes. They are independent mechanisms and work together.

**Do `PrivateAssets`, `IncludeAssets`, `ExcludeAssets` still go in the project?**
Yes. Only `Version` moves to the central file. Asset metadata stays on the `PackageReference` (or on the `GlobalPackageReference`).

**Does it work with Azure Functions, Blazor, Worker Services, test projects?**
Yes. Any SDK-style project that uses `PackageReference`.

**Is `Directory.Packages.props` case-sensitive?**
Keep the exact casing `Directory.Packages.props`. On Linux (CI and containers) file lookup is case-sensitive.

**How do I see what actually got resolved?**
`dotnet list package --include-transitive` or inspect `obj/project.assets.json`.

---

## References

- Microsoft Learn: *Central Package Management* (NuGet docs)
- Microsoft Learn: *Package source mapping*
- Microsoft Learn: *Auditing NuGet packages for security vulnerabilities*
