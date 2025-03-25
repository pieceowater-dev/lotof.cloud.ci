# lotof.cloud.ci

This repository contains CI/CD workflows and custom GitHub Actions for managing the `lotof.cloud` project.

## Overview

The `lotof.cloud.ci` repository includes automation scripts and configurations to streamline the deployment process. It leverages GitHub Actions to perform tasks such as updating Docker image versions in the `lotof.cloud.state` repository.

## Custom GitHub Actions

### `update-state`

The `update-state` action updates the Docker image version in the `lotof.cloud.state` repository. It performs the following steps:

1. Sets up an SSH key for secure access to the repository.
2. Clones the `lotof.cloud.state` repository.
3. Updates the image version in the `manifest.yaml` file.
4. Commits and pushes the changes to the specified branch.

#### Inputs

- `repo-name`: The name of the repository to update (required).
- `commit-sha`: The commit SHA of the Docker image (required).
- `branch`: The branch to push the changes to (required).
- `ssh-key`: The SSH private key for authentication (required).

## Usage

To use the `update-state` action, include it in your workflow file:

```yaml
jobs:
    update-state:
        runs-on: ubuntu-latest
        steps:
            - name: Update Docker image version
                uses: ./.github/actions/update-state
                with:
                    repo-name: "your-repo-name"
                    commit-sha: ${{ github.sha }}
                    branch: "main"
                    ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.