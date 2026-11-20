# Deployment workflow

Support one reusable GitHub Actions job for metadata, package installation and optional app hooks. The calling organization selects the GitHub Environment, runner and optional Linux job container. GitHub-hosted Ubuntu is the default.

- Verify the release SHA and target org before deploying. Use production validation and quick deploy for production metadata.
- Support metadata-only Salesforce projects and package installs. Enable Alchemy and the app smoke test independently.
- Provide working caller examples for a sandbox and a staged release. Check both caller interfaces in CI.
- Provide a standalone scratch org CI example that app repos can copy to create an org, deploy metadata, run Apex tests and clean up on same-repository pull requests targeting `main`. Include it in workflow linting.
- Show how to inspect a completed scratch org job in GitHub Actions in a recorded walkthrough.
