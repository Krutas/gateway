# GitHub Actions CI Plan

## Workflow: `ci.yml`

Runs on every push and PR to `main`.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -e ".[dev]"
      - name: Run tests
        run: pytest -q
      - name: Check packaging
        run: python -m build
```

## Workflow: `release.yml`

Runs on version tags `v*`.

```yaml
name: Release

on:
  push:
    tags:
      - "v*"

jobs:
  pypi:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install build twine
      - run: python -m build
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          password: ${{ secrets.PYPI_API_TOKEN }}
```

## Pre-Merge Checklist

- [ ] `pytest -q` passes on Python 3.11 and 3.12.
- [ ] New code has tests.
- [ ] README updated if user-facing behavior changed.
- [ ] Version bumped in `pyproject.toml` if releasing.
