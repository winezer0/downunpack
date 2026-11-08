# downunpack

Standalone Go module that composes [`downutils`](https://github.com/winezer0/downutils) downloads with optional [`dezip`](https://github.com/winezer0/dezip) extraction. It can be used by applications without importing a parent application's `internal` package.

`Run` downloads one item, then either preserves the file or extracts it. `ProcessFile` applies the same extraction rules to an existing local file without downloading it, for callers that perform their own download and integrity checks. `RunMany` processes requests sequentially and records each runtime failure without stopping later requests.

`ExtractAs` accepts `auto`, `zip`, `tar`, `gzip`, and `skip`; an empty value means `skip`. `tar.gz` and other `tar.*` labels map to `tar`. The legacy value `exe` is unsupported and returns an invalid mode error. The default build extracts ZIP files and preserves ordinary files; TAR and compressed streams require the `dezip_full` build tag.

Applications that need the additional formats must compile with `go build -tags=dezip_full` (or pass the same tag to `go test`/`go run`).

Extraction uses a staging directory and publishes the requested destination only after format and optional `ExpectedFile` validation succeed. The downloaded archive remains unless `RemoveArchiveAfterSuccess` is set. A failed or skipped download never starts extraction.

The download path and extraction destination must not overlap. `Run` rejects a conflicting request before downloading, and `RunMany` prevalidates overlapping paths across the batch before starting any download.

## Staged resource API

`Prepare` selects a remote candidate, downloads and verifies it with `downutils`, optionally extracts it with `dezip`, then returns the staged payload paths. It never replaces an installed file or directory.

```go
result, err := downloadunpack.Prepare(ctx, downloadunpack.ResourceRequest{
	Sources: []downloadunpack.Source{{
		Name:     "netgap_windows_amd64_v0.1.0.zip",
		URL:      assetURL,
		Checksum: "sha256:" + checksumURL,
		Size:     assetSize,
	}},
	SourcePattern:  "netgap_windows_amd64_*.zip",
	ExtractAs:      "auto",
	PayloadPattern: "netgapgui.exe",
})
if err != nil {
	return err
}
defer result.Cleanup()
// Use result.PayloadPath as the staged file or directory.
```

`SourcePattern` matches asset names supplied by the caller, such as Release API results. When omitted, exactly one source must be supplied. A pattern that matches zero or multiple candidates fails. `Source.Size` enables exact size checking; `MaxDownloadBytes` limits the transfer. `Checksum` is passed to downutils, so checksum URLs use the same HTTP client and proxy as the resource download.

`ExtractAs` defaults to `auto`; `skip` returns the downloaded file directly and does not allow payload selectors. In the default build, `auto` extracts ZIP files and preserves ordinary files. With the `dezip_full` build tag, it also extracts the additional formats supported by dezip. `PayloadPattern` matches a relative path inside the extraction root and may select a regular file or directory. An empty pattern or `.` selects the full extraction root. `EntrypointPattern` optionally selects exactly one regular file relative to the selected payload. Neither selector guesses which executable is the main program.

Patterns use `*`, `?`, and character classes within path components; `**` matches zero or more directories. Matching is case-sensitive and paths use `/` on every platform. Selectors must resolve to exactly one path. Symlinks are not followed during matching.

On success, `StageDir`, `PayloadPath`, and optional `EntryPoint` point into a private staging directory. The caller owns that directory and must call `Cleanup` after consuming the payload. Any failed download, checksum, extraction, or selector match cleans the staging directory. If `HTTPClient` is provided, it is reused for the download and checksum request; otherwise the package builds one from `DownloadOptions`, including its proxy settings.
