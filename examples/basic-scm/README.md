# Basic SCM Example - myadder_scm

This example extends the [basic](../basic) example with **dynamic versioning**
via `setuptools-scm`. Instead of a hard-coded `version` in `pyproject.toml`,
the version is resolved automatically from Git tags.

## Key differences from `basic`

| | `basic` | `basic-scm` |
|--|---------|-------------|
| Version source | `version = "0.1.0"` (static) | `dynamic = ["version"]` + Git tags |
| Build requires | `conan-py-build` | `conan-py-build` + `setuptools-scm` |
| `__init__.py` | `__version__ = "0.1.0"` | Imports from `_version.py` with `"0.0.0"` fallback |
| SCM config | (none) | `[tool.setuptools_scm]` |

## pyproject.toml highlights

```toml
[build-system]
requires = ["conan-py-build", "setuptools-scm"]
build-backend = "conan_py_build.build"

[project]
name = "myadder_scm"
dynamic = ["version"]            # no static version key

[tool.conan-py-build]
conanfile-path = "conan"

[tool.conan-py-build.version]
provider = "setuptools_scm"       # tells the backend to use setuptools-scm

[tool.setuptools_scm]
version_scheme = "guess-next-dev"
local_scheme = "no-local-version"
fallback_version = "0.0.0"
root = "."
version_file = "src/myadder_scm/_version.py"
```

The `[tool.conan-py-build.version]` section tells the backend which version
provider to use. The `setuptools-scm` options are configured in
`[tool.setuptools_scm]` — see the
[setuptools-scm docs](https://setuptools-scm.readthedocs.io/) for all
available options.

## Build and Install

```bash
# From the repo root, install the backend first
pip install -e .

# Create a git tag so setuptools-scm can resolve a version
cd examples/basic-scm
git tag v1.0.0

# Build the wheel
pip wheel . --no-build-isolation -w dist/ -v

# Or install directly
pip install . --no-build-isolation -v
```

### Test the wheel

```bash
pip install dist/myadder_scm-*.whl

python -c "import myadder_scm; print(myadder_scm.__version__); print(myadder_scm.add(2, 3))"

pip uninstall myadder_scm -y
```
