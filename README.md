# llm registry

Package index for [llm](https://github.com/llm-build-system/llm). The registry
maps package versions to source archives and their SHA-256 checksums. Each
archive contains a `.llm` project that llm builds for the selected target.

## Using packages

Declare the registry in your project's `.llm` file:

```toml
[registries]
official = "https://github.com/llm-build-system/llm-registry"
```

Add a dependency and build:

```sh
llm add official:bzip2@1.0.8
llm build --release
```

Dependencies can also be declared directly:

```toml
[dependencies]
private = ["official:bzip2@1.0.8"]
```

llm downloads and verifies the selected source archive. The project's `.lock`
records the exact version and checksum. See the
[llm README](https://github.com/llm-build-system/llm#libraries-and-dependencies)
for version requirements, updates and authentication.

## Index entries

Each file in [packages/](packages/) is named after its package, without an
extension. Each line declares one version:

```text
<archive-url>@<version>:<sha256>
```

For example, `packages/bzip2` contains:

```text
https://github.com/llm-build-system/bzip2/releases/download/v1.0.8/bzip2-1.0.8.tar.gz@1.0.8:3bbb0a9ffdf23771b392d7cd7774c54eb5497c9ce4a04a1bcccac766f63080a4
```

Package versions follow their upstream versions. Updates to a port's build
integration retain that upstream version. When an archive changes, its checksum
in the index is updated to match. Run `llm update` to refresh an existing
project's lockfile. Index files contain no headers or additional metadata.

A normal build continues to use the exact archive hash in its lockfile while a
valid copy is available in the local cache. If that archive is no longer
available locally and the registry supplies a different hash, the build stops
instead of silently adopting the replacement. `llm update` selects the current
archives within the project's declared version requirements and refreshes the
lockfile. Replacing an archive does not revoke valid older cached copies.

## Create your own registry

A registry is a directory of package index files. It can be hosted on your own
HTTP or HTTPS server, stored locally or published in a GitHub repository.
Package archives can be hosted separately from the index.

```text
my-registry/
└── packages/
    └── bzip2
```

The index only needs `packages/` and its package files. A registry does not need
its own `.llm` file. Store one version per line, using the format described in
[Index entries](#index-entries). Do not add blank lines, comments or duplicate
versions to a package file.

To try it with an existing package, copy the complete bzip2 entry shown above
into `packages/bzip2`. That entry can continue to point to the existing public
archive. Choose one of the hosting options below, then declare the registry in
a consumer project's `.llm`:

```toml
type = "executable"

[registries]
custom = "https://packages.example.com/my-registry"
```

Replace the example URL with your registry address, then add the package and
build:

```sh
llm add custom:bzip2@1.0.8
llm build --release
```

### Choose where to host the index

**Your own HTTP or HTTPS server:** llm requests
`<registry-url>/packages/<package-name>`. Every package response must carry the
same strong, quoted `ETag` identifying the complete index revision. The server
must honor subsequent `If-Match` requests and reject an outdated revision.
Ordinary per-file ETags are insufficient. Publish index changes consistently
as a new revision. Serving the files without these snapshot guarantees is not
sufficient. HTTPS is recommended.

**A local directory:** use an absolute URL such as
`file:///path/to/my-registry` or `file:///C:/path/to/my-registry` on Windows.
No server or Git repository is required. A local registry may reference archives
through absolute `file://`, HTTPS or HTTP URLs. A remote registry cannot
reference local file archives.

**A GitHub repository:** use its repository URL, for example
`https://github.com/YOUR-ACCOUNT/my-registry`. Put `packages/` on the default
branch. llm reads the index at a single commit through GitHub's API, so no
separate HTTP server configuration is needed.

For an authenticated HTTPS registry, run `llm login custom` before adding the
package. The server must accept Bearer token authentication. For GitHub, enter
a token with read access to both the index and any private package repositories.
Never put tokens in index entries or commit them to source control. HTTP and
local file registries do not support token authentication.

### Publish your own package

Prepare a library project with a `.llm` file, its sources and public headers.
A compiled library typically declares `type = "static-library"`. A header-only
library declares `type = "header-only-library"`. See the
[project declarations](https://github.com/llm-build-system/llm#the-llm-file)
for public includes, definitions, modules and dependencies.

Create a `.tar.gz` archive containing the project at the archive root or inside
one enclosing directory. Include its `.llm`, all required build inputs and the
applicable original license texts and notices. Leave out build outputs,
credentials and local files. Dependencies must use registry references or
paths contained within the archive, not paths to neighboring repositories.

Upload the archive, for example as a GitHub release asset. Calculate SHA-256
from the final archive bytes:

```powershell
Get-FileHash ./my-library-1.0.0.tar.gz -Algorithm SHA256
```

On Linux use `sha256sum my-library-1.0.0.tar.gz`. On macOS use
`shasum -a 256 my-library-1.0.0.tar.gz`.

Create `packages/my-library` with one line in this format, replacing every
placeholder with the actual value:

```text
<archive-url>@1.0.0:<64-character-sha256>
```

The package version is `1.0.0`. A Git tag such as `v1.0.0` is part of the
archive URL and does not change that version. Add future versions as separate
lines in the same file. If an existing archive is replaced, update its checksum
and follow the lockfile behavior described in [Index entries](#index-entries).

## Submit a package to this registry

Submit a pull request to this repository to add a package or a new version.
Add `packages/<package-name>` or append a version to an existing package file,
using the index format above. Each entry must point to a downloadable llm
package archive and include its exact version and SHA-256 checksum.

The registry indexes packages. It does not take over their repositories or
hosting. Third-party packages remain in their maintainers' repositories and
release locations. Only the index reference is added here. Listing a package
does not mean its repository will be added to the llm-build-system organization.
The organization hosts only selected packages relevant to its own needs or
maintenance responsibilities.

In the pull request, include:

- A short description and links to the upstream project and the llm package
  repository.
- The package license and any required notices.
- The platforms and build configurations you actually tested.
- An explicit explanation for any change to an existing archive URL or checksum.

Package archives must include a working `.llm` project, required build inputs
and applicable license texts and notices. Preserve upstream attribution and
identify independent ports clearly. Do not include credentials or private data.

Submissions are reviewed before inclusion. Registry inclusion is not a security
guarantee, as explained below. Report vulnerabilities through the private
security reporting process rather than a public pull request.

## Package safety and responsibility

Use of this registry and its packages is voluntary. Inclusion in the registry
does not constitute a security review, audit, endorsement or guarantee of
safety. Users are responsible for evaluating the origin, suitability and
security of the packages they choose and for keeping their dependencies updated.
A matching SHA-256 checksum verifies archive contents. It does not establish
that the code is safe.

Packages are provided "as is", without warranties. To the extent permitted by
applicable law, the llm maintainers disclaim liability for damages arising from
use of the registry or listed packages. Each package remains subject to its own
license terms and copyright notices.

## Security reports

Report vulnerabilities privately through
[GitHub](https://github.com/llm-build-system/llm-registry/security/advisories/new).
Do not post security-sensitive details in public issues or discussions. See the
[security reporting policy](https://github.com/llm-build-system/.github/blob/main/SECURITY.md).

For general contact and project updates, use [@3754Rx on X](https://x.com/3754Rx).

## Upstream projects

The package repositories maintain llm build integration for their upstream
projects. Those libraries remain the work of their original authors. Each
package links to its original project and authors and preserves its license
and copyright notices. The llm ports are not official upstream releases.
