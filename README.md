# Job Cache Action

A GitHub action for detecting whether a job has already run for the checked out files.

This can be used to avoid re-running tests you have already run, speeding up delivery. In particular, the following cases are supported:

- Skipping tests on a revert commit
- Skipping tests on a (clean) squash-merge
- Skipping tests which only depend on a subset of files (e.g. different test types, or sub-projects of a mono-repo)

## Usage

### Basic usage

The following example shows a simple testing job, which skips testing if tests have previously passed on an identical commit. This is useful for reverts, squashes etc.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@v3
      - id: job-cache
        name: Check job cache
        uses: jacksmith15/job-cache-action@v1
      - id: test
        name: Run tests
        if: steps.job-cache.passed == 'false'
        run: |
          make test
```

> :information_source: The `GH_TOKEN` environment variable is necessary for the action to check for cache hits by default. If you'd prefer to avoid using a token, you can still use [branch-local caching](#branch-local-caching).

### Filtering input files

You can specify a reduced set of files to detect changes in to improve the cache hit rate. This allows skipping tests for a commit which doesn't affect the tests, e.g. documentation. In the following example, tests are skipped if they previously passed for a commit with identical Python files.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@v3
      - id: job-cache
        name: Check job cache
        uses: jacksmith15/job-cache-action@v1
        with:
          paths: '*.py pytest.ini .github'
      - id: test
        name: Run tests
        if: steps.job-cache.passed == 'false'
        run: |
          pytest
```

> :warning: It's usually a good idea to include the `.github` directory, as this defines the workflow, and so is likely to affect whether the job passes or fails.

> :information_source: The paths follow the same rules as [git pathspec](https://git-scm.com/docs/gitglossary#Documentation/gitglossary.txt-aiddefpathspecapathspec). See that link for information on globbing, excludes and more. 

### Multiple caches in the same repository

You can namespace the cache keys using the `prefix` option. This allows multiple distinct caches in the same repository. This is useful in a monorepo or if you want to cache different types of job separately. In the following example, see how different jobs use different cache key prefixes to partition the cache:

```yaml
jobs:
  typecheck:
    runs-on: ubuntu-latest
    env:
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@v3
      - id: job-cache
        name: Check job cache
        uses: jacksmith15/job-cache-action@v1
        with:
          prefix: typecheck/
          paths: '*.py *.pyi mypy.ini .github'
      - id: test
        name: Run type-checker
        if: steps.job-cache.passed == 'false'
        run: |
          mypy
  test:
    runs-on: ubuntu-latest
    steps:
      - id: job-cache
        name: Check job cache
        uses: jacksmith15/job-cache-action@v1
        with:
          prefix: pytest/
          paths: '*.py pytest.ini .github'
      - id: test
        name: Run tests
        if: steps.job-cache.passed == 'false'
        run: |
          pytest

```

> :memo: Note how the type-checker job uses different paths for the job cache. This means that a change in a `.pyi` file (type stubs), will re-run the type-checker but not the unit tests.


### Branch-local caching

By default the cache is shared between workflows from all branches. This allows tests to be skipped on `main` if they've already passed on a PR before merging (provided the merge was clean for the relevant files).

If you'd prefer to re-run tests on `main`, you can set `global: 'false'` on the action. In this mode, workflows on branches will only detect previous passes from the current branch, the default branch, and the base branch if the workflow is running from a PR (following the [same rules as regular Actions Cache usage](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache)).

For example:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - id: job-cache
        name: Check job cache
        uses: jacksmith15/job-cache-action@v1
        with:
          global: 'false'
      - id: test
        name: Run tests
        if: steps.job-cache.passed == 'false'
        run: |
          make test
```

Using the action in local mode also allows the action to be used without a GitHub token configured.

## How does it work?

Under-the-hood this is storing empty files in the [GitHub Actions Cache](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) when a job completes successfully. The key is computed as a SHA-256 of all the specified files.

[Default restrictions on cross-branch cache access](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache) are bypassed by querying the cache via the (index-only) REST API instead, since we only care about whether the key exists.

GitHub Actions Cache usage limits are based on total size (10 GB) and there is no limit on the number of keys. Because this action doesn't store any data (just a blank placeholder file) it is unlikely to exceed that quota.

Entries in the GitHub Actions Cache have a default TTL of 7 days, and you can manually remove individual cache entries via the UI and the API.
