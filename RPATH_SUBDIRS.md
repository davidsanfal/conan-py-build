# Proposal: Auto-detect shared library subdirectories for RPATH

## Problem

Currently `patch_rpath` adds only `@loader_path` (macOS) / `$ORIGIN` (Linux)
to each Python extension module. This lets the extension find shared libraries
**in its own directory**, but not in subdirectories like `lib/`.

A common wheel layout after `install-dir` + `cmake install` looks like:

```
pymandos/
├── mandos_pylib.abi3.so      ← extension module
├── __init__.py
├── lib/
│   └── libCore.dylib          ← project shared lib (cmake install DESTINATION lib)
├── libembree4.dylib           ← runtime_deploy (same dir, found via @loader_path)
└── libtbb.dylib
```

The extension has `@loader_path` in its RPATH, so it finds `libembree4.dylib`
(same dir). But it **cannot** find `lib/libCore.dylib` unless the project's
CMake explicitly sets `INSTALL_RPATH "@loader_path/lib"` with
platform-conditional logic.

This forces every project to write:

```cmake
if(APPLE)
  set_target_properties(myext PROPERTIES INSTALL_RPATH "@loader_path/lib")
else()
  set_target_properties(myext PROPERTIES INSTALL_RPATH "$ORIGIN/lib")
endif()
```

## Proposed change

Enhance `patch_rpath` in `wheel_deploy.py` to scan for subdirectories
containing shared libraries relative to each extension module, and add them
as additional RPATH entries automatically.

### Current behavior

```python
def patch_rpath(staging_dir: Path) -> None:
    # Adds only @loader_path / $ORIGIN
    for path in staging_dir.rglob("*.so"):
        if _is_python_extension_module(path):
            add_rpath(path, "@loader_path")  # or $ORIGIN on Linux
```

### Proposed behavior

```python
def patch_rpath(staging_dir: Path) -> None:
    if sys.platform == "darwin":
        base_rpath = "@loader_path"
        ...
    elif sys.platform == "linux":
        base_rpath = "$ORIGIN"
        ...

    for path in staging_dir.rglob("*.so"):
        if not _is_python_extension_module(path):
            continue

        rpaths = {base_rpath}

        # Scan for subdirectories with shared libraries relative to the extension
        ext_dir = path.parent
        lib_globs = ("*.so", "*.so.*", "*.dylib") if sys.platform == "darwin" else ("*.so", "*.so.*")
        for child in ext_dir.rglob("*"):
            if child.parent == ext_dir:
                continue  # skip same-dir libs (already covered by base_rpath)
            if any(child.name.endswith(g.lstrip("*")) for g in lib_globs) or child.suffix in (".so", ".dylib"):
                rel = child.parent.relative_to(ext_dir)
                rpaths.add(f"{base_rpath}/{rel}")

        for rpath in rpaths:
            _add_rpath(path, rpath)
```

### Example result

For the layout above, `mandos_pylib.abi3.so` would get:

| Platform | RPATH entries added |
|---|---|
| macOS | `@loader_path`, `@loader_path/lib` |
| Linux | `$ORIGIN`, `$ORIGIN/lib` |

## Trade-offs

| Pro | Con |
|---|---|
| Projects don't need platform-conditional RPATH in CMake | Adds RPATH entries the project might not need (harmless) |
| Works automatically with any `install-dir` + subdirectory layout | Slightly more filesystem scanning during wheel build |
| Consistent behavior across platforms without project awareness | Projects that already set correct RPATH get duplicate (benign) entries |

## Scope

- Only affects extension modules (`.so` / `.pyd`) detected by
  `_is_python_extension_module`.
- Only scans within the extension's parent directory (the package dir), not the
  entire staging area.
- Duplicate RPATH entries are silently ignored by both `install_name_tool` and
  `patchelf`.
- No behavior change for projects that keep all shared libs in the same
  directory as the extension (the current common case).
