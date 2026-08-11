# Shared Workflows
This repository contains Github workflows that can be reused in other repositories to avoid duplication.

## How to use shared workflows
To use a workflow from this repository in another repository workflow, you need to use the following syntax:

```
jobs:
  job_id:
     uses: wp-media/workflows/.github/workflows/name.yml@ref
```

Replace `name` by the workflow filename, and `ref` can be either a branch name, a tag or a SHA commit.

## Currently available workflows
- `phpcs.yml`: A workflow to run PHP CodeSniffer
- `phpstan.yml`: A workflow to run PHPStan

## Composite actions for PHPUnit tests
Running a PHPUnit unit/integration suite is **not** shipped as a reusable workflow. A reusable workflow is a whole job: the caller cannot add its own steps to it, and secrets only reach the test process if the workflow itself maps them to `env`. That makes it a poor fit for tests, where each repository needs to keep its own secrets and often splits the run across several steps.

Instead, the duplicated *setup* is shipped as two composite actions. The consuming repository keeps ownership of its job — the `services:`, the `env:`/secrets, and the test step(s) — while the heavy setup collapses into two `uses:` lines.

- `.github/actions/setup-php-composer`: sets up PHP (with the PHP and PHPUnit problem matchers) and installs Composer dependencies, with an optional extra dev requirement.
- `.github/actions/setup-wp-tests`: prepares the WordPress PHPUnit test suite — caches/checks out the `develop.svn` test suite (installing SVN only on a cache miss), applies the MySQL 8 `mysql_native_password` auth workaround, and runs the **bundled** `install-wp-tests.sh`. Consuming repositories no longer need their own copy of the script in `bin/`.

Replace `ref` below with a branch name, a tag or a commit SHA.

### `setup-php-composer` inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `php-version` | yes | — | PHP version to set up. |
| `coverage` | no | `none` | Coverage driver for `setup-php`: `xdebug`, `pcov` or `none`. |
| `tools` | no | `composer:v2` | Tools to install via `setup-php`. |
| `ini-values` | no | `''` | `php.ini` values to set (e.g. `memory_limit=2G`). |
| `composer-options` | no | `--no-scripts` | Options for the Composer install step. |
| `extra-require` | no | `''` | Extra dev package(s) to `composer require --dev` after install, e.g. `wpackagist-plugin/woocommerce "^7"` or `phpunit/phpcov`. |

### `setup-wp-tests` inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `wp-version` | yes | — | WordPress version passed to `install-wp-tests.sh` (e.g. `latest`, `5.9`). |
| `db-name` | no | `wordpress_test` | Test database name. |
| `db-user` | no | `root` | Database user. |
| `db-pass` | no | `root` | Database password. |
| `db-host` | no | `127.0.0.1:3306` | Database host (`host:port`). |
| `wp-tests-dir` | no | `/tmp/tests/phpunit` | Directory the WP test library is installed into (exported as `WP_TESTS_DIR`). |
| `wp-core-dir` | no | `/tmp/wordpress-develop` | Directory WordPress core is installed into (exported as `WP_CORE_DIR`). |
| `configure-mysql-auth` | no | `true` | Apply the MySQL 8 `mysql_native_password` workaround before installing. |
| `cache-version` | no | `1` | Bump to manually invalidate the cached WP test suite. |

### Example caller
The caller owns everything job-level — triggers, `services:`, secrets, and the test steps. Only the setup is delegated:

```yaml
name: Unit/Integration tests PHP8

on:
  pull_request:
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        php-versions: ['8.0', '8.1', '8.2', '8.3', '8.4']

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping --silent"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    # Secrets stay in the consuming repository.
    env:
      ROCKET_KEY: ${{ secrets.ROCKET_KEY }}
      ROCKET_CLOUDFLARE_API_KEY: ${{ secrets.ROCKET_CLOUDFLARE_API_KEY }}
      # ... other ROCKET_* / ROCKETCDN_* / ROCKET_CLOUDFLARE_* secrets

    steps:
      - uses: actions/checkout@v7

      - uses: wp-media/workflows/.github/actions/setup-php-composer@ref
        with:
          php-version: ${{ matrix.php-versions }}

      - uses: wp-media/workflows/.github/actions/setup-wp-tests@ref
        with:
          wp-version: 'latest'

      # The test run stays in the caller and can be as many steps as needed.
      - run: composer run-tests
```

For a coverage build, the caller drives it end to end — no shared coverage/Codacy logic:

```yaml
      - uses: wp-media/workflows/.github/actions/setup-php-composer@ref
        with:
          php-version: '8.4'
          coverage: xdebug
          extra-require: phpunit/phpcov

      - uses: wp-media/workflows/.github/actions/setup-wp-tests@ref
        with:
          wp-version: 'latest'

      - run: composer run-tests-coverage
      - run: composer report-code-coverage
      - uses: codacy/codacy-coverage-reporter-action@v1
        with:
          project-token: ${{ secrets.CODACY_PROJECT_TOKEN }}
          coverage-reports: tests/report/coverage.clover
```