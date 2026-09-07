# go-plugin protos - vendored, pinned

Upstream: https://github.com/hashicorp/go-plugin
Tag:      v1.8.0
Commit:   155dcddc94873a285e14b7fa24b2f6ab6139668e
Fetched:  2026-08-23
License:  MPL-2.0 (same as TSIGMA)

Files are byte-for-byte upstream copies of `internal/plugin/*.proto`, not forks.
PROTOCOL.md freezes these as TSIGMA Plugin Protocol v1; this directory is the
single pinned copy every host and third-party plugin author generates from.

Re-fetch:
  for f in grpc_broker grpc_controller grpc_stdio; do
    curl -sSL -o "$f.proto" \
      "https://raw.githubusercontent.com/hashicorp/go-plugin/155dcddc94873a285e14b7fa24b2f6ab6139668e/internal/plugin/$f.proto"
  done

grpc.health.v1 is NOT vendored - it is a gRPC standard, consumed from whatever
health-checking package the implementer's own gRPC stack ships.
