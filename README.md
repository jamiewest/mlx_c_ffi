# mlx_ffi

Dart FFI bindings for [`mlx-c`](https://github.com/ml-explore/mlx-c), generated with [`ffigen`](https://pub.dev/packages/ffigen).

## What is included

- `lib/src/generated/mlx_bindings.dart`: auto-generated low-level bindings (full `mlx_*` symbol surface from `mlx/c/*.h`).
- `lib/mlx_ffi.dart`: high-level API (`Mlx`, `MlxArray`) including dynamic library loading.
- `tool/ffigen_include/`: parse-only header overlay so `ffigen` can generate APIs that use C complex/fp16/bf16 types.
- `third_party/mlx-c`: the upstream `mlx-c` source as a git submodule, used for building native libraries and regenerating bindings.

## Building from source

The submodule must be initialised before building:

```bash
git submodule update --init --recursive
```

### Build shared macOS library

```bash
cmake -S third_party/mlx-c -B build/mlx-shared \
  -DBUILD_SHARED_LIBS=ON \
  -DMLX_C_BUILD_EXAMPLES=OFF \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build/mlx-shared --config Release --parallel
```

Output: `build/mlx-shared/libmlxc.dylib` (and `libmlx.dylib` as a transitive dependency).

### Build Apple XCFramework

Builds a multi-slice XCFramework (macOS + iOS + iOS simulator):

```bash
dart run tool/build_apple_xcframework.dart
```

Common variants:

```bash
# Only macOS slice
dart run tool/build_apple_xcframework.dart --platforms macos

# Reuse existing build tree and set deployment targets
dart run tool/build_apple_xcframework.dart --no-clean --ios-min 17.0 --macos-min 14.0
```

Output default: `build/apple/MLXC.xcframework`

## Runtime library

The high-level wrapper expects `mlx-c` to be available as a dynamic library at runtime:

| Platform | Default name    |
|----------|-----------------|
| macOS    | `libmlxc.dylib` |
| Linux    | `libmlxc.so`    |
| Windows  | `mlxc.dll`      |

You can also provide an explicit path:

```dart
final mlx = Mlx.open(libraryPath: '/path/to/libmlxc.dylib');
```

## Generate bindings

Run after updating the `mlx-c` submodule to regenerate `mlx_bindings.dart`:

```bash
dart pub get
dart run ffigen --config ffigen.yaml
```

Validate symbol coverage:

```bash
./tool/verify_symbol_coverage.sh
```

## CLI example

```bash
dart run bin/mlx_cli_example.dart --library /absolute/path/to/libmlxc.dylib
```

Optional GPU stream:

```bash
dart run bin/mlx_cli_example.dart --gpu --library /absolute/path/to/libmlxc.dylib
```

## Automated GitHub Releases

`.github/workflows/release-on-build-change.yml` runs on pushes of `v*` tags or manual dispatch. It:

1. runs `dart test` to gate on a passing test suite,
2. builds shared macOS libraries (`libmlxc.dylib`, plus `libmlx.dylib` when present),
3. builds and zips `MLXC.xcframework` (macOS + iOS + iOS simulator slices),
4. computes a build fingerprint from compiled artifacts, and
5. creates a new GitHub Release only when that fingerprint differs from the latest release.

Release assets:

- `MLXC.xcframework.zip`
- `mlx-macos-dylibs.tar.gz`
- `mlx_ffi-source.zip`
- `build-fingerprint.txt`
- `build-manifest.json`
