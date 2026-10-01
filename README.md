# grpc-csharp-ext-universal

A **universal macOS** (`x86_64` + `arm64`) build of `libgrpc_csharp_ext.dylib`, the
native library behind the NuGet package [`Grpc.Core`](https://www.nuget.org/packages/Grpc.Core/2.46.6)
2.46.6.

The NuGet package only ships `runtimes/osx-x64/native/libgrpc_csharp_ext.x64.dylib`,
a pure `x86_64` binary. Processes running natively on Apple Silicon (for example
RimWorld on an M1–M4 Mac) cannot load it. This repository builds the same library
for both architectures from the official source and merges them with `lipo`.

It exists for the RimWorld mod
[RimionshipSeedOfTheMonth](https://github.com/Rimionship/RimionshipSeedOfTheMonth).

## Source

- Repository: [`grpc/grpc`](https://github.com/grpc/grpc), tag **`v1.46.6`**
  (commit `18dda3c586b2607d8daead6b97922e59d867bb7d`, checked by the workflow),
  with all submodules (BoringSSL, Abseil, c-ares, re2, zlib, ... are linked in
  statically).
- Why this tag: `grpc.core.nuspec` of `Grpc.Core` 2.46.6 records
  `<repository commit="cdc97245c7977e85bb0234941f5c6cd9cb7accdf">`. `v1.46.6` is the
  direct child of that commit and only changes two scripts under
  `tools/internal_ci/`; `src/csharp/build/dependencies.props` at the tag says
  `GrpcCsharpVersion 2.46.6`. The library reports gRPC core version `24.0.0`.

## Build

[`.github/workflows/build.yml`](.github/workflows/build.yml), everything in GitHub
Actions (actions pinned to commit SHAs):

1. One native job per architecture: `macos-15` (arm64) and `macos-15-intel` (x86_64).
   CMake, `Release`, `-DgRPC_BUILD_CSHARP_EXT=ON`, target `grpc_csharp_ext`,
   same flags as upstream's `tools/run_tests/artifacts/build_artifact_csharp.sh`
   (`gRPC_BACKWARDS_COMPATIBILITY_MODE=ON`, `gRPC_XDS_USER_AGENT_IS_CSHARP=ON`).
2. Deployment target: **arm64 11.0** (the first macOS on Apple Silicon, so no
   real restriction) and **x86_64 10.13** (the oldest target current Xcode
   accepts; Intel Macs on 10.13–10.15 keep working, as they did with the NuGet
   build, which targeted 10.10).
3. `lipo -create` to `libgrpc_csharp_ext.dylib`; the log shows `lipo -info` and
   `otool -L`, and the job fails if anything other than system libraries
   (`/usr/lib/…`, `/System/Library/…`) is linked.
4. Smoke test: the universal file is loaded with `ctypes` natively on arm64 and on
   x86_64, `grpcsharp_init()` / `grpcsharp_version_string()` are called.
5. SHA256, build provenance attestation (`actions/attest-build-provenance`), and on
   tags `v*` a GitHub release with `libgrpc_csharp_ext.dylib` and
   `libgrpc_csharp_ext.dylib.sha256`.

## Verify a downloaded file

```sh
gh attestation verify libgrpc_csharp_ext.dylib --repo Rimionship/grpc-csharp-ext-universal
shasum -a 256 -c libgrpc_csharp_ext.dylib.sha256
lipo -info libgrpc_csharp_ext.dylib   # macOS: x86_64 arm64
```

`gh attestation verify` proves the file was produced by this repository's
workflow (it shows the commit and run); the workflow shows which `grpc/grpc`
commit it was built from.

## License

gRPC is licensed under the Apache License 2.0; see [LICENSE](LICENSE) (copied from
`grpc/grpc` at `v1.46.6`). The bundled third-party libraries keep their own licenses
(see the `third_party/` directory of the gRPC source).
