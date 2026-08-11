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
Two composite actions handle the PHPUnit setup so your workflow only declares the job and the test steps:

- `.github/actions/setup-php-composer`: sets up PHP and installs Composer dependencies.
- `.github/actions/setup-wp-tests`: caches/installs the WordPress test suite (bundling `install-wp-tests.sh`) and configures the MySQL 8 auth workaround.

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
Declare the job (`services:`, secrets), delegate the setup, then run your own test steps:

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

    env:
      ROCKET_KEY: ${{ secrets.ROCKET_KEY }}
      ROCKET_CLOUDFLARE_API_KEY: ${{ secrets.ROCKET_CLOUDFLARE_API_KEY }}
      # ... other secrets your tests need

    steps:
      - uses: actions/checkout@v7

      - uses: wp-media/workflows/.github/actions/setup-php-composer@ref
        with:
          php-version: ${{ matrix.php-versions }}

      - name: Setup problem matchers for PHP
        run: echo "::add-matcher::${{ runner.tool_cache }}/php.json"

      - name: Setup problem matchers for PHPUnit
        run: echo "::add-matcher::${{ runner.tool_cache }}/phpunit.json"

      - uses: wp-media/workflows/.github/actions/setup-wp-tests@ref
        with:
          wp-version: 'latest'

      - run: composer run-tests
```

The problem matchers are registered in the caller (not in `setup-php-composer`) so
that PHP/PHPUnit annotations surface against the workflow that owns the test run.

Coverage build:

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