# osv-scalibr: an old-format .csproj fails the whole scan

`Legacy.csproj` uses the Visual Studio 2003 (pre-MSBuild) project format, with `<VisualStudioProject>` as the root element.
The `dotnet/csproj` extractor rejects it with: `expected element type <Project> but have <VisualStudioProject>`.

| Folder | Contents | `dotnet/csproj` status | Scan status | Exit code |
|---|---|---|---|---|
| `passes/` | old `Legacy.csproj` + modern `Modern.csproj` (Newtonsoft.Json 12.0.1) | PARTIALLY_SUCCEEDED | SUCCEEDED | 0 |
| `fails/`  | old `Legacy.csproj` + `packages.config` (Newtonsoft.Json 12.0.1) | FAILED | PARTIALLY_SUCCEEDED | **1** |

In both cases the SPDX output is written and includes Newtonsoft.Json.

Why: in `plugin.StatusFromErr`, a plugin that hits file errors is marked `FAILED` when it extracted
**no** packages, and `PARTIALLY_SUCCEEDED` when it extracted at least one
(`extractor/filesystem/filesystem.go` `errToExtractorStatus`, `foundInv`). If any plugin is `FAILED`,
the overall scan becomes `PARTIALLY_SUCCEEDED` (`scalibr.go`). For anything other than `SUCCEEDED`, the
command-line tool exits 1 (`binary/scanrunner/scanrunner.go`).

## Reproduce

```sh
docker run --rm -e GOTOOLCHAIN=auto -v "$PWD":/src:ro golang:1.25 sh -c '
  go install github.com/google/osv-scalibr/binary/scalibr@latest &&
  for d in passes fails; do
    /go/bin/scalibr --root=/ --plugins=dotnet/csproj,dotnet/packagesconfig \
      -o spdx23-json=/tmp/$d.json /src/$d; echo "$d exit=$?"
  done'
```

Tested with osv-scalibr v0.5.3.

## osv-scanner on `fails/`

`ghcr.io/google/osv-scanner:latest` reports GHSA-5crp-9r3c-p9vr from `packages.config` and logs no
error for `Legacy.csproj`. Its exit code is 1 because a vulnerability was found, not because of an error.
