# Deployment workflow

Support one reusable GitHub Actions job for metadata, package installation and optional app hooks. The calling organization selects the GitHub Environment, runner and optional Linux job container. GitHub-hosted Ubuntu is the default.

- Verify the release SHA and target org before deploying. Use production validation and quick deploy for production metadata.
- Support metadata-only Salesforce projects and package installs. Enable Alchemy and the app smoke test independently.
- Provide working caller examples for a sandbox and a staged release. Check both caller interfaces in CI.
- Support an optional caller-owned npm build script for generated metadata assets. Build in the deployment checkout before Salesforce login.
