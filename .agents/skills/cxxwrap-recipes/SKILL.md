---
name: cxxwrap-recipes
description: Rules for Yggdrasil recipes built against Julia's headers. Use when a build_tarballs.jl depends on libcxxwrap_julia_jll or libjulia_jll, includes L/libjulia/common.jl or calls libjulia_platforms, uses the julia_version platform tag, or when adding a new Julia version to libjulia.
---

# CxxWrap and libjulia packages

[CxxWrap.jl](https://github.com/JuliaInterop/CxxWrap.jl) wrappers, and anything else built against Julia's headers, are tied to a Julia minor version. `libjulia_platforms(v)` tags each platform with `julia_version=v`. Julia's host platform carries the same tag and matches it on major.minor, so no augmentation block is needed.

Standard recipe shape (see `L/libcxxwrap_julia/build_tarballs.jl`):

```julia
include("../../L/libjulia/common.jl")  # also loads macos_sdks.jl
# ...
script = raw"""
cmake -S package -B build \
    -DJulia_PREFIX=${prefix} \
    -DCMAKE_FIND_ROOT_PATH=${prefix} \
    -DCMAKE_INSTALL_PREFIX=${prefix} \
    -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TARGET_TOOLCHAIN} \
    -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel ${nproc}
cmake --install build
install_license package/LICENSE
"""

platforms = vcat(libjulia_platforms.(julia_versions)...)
platforms = expand_cxxstring_abis(platforms)

dependencies = [
    BuildDependency(PackageSpec(; name="libjulia_jll", version="1.11.4")),
    Dependency("libcxxwrap_julia_jll"; compat="0.14.12"),
]

build_tarballs(ARGS, name, version, sources, script, platforms, products, dependencies;
    julia_compat=libjulia_julia_compat(julia_versions),
    preferred_gcc_version=v"10")
```

- **`libjulia_jll`** is always a `BuildDependency`: the running Julia supplies `libjulia`. Pin it to `jllversion` from `L/libjulia/build_tarballs.jl`. Older releases lack artifacts for newer Julia versions; the history comment in `common.jl` lists the first release for each.
- **`julia_compat`**: use `libjulia_julia_compat(julia_versions)`, which yields `~1.10, ~1.11, …` and admits only Julia versions that have an artifact. A plain `"1.10"` admits future Julia versions the JLL was not built for.
- **Julia versions**: `julia_versions` comes from `common.jl` with prerelease tags stripped, so `1.15.0-DEV` appears as `v"1.15.0"`. To drop some, filter it right after the include, before computing platforms and `julia_compat`, e.g. `filter!(>=(v"1.11"), julia_versions)`. Filter platforms after `libjulia_platforms`, then call `expand_cxxstring_abis`.
- **Versioning**: when `common.jl` adds or drops a Julia version, an unfiltered recipe's `julia_compat` changes on its next build. That rebuild needs a new JLL version, not a `# Build trigger` bump (see [Dependencies](../../../AGENTS.md#dependencies)). Without a new upstream release, keep `version` and raise `ygg_version` (see [`jll-versioning`](../jll-versioning/SKILL.md)). `@assert libjulia_min_julia_version <= v"1.10.0"` makes dropping the oldest Julia version fail the build instead of silently narrowing `julia_compat`.
- **`libcxxwrap_julia_jll` compat**: before 1.0, each minor release (0.13 → 0.14) breaks ABI. Use the minor that the Julia package's CxxWrap.jl requires (CxxWrap.jl 0.17 needs libcxxwrap_julia_jll 0.14). BinaryBuilder builds against the oldest version in the bound, so the lower bound must be a release built for every Julia version in `julia_versions`.
- **GCC**: the exported `JlCxx::cxxwrap_julia` CMake target requires C++20 (`cxx_std_20`), and `libcxxwrap_julia_jll` is built with GCC 10, so use GCC 10 or newer; see [GCC Version Selection](../../../AGENTS.md#gcc-version-selection).
- **Local builds**: the triplet carries the tag, e.g. `x86_64-linux-gnu-cxx11-julia_version+1.12.0`.

Supporting a new Julia version takes three PRs, each after the previous JLL is registered:

1. `L/libjulia`: in `common.jl`, add the version to `julia_full_versions` with a `checksums` entry (a `GitSource` branch for `-DEV`), add a `version.minor` branch for its `LLVM_full_jll` and other build dependencies in `build_julia`, and update the history comment. Put any patches in `bundled/patches/<version>/` and bump `jllversion` in `build_tarballs.jl`.
2. `L/libcxxwrap_julia`: raise the `libjulia_jll` pin and give the JLL a new version (see Versioning above).
3. Each wrapper: the same, and raise the `libcxxwrap_julia_jll` lower bound.
