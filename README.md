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