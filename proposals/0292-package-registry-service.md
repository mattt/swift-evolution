# Package Registry Service

* Proposal: [SE-0292](0292-package-registry-service.md)
* Authors: [Bryan Clark](https://github.com/clarkbw),
           [Whitney Imura](https://github.com/whitneyimura),
           [Mattt Zmuda](https://github.com/mattt)
* Review Manager: [Tom Doron](https://github.com/tomerd)
* Status: **Returned for revision**
* Implementation: [apple/swift-package-manager#3023](https://github.com/apple/swift-package-manager/pull/3023)
* Review: [Review](https://forums.swift.org/t/se-0292-package-registry-service/)

## Introduction

Swift Package Manager downloads dependencies using Git.
Our proposal defines a standard web service interface
that it can also use to download dependencies from a package registry.

## Motivation

A package dependency is specified by a URL for its source repository.
When a project is built for the first time,
Swift Package Manager clones the Git repository for each dependency
and attempts to resolve the version requirements from the available tags.

Although Git is a capable version-control system,
it's not well-suited to this kind of workflow for the following reasons:

* **Reproducibility**:
  A version tag in the Git repository for a dependency
  can be reassigned to another commit at any time.
  This can cause the same source code to produce different build results
  depending on when it was built.
* **Availability**:
  The Git repository for a dependency can be moved or deleted,
  which can cause subsequent builds to fail.
* **Efficiency**:
  Cloning the Git repository for a dependency
  downloads all versions of a package when only one is used at a time.
* **Speed**:
  Cloning a Git repository for a dependency can be slow
  for repositories with large histories.

Many language ecosystems have a <dfn>package registry</dfn>, including
[RubyGems] for Ruby,
[PyPI] for Python,
[npm] for JavaScript, and
[crates.io] for Rust.
In fact,
many Swift developers make apps today using
[CocoaPods] and its index of libraries.

A package registry can offer faster and more reliable dependency resolution
than downloading dependencies using Git.
It can also support other useful functionality,
such as:

* **Security advisories**:
  Vulnerabilities can be communicated directly to package consumers
  in a timely manner.
* **Discoverability**:
  Package maintainers can annotate their releases with project metadata,
  including its authors, license, and other information.
* **Search**:
  A registry can provide a standard interface for searching available packages,
  or provide the information necessary for others to create a search index.
* **Flexibility**:
  Swift Package Manager requires that external dependencies
  be hosted in a Git repository with a package manifest located in its root.
  For some projects,
  this requirement is a barrier to adopting Swift Package Manager.
  A package registry imposes no requirements on
  version control software or project structure.

## Proposed solution

This proposal defines a standard interface for package registry services,
and describes how Swift Package Manager can integrate with them
to download dependencies.

The goal of this proposal is to make dependency resolution
more available and reproducible.
We believe our proposed solution
can meet or exceed the current performance of dependency resolution
and will allow for new functionality to be built in the future.

## Detailed design

### Package registry service

A package registry service implements the following REST API endpoints
for listing releases for a package,
fetching information about a release,
and downloading the source archive for a release:

| Method | Path                                                             | Description                                      |
| ------ | ---------------------------------------------------------------- | ------------------------------------------------ |
| `GET`  | `/{namespace}/{package}`                                         | List package releases                            |
| `GET`  | `/{namespace}/{package}/{version}`                               | Fetch metadata for a package release             |
| `GET`  | `/{namespace}/{package}/{version}/Package.swift{?swift-version}` | Fetch manifest for a package release             |
| `GET`  | `/{namespace}/{package}/{version}.zip`                           | Download source archive for a package release    |

A formal specification for the package registry interface
is provided alongside this proposal.
In addition,
an OpenAPI (v3) document,
a reference implementation written in Swift,
and a benchmarking harness
are provided for the convenience of developers interested
in building their own package registry.

### Changes to Swift Package Manager

#### Package identity

Currently, the identity of a package is computed from
the last path component of its effective URL
(which can be changed with dependency mirroring).
However, this approach can lead to conflation of
distinct packages with similar names
as well as duplication of the same package under different names.

We propose using a namespace-scoped identifier
in the form `@namespace/PackageName`
to identify package dependencies.

A package *namespace* designates a single individual or organization
within a package registry.
A namespace consists of
an at-sign (`@`) followed by alphanumeric characters and hyphens.
Hyphens may not occur at the beginning or end,
nor consecutively within a package namespace.
The maximum length of a package namespace is 40 characters.
A valid package namespace matches the following regular expression pattern:

```regexp
\A@[a-zA-Z\d](?:[a-zA-Z\d]|-(?=[a-zA-Z\d])){0,39}\z
```

A package's *name* is specified by the `name` parameter
provided in its manifest (`Package.swift`) file.
A package name must be unique within the scope of its namespace.

The maximum length of a package namespace is 128 characters.
A valid package namespace matches the following regular expression pattern:

```regexp
\A\p{XID_Start}\p{XID_Continue}{0,127}\z
```

> For more information,
> see [Unicode Identifier and Pattern Syntax][UAX31].

Package namespaces are case-insensitive
(for example, `mona` ≍ `MONA`).
Package names are
case-insensitive
(for example, `mona` ≍ `MONA`),
diacritic-insensitive
(for example, `Å` ≍ `A`), and
width-insensitive
(for example, `Ａ` ≍ `A`).
Package names are compared using
[Normalization Form Compatible Composition (NFKC)][UAX15].

#### New `PackageDescription` APIs

The `Package.Dependency` type adds the following static methods:

```swift
extension Package.Dependency {
    /// Adds a dependency on a package with the specified identifier
    /// that uses the provided version requirement.
    public static func package(
        _ id: String,
        url: String? = nil,
        _ requirement: Package.Dependency.Requirement
    ) -> Package.Dependency

    /// Adds a dependency on a package with the specified identifier
    /// that uses the provided version requirement
    /// starting with the given minimum version,
    /// going up to the next major version.
    public static func package(
        _ id: String,
        path: String? = nil,
        from version: Version
    ) -> Package.Dependency
}
```

These methods may be called in the `dependencies` field of a package manifest
to declare one or more dependencies by their respective package identifier.

```swift
dependencies: [
   .package("@mona/LinkedList", from: "1.1.0"),
   .package("@mona/RegEx", .exact("2.0.0"))
]
```

If an explicit `url` or `path` is provided,
Swift Package Manager resolves the dependency using Git,
assigning the specified identifier to the package.

```swift
dependencies: [
   .package("@mona/LinkedList", url: "https://mona.example.com/LinkedList" from: "1.1.0"),
   .package("@mona/RegEx", path: "/Users/mona/Code/Regex", .exact("2.0.0"))
]
```

#### Automatic migration for URL-based dependencies

For compatibility with previous versions of Swift,
Swift Package Manager may resolve a `.package` dependency declaration
that specifies a `name` and `url`
using its equivalent package identity
if it satisfies the following criteria:

* The dependency has a url with an `https` scheme
  and a hostname equal to `github.com`.
* The dependency url doesn't contain
  a userinfo component,
  a port subcomponent,
  a query component,
  or a fragment component.
* The path component of the dependency url
  consists of two segments separated by a slash (`/`)
  with no trailing slash.
  * The first path segment is a valid package identifier namespace.
  * The last path segment is a valid package name
    equal to the declared package name,
    and doesn't contain a file extension (for example, `.git`).
* The dependency specifies an exact version or range of versions.

Packages with an alternate location specified by a [dependency mirror][SE-0219]
can be resolved only using Git.

The resulting package identity consists of an at sign (`@`)
followed by the first path component of the url,
a slash (`/`),
and the package name.
For example,
here are a list of dependencies that do and don't qualify:

```swift
// ✅ These dependencies will resolve through a package registry
//    using the package identity `@mona/LinkedList`.
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList", from: "1.1.0")
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList", .exact("1.1.0"))
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList", .upToNextMajor(from: "1.1.0"))
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList", .upToNextMinor(from: "1.1.0"))

// ❌ These dependencies can only be resolved using Git
.package(url: "https://github.com/mona/LinkedList", from: "1.1.0") // No name
.package(name: "SwiftLinkedList", url: "https://github.com/mona/LinkedList", from: "1.1.0") // Name doesn't match last path component of url
.package(name: "LinkedList", url: "git@github.com:mona/LinkedList.git", from: "1.1.0") // No https scheme
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList.git", from: "1.1.0") // .git file extension
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList", .branch("master")) // No version
.package(name: "LinkedList", url: "https://github.com/mona/LinkedList", .revision("d6ca4e56219a8a5f0237d6dcdd8b975ec7e24c89")) // No version
.package(path: "../LinkedList") // No url or name
```

#### Module name collision resolution

Swift Package Manager cannot build a project
if any of the following are true:

1. Two or more packages in the project
   are located by URLs with the same (case-insensitive) last path component
   (for example, `github.com/mona/LinkedList` and `github.com/OctoCorp/linkedlist`)
2. Two or more packages in the project
   declare the same name in their package manifest
   (for example, `let package = Package(name: "LinkedList"`))
3. Two or more modules provided by packages in the project
   have the same name
   (`let package = Package(products: [.library(name: "LinkedList")])`)

This proposal directly addresses points #2 and #3
and describes a possible resolution for point #1 under
["Future Directions"](#package-dependency-url-normalization).

Consider the following package manifest,
which Swift Package Manager currently fails to resolve
for all three of the reasons listed above
(assume both external dependencies are named "LinkedList"
and contain a library product named "LinkedList").

```swift
// swift-tools-version:5.3
import PackageDescription

let package = Package(name: "Example",
                      dependencies: [
                        .package(name: "LinkedList",
                                 url: "https://github.com/mona/LinkedList",
                                 from: "1.1.0"),
                        .package(name: "LinkedList",
                                 url: "https://github.com/OctoCorp/LinkedList",
                                 from: "0.1.0")
                      ],
                      targets: [
                        .target(
                            name: "Greeter",
                            dependencies: [
                                .product(name: "LinkedList",
                                         package: "LinkedList"), // github.com/mona/LinkedList
                            ])
                        .target(
                            name: "Meeter",
                            dependencies: [
                                .product(name: "LinkedList",
                                         package: "LinkedList") // github.com/OctoCorp/linkedlist
                            ])
                      ])
```

Ambiguous `name` parameters for dependency `.package` declarations
can be resolved by using namespaced package identifiers.

```diff
-                        .package(name: "LinkedList",
-                                 url: "https://github.com/mona/LinkedList",
+                        .package("@mona/LinkedList",
                                  from: "1.1.0"),

-                        .package(name: "LinkedList",
-                                 url: "https://github.com/OctoCorp/LinkedList",
+                        .package(name: "@OctoCorp/LinkedList",
                                  from: "0.1.0")
```

Package identifiers can be used in `.product` declarations
to unambiguously reference a particular package's module.

```diff
                         .product(name: "LinkedList",
-                                 package: "LinkedList"),
+                                 package: "@mona/LinkedList"),

                         .product(name: "LinkedList",
-                                 package: "LinkedList")
+                                 package: "@OctoCorp/LinkedList")
```

For compatibility and convenience,
`.product` declarations may reference a package without its respective namespace
if that package's name is unique within the dependency graph.

#### Dependency graph resolution

In its `PackageGraph` module, Swift Package Manager defines
the `PackageContainer` protocol as the top-level unit of package resolution.
Conforming types are responsible for
determining the available tags for a package
and its contents at a particular revision.
A `PackageContainerProvider` protocol adds a level of indirection
for resolving package containers.

There are currently two concrete implementations of `PackageContainer`:
`LocalPackageContainer` and `RepositoryPackageContainer`.
This proposal adds a new `RegistryPackageContainer` type
that adopts `PackageContainer`
and performs equivalent operations with HTTP requests to a registry service.
These client-server interactions are facilitated by
a new `RegistryManager` type.

The following table lists the
tasks performed by Swift Package Manager during dependency resolution
alongside the Git operations used
and their corresponding package registry API calls.

| Task                                  | Git operation               | Registry request                         |
| ------------------------------------- | --------------------------- | ---------------------------------------- |
| Fetch the contents of a package       | `git clone && git checkout` | `GET /{namespace}/{package}/{version}.zip`           |
| List the available tags for a package | `git tag`                   | `GET /{namespace}/{package}`                         |
| Fetch a package manifest              | `git clone`                 | `GET /{namespace}/{package}/{version}/Package.swift` |

Package registries support
[version-specific _manifest_ selection][version-specific-manifest-selection]
by providing a list of versioned manifest files for a package
(for example, `Package@swift-5.3.swift`)
in its response to `GET /{namespace}/{package}/{version}/Package.swift`.
However, package registries don't support
[version-specific _tag_ selection][version-specific-tag-selection],
and instead rely on [Semantic Versioning][SemVer]
to accommodate different versions of Swift
(for example,
by using major release versions
or build metadata like `1.0.0+swift-5_3`).

### Changes to `Package.resolved`

Swift package registry releases are archived as Zip files.

When an external package dependency is downloaded through a registry,
Swift Package Manager compares the integrity checksum provided by the server
against any existing checksum for that release in `Package.resolved`
as well as the integrity checksum reported by the `compute-checksum` subcommand:

```terminal
$ swift package compute-checksum LinkedList-1.2.0.zip
1feec3d8d144814e99e694cd1d785928878d8d6892c4e59d12569e179252c535
```

If no prior checksum exists,
it's saved to `Package.resolved`.

```json
{
    "object": {
        "pins": [
            {
                "package": "@mona/LinkedList",
                "state": {
                    "checksum": "ed008d5af44c1d0ea0e3668033cae9b695235f18b1a99240b7cf0f3d9559a30d",
                    "version": "1.2.0"
                }
            }
        ]
    },
    "version": 1
}
```

If the checksum reported by the server is different from the existing checksum
(or the checksum of the downloaded artifact is different from either of them),
that's an indication that a package's contents may have changed at some point.
Swift Package Manager will refuse to download dependencies
if there's a mismatch in integrity checksums.

```terminal
$ swift build
error: checksum of downloaded source archive of dependency '@mona/LinkedList' (c2b934fe66e55747d912f1cfd03150883c4f037370c40ca2ad4203805db79457) does not match checksum specified by the manifest (ed008d5af44c1d0ea0e3668033cae9b695235f18b1a99240b7cf0f3d9559a30d)
```

Once the correct checksum is determined,
the user can update `Package.resolved` with the correct value
and try again.

### Archive-source subcommand

An anecdotal look at other package managers suggests that
a checksum mismatch is more likely to be a
disagreement in how to create the archive and/or calculate the checksum
than, say, a forged or corrupted package.

This proposal adds a new `swift package archive-source` subcommand
to provide a standard way to create source archives for package releases.

```manpage
SYNOPSIS
	swift package archive-source [--output=<file>]

OPTIONS
	-o <file>, --output=<file>
		Write the archive to <file>.
		If unspecified, the package is written to `\(PackageName).zip`.
```

Run the `swift package archive-source` subcommand
in the root directory of a package
to generate a source archive for the current working tree.
For example:

```terminal
$ tree -a -L 1
LinkedList
├── .git
├── Package.swift
├── README.md
├── Sources
└── Tests

$ head -n 5 Package.swift
// swift-tools-version:5.3
import PackageDescription

let package = Package(
name: "LinkedList",

$ swift package archive-source
Created LinkedList.zip
```

By default,
the filename of the generated archive is
the name of the package with a `.zip` extension
(for example, "LinkedList.zip").
This can be configured with the `--output` option:

```terminal
$ git checkout 1.2.0
$ swift package archive-source --output="LinkedList-1.2.0.zip"
# Created LinkedList-1.2.0.zip
```

The `archive-source` subcommand has the equivalent behavior of
[`git-archive(1)`] using the `zip` format at its default compression level.
Therefore, the following command produces
equivalent output to the previous example:

```terminal
$ git archive --format zip --output LinkedList-1.2.0.zip 1.2.0
```

If desired, this behavior may be changed in future tool versions.

> **Note**:
> `git-archive` ignores files with the `export-ignore` Git attribute.
> By default, this ignores hidden files and directories,
> including`.git` and `.build`.

### Changes to config subcommand

#### Set-registry subcommand

By default,
Swift Package Manager resolves package dependencies with namespaced identifiers
using a predefined package registry.

This proposal adds a new `swift package config set-registry` subcommand
that lets users specify one or more comma-delimited (`,`) registry URLs
to be consulted, in order,
when resolving dependencies through the package registry interface.

```manpage
SYNOPSIS
	swift package config set-registry <url>[,<url>]...

OPTIONS
	-o <file>, --output=<file>
		Write the archive to <file>.
		If unspecified, the package is written to `\(PackageName).zip`.
```

Running this subcommand in the root directory of a package
creates or updates the `.swiftpm/config` file
with a new top-level `registries` key
that's associated with an array containing the specified registry URLs.

```json
{
  "registries": [
      "https://internal.example.com"
  ],
  "version": 1
}

```

For example,
a build server that doesn't allow external network connections
may configure a registry URL to resolve dependencies
using an internal registry service.

```terminal
$ swift package config set-registry https://internal.example.com/
```

A custom registry can be used for a variety of purposes:

- **Geographic colocation**:
  Developers working under adverse networking conditions can
  host a mirror of official package sources on a nearby network.
- **Policy enforcement**:
  A corporate network can enforce quality or licensing standards,
  so that only approved packages are available.
- **Auditing**:
  A registry may analyze or meter access to packages
  for the purposes of ranking popularity or charging licensing fees.

#### Unset-registry subcommand

This proposal also adds a new `swift package config unset-registry` subcommand
to complement the `set-registry` command.

Running the `unset-registry` subcommand in the root directory of a package
updates the `.swiftpm/config` file
to remove any top-level `registries` key.
If the resulting configuration is empty
(that is, containing only a top-level `version` key),
this command deletes the `.swiftpm/config` file.

#### Set-mirror option for package identifiers

A user can currently specify an alternate location for a package
by setting a [dependency mirror][SE-0219] for that package's URL.

```terminal
$ swift package config set-mirror \
    --original-url https:///github.com/mona/linkedlist \
    --mirror-url https:///github.com/octocorp/swiftlinkedlist
```

This proposal updates the `swift package config set-mirror` subcommand
to accept a `--package-identifier` option in place of an `--original-url`.
Running this subcommand with a `--package-identifier` option
creates or updates the `.swiftpm/config` file,
modifying the array associated with the top-level `object` key
to add a new entry or update an existing entry
for the specified package identifier,
that assigns its alternate location.

```json
{
  "object": [
    {
      "mirror": "https://github.com/OctoCorp/SwiftLinkedList.git",
      "original": "@mona/LinkedList"
    }
  ],
  "version": 1
}

```

When a mirror URL is set for a package identifier,
Swift Package Manager resolves any dependencies with that identifier
through Git using the provided URL.

## Security

Adding external dependencies to a project
increases the attack surface area of your software.
However, much of the associated risk can be mitigated,
and a package registry can offer stronger guarantees for safety and security
compared to downloading dependencies using Git.

Core security measures,
such as use of HTTPS and integrity checksums,
are required by the registry service specification.
Additional decisions about security
are delegated to to the registries themselves.
For example,
registries are encouraged to adopt a
scoped, revocable authorization framework like [OAuth 2.0][RFC 6749],
but this isn't a strict requirement.
Package maintainers and consumers should
consider a registry's security posture alongside its other features
when deciding where to host and fetch packages.

Our proposal's package identity scheme is designed to prevent or mitigate
vulnerabilities common to packaging systems and networked applications:

- Namespaces are clearly marked by an at-sign (`@`) prefix
  and restricted to a limited set of characters,
  which prevents [homograph attacks].
  For example,
  "А" (U+0410 CYRILLIC CAPITAL LETTER A) is an invalid namespace character,
  and cannot be confused for "A" (U+0041 LATIN CAPITAL LETTER A).
- Namespaces disallow leading, trailing, or consecutive hyphens (`-`),
  which mitigates look-alike namespace and package names
  (for example, "@llvm--swift",
  is invalid and cannot be confused for "@llvm-swift")
- Packages are registered within a namespace scope,
  which mitigates [typosquatting].
  Package registries may further restrict the assignment of new namespaces
  that are intentionally misleading
  (for example "@G00gle", which looks like "@Google").
- Package names must be valid Swift identifiers,
  which disallow punctuation and whitespace characters used in
  [cross-site scripting][xss] and
  [CRLF injection][http header injection] attacks.

To better understand the security implications of this proposal —
and Swift dependency management more broadly —
we employ the
<abbr title="Spoofing, Tampering, Repudiation, Information disclosure, Denial of Service, Escalation of privilege">
[STRIDE]
</abbr> mnemonic below:

### Spoofing

An attacker could interpose a proxy between the client and the package registry
to intercept credentials for that host
and use them to impersonate the user in subsequent requests.

The impact of this attack is potentially high,
depending on the scope and level of privilege associated with these credentials.
However, the use of secure connections over HTTPS
goes a long way to mitigate the overall risk.

Swift Package Manager could further mitigate this risk
by taking the following measures:

* Enforcing HTTPS for all URLs
* Resolving URLs using DNS over HTTPS (DoH)
* Requiring URLs with Internationalized Domain Names (IDNs)
  to be represented as Punycode

### Tampering

An attacker could interpose a proxy between the client and the package registry
to construct and send Zip files containing malicious code.

Although the impact of such an attack is potentially high,
the risk is largely mitigated by the use of cryptographic checksums
to verify the integrity of downloaded source archives.

```terminal
$ echo "$(swift package compute-checksum LinkedList-1.2.0.zip) *LinkedList-1.2.0.zip" | \
    shasum -a 256 -c -
LinkedList-1.2.0.zip: OK
```

Integrity checks alone can't guarantee
that a package isn't a forgery;
an attacker could compromise the website of the host
and provide a valid checksum for a malicious package.

`Package.resolved` provides a [Trust on first use (TOFU)][TOFU] security model
that can offer strong guarantees about the integrity of dependencies over time.
A registry can further improve on this model
by implementing a [transparent log] or some comparable,
tamper-proof system for associating artifacts with valid checksums.

### Repudiation

A compromised host could serve a malicious package with a valid checksum
and be unable to deny its involvement in constructing the forgery.

This threat is unique and specific to binary and source artifacts;
Git repositories can have their histories audited,
and individual commits may be cryptographically signed by authors.
Unless you can establish a direct connection between
an artifact and a commit in a source tree,
there's no way to determine the provenance of that artifact.

Both a checksum database and the use of digital signatures
can provide similar non-repudiation guarantees.

### Information disclosure

A user may inadvertently reveal the existence of a private registry service
or expose hardcoded credentials
by checking in their project's `.swiftpm/config` file.

An attacker could scrape public code repositories for `.swiftpm/config` files
and attempt to reuse those credentials to impersonate the user.

```json
{
  "registries": [
      "https://<USERNAME>:<TOKEN>@swift.pkg.github.com/<OWNER>/"
  ],
  "version": 1
}

```

This kind of attack can be mitigated on an individual basis
adding `.swiftpm/config` to a project's `.gitignore` file.
The risk could be mitigated for all users
if Swift Package Manager included a `.gitignore` file
in its new project template.
Code hosting providers can also help mitigate this risk
by [detecting secrets][secret scanning]
that are committed to public repositories.

> **Important**:
> Never store credentials in code.

### Denial of service

An attacker could scrape public code repositories
for `.swiftpm/config` files that declare one or more custom registries
and launch a denial-of-service attack
in an attempt to reduce the availability of those resources.

The likelihood of this attack is generally low
but could be used in a targeted way
against resources known to be important or expensive to distribute.

### Escalation of privilege

Even authentic packages from trusted creators can contain malicious code.

Code analysis tools can help to some degree,
as can system permissions and other OS-level security features.
But developers are ultimately the ones responsible
for the code they ship to users.

## Impact on existing packages

Current packages won't be affected by this change,
as they'll continue to be able to download dependencies directly through Git.

Swift Package Manager can migrate existing package dependencies
to take advantage of Swift package registries
without changing their specification,
as described in
"[Automatic migration for URL-based dependencies"](#automatic-migration-for-url-based-dependencies).

## Alternatives considered

### Use of alternative naming schemes

Some package systems,
including [RubyGems], [PyPI], and [CocoaPods]
identify packages with bare names in a flat namespace
(for example, `rails`, `pandas`, or `Alamofire`).
Other systems,
including [Maven],
use [reverse domain name notation] to identify software components
(for example, `com.squareup.okhttp3`).

We considered these and other schemes for identifying packages,
but they were rejected in favor of a namespace-scoped package identity
similar to the one used by [npm].

### Use of `tar` or other archive formats

Swift Package Manager currently uses Zip archives for binary dependencies,
which is reason enough to use it again here.

We briefly considered `tar` as an archive format
but concluded that its behavior of preserving symbolic links and executable bits
served no useful purpose in the context of package management,
and instead raised concerns about portability and security.

> As an aside,
> Zip files are also a convenient format for package registries,
> because they support the access of individual files within an archive.
> This allows a registry to satisfy
> the package manifest endpoint (`GET /{namespace}/{package}/{version}/Package.swift`)
> without storing anything separately from the archive used for the
> package archive endpoint (`GET /{namespace}/{package}/{version}.zip`).

### Addition of an `unarchive-source` subcommand

This proposal adds an `archive-source` subcommand
as a standard way for developers and registries
to create source archives for packages.
Having a canonical tool for creating source archives
avoids any confusion when attempting to verify the integrity of
Zip files sent from a registry
with the source code for that package.

We considered including a complementary `unarchive-source` subcommand
but ultimately decided against it,
reason being that unarchiving a Zip archive
is unambiguous and well-supported on most platforms.

### Use of digital signatures

[SE-0272] includes discussion about
the use of digital signatures for binary dependencies,
concluding that they were unsuitable
because of complexity around transitive dependencies.
However, it's unclear what specific objections were raised in this proposal.
We didn't see any inherent tension with the example provided,
and no further explanation was given.

Without understanding the context of this decision,
we decided it was best to abide by their determination
and instead consider adding this functionality in a future proposal.
For the reasons outlined in the preceding Security section,
we believe that digital signatures may offer additional guarantees
of authenticity and non-repudiation beyond what's possible with checksums alone.

## Future directions

Defining a standard interface for package registries
lays the groundwork for several useful features.

### Package dependency URL normalization

As described in ["Module name collision resolution"](#module-name-collision-resolution)
Swift Package Manager cannot build a project
if two or more packages in the project
are located by URLs with the same (case-insensitive) last path component.
Swift Package Manager may improve support URL-based dependencies
by normalizing package URLs to mitigate insignificant variations.
For example,
a package with an ["scp-style" URL][scp-url] like
`git@github.com:mona/LinkedList.git`
may be determined to be equivalent to a package with an HTTPS scheme like
`https:///github.com/mona/LinkedList`.

### Offline cache

Swift Package Manager could implement an [offline cache]
that would allow it to work without network access.
While this is technically possible today,
a package registry makes for a simpler and more secure implementation
than would otherwise be possible with Git repositories alone.

### Package publishing

A package registry is responsible for determining
which package releases are made available to a consumer.
This proposal sets no policies for how
package releases are published to a registry.

Many package managers —
including the ones mentioned above —
and artifact repository services, such as
[Docker Hub],
[JFrog Artifactory],
and [AWS CodeArtifact]
follow what we describe as a <dfn>"push"</dfn> model of publication:
When a package owner wants to releases a new version of their software,
they produce a build locally and push the resulting artifact to a server.
This model has the benefit of operational simplicity and flexibility.
For example,
maintainers have an opportunity to digitally sign artifacts
before uploading them to the server.

Alternatively,
a system might incorporate build automation techniques like
continuous integration (CI) and continuous delivery (CD)
into what we describe as a "pull" model:
When a package owner wants to release a new version of their software,
their sole responsibility is to notify the package registry;
the server does all the work of downloading the source code
and packaging it up for distribution.
This model can provide strong guarantees about
reproducibility, quality assurance, and software traceability.

We intend to work with industry stakeholders
to develop standards for publishing Swift packages
in a future, optional extension to the registry specification.

### Package removal

There are several reasons why a package release may be removed, including:

* The package maintainer publishing a release by mistake
* A security vulnerability being discovered in a release
* The registry being compelled by law enforcement to remove a release

However, removing a package release has the potential to
break any packages that depend on it.
Many package management systems have their own processes for
how removal works (or whether it's supported in the first place).

It's unclear whether or to what extent such policies should be
informed by registry specification itself.
For now,
a registry is free to exercise its own discretion
about how to respond to out-of-band removal requests.

We plan to consider these questions
as part of the future, optional extension to the specification
described in the previous section.

### Binary framework distribution

The package registry specification could be amended
to support distributing packages as [XCFramework] bundles.

```http
GET /github.com/mona/LinkedList/1.1.1.xcframework HTTP/1.1
Host: packages.github.com
Accept: application/vnd.swift.registry.v1+xcframework
```

Swift Package Manager could then use XCFramework archives as
[binary dependencies][SE-0272]
or as part of a future binary artifact distribution mechanism.

```swift
let package = Package(
    name: "SomePackage",
    /* ... */
    targets: [
        .binaryTarget(
            name: "LinkedList",
            url: "https://packages.github.com/github.com/mona/LinkedList/1.1.1.xcframework",
            checksum: "ed04a550c2c7537f2a02ab44dd329f9e74f9f4d3e773eb883132e0aa51438b37"
        ),
    ]
)
```

### Package installation from the command-line

Swift Package Manager could be extended with an `install` subcommand
that lets users add a dependency to their package manifest
using its namespace-scoped name.

```terminal
$ swift package install @mona/LinkedList
# Installed LinkedList 1.2.0
```

```diff
+    .package("@mona/LinkedList", .exact("1.2.0"))
```

### Security auditing

The response for listing package releases could be updated to include
information about security advisories.

```jsonc
{
    "releases": { /* ... */ },
    "advisories": [{
        "cve": "CVE-20XX-12345",
        "cwe": "CWE-400",
        "package_name": "@mona/LinkedList",
        "vulnerable_versions": "<=1.0.0",
        "patched_versions": ">1.0.0",
        "severity": "moderate",
        "recommendation": "Update to version 1.0.1 or later.",
        /* additional fields */
    }]
}
```

Swift Package Manager could communicate this information to users
when installing or updating dependencies
or as part of a new `swift package audit` subcommand.

```terminal
$ swift package audit
┌───────────────┬──────────────────────────────────────────────────────────────┐
│ High          │ Regular Expression Denial of Service                         │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Package       │ @mona/RegEx                                                  │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Dependency of │ PatternMatcher                                               │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ Path          │ SomePackage > PatternMatcher > RegEx                         │
├───────────────┼──────────────────────────────────────────────────────────────┤
│ More info     │ https://example.com/advisories/526                           │
└───────────────┴──────────────────────────────────────────────────────────────┘

Found 3 vulnerability (1 low, 1 moderate, 1 high) in 12 scanned packages.
  Run `swift package audit fix` to fix 3 of them.
```

### Package search

The package registry API could be extended to add a search endpoint
to allow users to search for packages by name, keywords, or other criteria.
This endpoint could be used by clients like Swift Package Manager.

```terminal
$ swift package search LinkedList
LinkedList (github.com/mona/LinkedList) - One thing links to another.

$ swift package search --author "Mona Lisa Octocat"
LinkedList (github.com/mona/LinkedList) - One thing links to another.
RegEx (github.com/mona/RegEx) - Expressions on the reg.
```

[BCP 13]: https://tools.ietf.org/html/rfc6838 "Media Type Specifications and Registration Procedures"
[RFC 2119]: https://tools.ietf.org/html/rfc2119 "Key words for use in RFCs to Indicate Requirement Levels"
[RFC 3230]: https://tools.ietf.org/html/rfc5843 "Instance Digests in HTTP"
[RFC 3492]: https://tools.ietf.org/html/rfc3492 "Punycode: A Bootstring encoding of Unicode for Internationalized Domain Names in Applications (IDNA)"
[RFC 3986]: https://tools.ietf.org/html/rfc3986 "Uniform Resource Identifier (URI): Generic Syntax"
[RFC 3987]: https://tools.ietf.org/html/rfc3987 "Internationalized Resource Identifiers (IRIs)"
[RFC 5234]: https://tools.ietf.org/html/rfc5234 "Augmented BNF for Syntax Specifications: ABNF"
[RFC 5843]: https://tools.ietf.org/html/rfc5843 "Additional Hash Algorithms for HTTP Instance Digests"
[RFC 6249]: https://tools.ietf.org/html/rfc6249 "Metalink/HTTP: Mirrors and Hashes"
[RFC 6570]: https://tools.ietf.org/html/rfc6570 "URI Template"
[RFC 6749]: https://tools.ietf.org/html/rfc6749 "The OAuth 2.0 Authorization Framework"
[RFC 7230]: https://tools.ietf.org/html/rfc7230 "Hypertext Transfer Protocol (HTTP/1.1): Message Syntax and Routing"
[RFC 7231]: https://tools.ietf.org/html/rfc7231 "Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content"
[RFC 7233]: https://tools.ietf.org/html/rfc7233 "Hypertext Transfer Protocol (HTTP/1.1): Range Requests"
[RFC 7234]: https://tools.ietf.org/html/rfc7234 "Hypertext Transfer Protocol (HTTP/1.1): Caching"
[RFC 7807]: https://tools.ietf.org/html/rfc7807 "Problem Details for HTTP APIs"
[RFC 8288]: https://tools.ietf.org/html/rfc8288 "Web Linking"
[RFC 8446]: https://tools.ietf.org/html/rfc8446 "The Transport Layer Security (TLS) Protocol Version 1.3"
[UAX15]: http://www.unicode.org/reports/tr15/ "Unicode Technical Report #15: Unicode Normalization Forms"
[UAX18]: http://www.unicode.org/reports/tr18/ "Unicode Technical Report #18: Unicode Regular Expressions"
[UAX31]: http://www.unicode.org/reports/tr31/ "Unicode Technical Report #31: Unicode Identifier and Pattern Syntax"
[UAX36]: http://www.unicode.org/reports/tr36/ "Unicode Technical Report #36: Unicode Security Considerations"
[IANA Link Relations]: https://www.iana.org/assignments/link-relations/link-relations.xhtml
[JSON-LD]: https://w3c.github.io/json-ld-syntax/ "JSON-LD 1.1: A JSON-based Serialization for Linked Data"
[SemVer]: https://semver.org/ "Semantic Versioning"
[Schema.org]: https://schema.org/
[SoftwareSourceCode]: https://schema.org/SoftwareSourceCode
[DUST]: https://doi.org/10.1145/1462148.1462151 "Bar-Yossef, Ziv, et al. Do Not Crawl in the DUST: Different URLs with Similar Text. Association for Computing Machinery, 17 Jan. 2009. January 2009"
[GitHub / Swift Package Management Service]: https://forums.swift.org/t/github-swift-package-management-service/30406
[RubyGems]: https://rubygems.org "RubyGems: The Ruby community’s gem hosting service"
[PyPI]: https://pypi.org "PyPI: The Python Package Index"
[npm]: https://www.npmjs.com "The npm Registry"
[crates.io]: https://crates.io "crates.io: The Rust community’s crate registry"
[CocoaPods]: https://cocoapods.org "A dependency manager for Swift and Objective-C Cocoa projects"
[reverse domain name notation]: https://en.wikipedia.org/wiki/Reverse_domain_name_notation
[UTI]: https://en.wikipedia.org/wiki/Uniform_Type_Identifier
[Maven]: https://maven.apache.org
[Docker Hub]: https://hub.docker.com
[JFrog Artifactory]: https://jfrog.com/artifactory/
[AWS CodeArtifact]: https://aws.amazon.com/codeartifact/
[ICANN]: https://www.icann.org
[thundering herd effect]: https://en.wikipedia.org/wiki/Thundering_herd_problem "Thundering herd problem"
[offline cache]: https://yarnpkg.com/features/offline-cache "Offline Cache | Yarn - Package Manager"
[XCFramework]: https://developer.apple.com/videos/play/wwdc2019/416/ "WWDC 2019 Session 416: Binary Frameworks in Swift"
[SE-0272]: https://github.com/apple/swift-evolution/blob/master/proposals/0272-swiftpm-binary-dependencies.md "Package Manager Binary Dependencies"
[transparent log]: https://research.swtch.com/tlog
[TOFU]: https://en.wikipedia.org/wiki/Trust_on_first_use "Trust on First Use"
[version-specific-tag-selection]: https://github.com/apple/swift-package-manager/blob/master/Documentation/Usage.md#version-specific-tag-selection "Swift Package Manager - Version-specific Tag Selection"
[version-specific-manifest-selection]: https://github.com/apple/swift-package-manager/blob/master/Documentation/Usage.md#version-specific-manifest-selection "Swift Package Manager - Version-specific Manifest Selection"
[STRIDE]: https://en.wikipedia.org/wiki/STRIDE_(security) "STRIDE (security)"
[SE-0219]: https://github.com/apple/swift-evolution/blob/master/proposals/0219-package-manager-dependency-mirroring.md "Package Manager Dependency Mirroring"
[swift-sh]: https://github.com/mxcl/swift-sh
[Beak]: https://github.com/yonaskolb/Beak
[Marathon]: https://github.com/JohnSundell/Marathon
[tspl]: https://docs.swift.org/swift-book/
[tspl-identifiers]: https://docs.swift.org/swift-book/ReferenceManual/LexicalStructure.html#ID412
[scp-url]: https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols#_the_ssh_protocol
[secret scanning]: https://docs.github.com/en/github/administering-a-repository/about-secret-scanning
[typosquatting]: https://en.wikipedia.org/wiki/Typosquatting
[homograph attacks]: https://en.wikipedia.org/wiki/IDN_homograph_attack
[xss]: https://en.wikipedia.org/wiki/Cross-site_scripting
[http header injection]: https://en.wikipedia.org/wiki/HTTP_header_injection
