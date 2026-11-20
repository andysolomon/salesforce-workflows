# Salesforce workflows

Reusable GitHub Actions workflows for Salesforce application repositories. Salesforce CLI ships source and package versions. The application's Alchemy stack configures the target org after deployment.

## Adopt the workflow

Copy [examples/release.yml](examples/release.yml) into the app's `.github/workflows/` directory. It deploys unpackaged metadata to Dev on a push to `main`. A manual run on `main` deploys the selected commit and released package versions to UAT, then optionally deploys that same release to Prod. Configure Prod approval before enabling the Prod job. Review the shared workflow release and pin the caller's `uses` reference to a full commit SHA for production use.

Keep pull request CI in the app repo. Run unit tests and type checks there, and make those checks required on the release branch. Call the deploy workflow from trusted refs. GitHub authenticates the runner to Salesforce and runs commands from the **calling app repo**.

### Configure GitHub environments

Create `dev`, `uat`, and `prod` in the calling repository. Set allowed deployment branches and tags for each. Configure Prod required reviewers where supported by your GitHub plan. Use an approval process appropriate to your repository before enabling production deployments. The reusable job declares the environment and receives its protection rules and credentials.

Set these variables and secrets for each environment:

| Name | Kind | Value |
| --- | --- | --- |
| `SF_LOGIN_URL` | Variable | Salesforce login endpoint for the environment |
| `SF_ORG_ID` | Variable | Expected 15- or 18-character org ID |
| `SF_CLIENT_ID` | Variable | OAuth consumer key for the JWT deployment app |
| `SF_USERNAME` | Variable | Username of the authorized deployment user |
| `SF_JWT_KEY` | Secret | PEM private key matching the app's certificate |
| `SF_PACKAGE_INSTALLATION_KEY` | Optional secret | Installation key if the package requires one. A call supports one key across its package list. |
| `CLOUDFLARE_ACCOUNT_ID` | Optional variable | Account hosting the Alchemy state store, if using Cloudflare |
| `CLOUDFLARE_API_TOKEN` | Optional secret | Token authorized to access that state store |
| `ALCHEMY_APP_SECRETS` | Optional secret | Payload read by your Alchemy stack, for example credentials for an external integration |

The example passes secrets by name. In the hosted runner, configure the Alchemy stack to use a persistent state backend and provision it before the first deployment. Use a distinct Alchemy stack name per app when sharing a backend, and use the same stage on every run for a given org. If the stack reads a structured secret payload, keep it out of logs. GitHub's automatic masking can miss individually parsed values.

The job serializes deployments to the same stage within one calling repo. GitHub concurrency groups are repository-scoped. If multiple app repos write to the same org, coordinate those deployments across repos.

## Deployment inputs

| Input | Value |
| --- | --- |
| `stage` | `dev`, `uat`, or `prod`. Also used as the CLI alias and Alchemy stage. |
| `release-sha` | Full 40-character app commit SHA. Pass the same SHA from UAT to Prod. |
| `manifest` | Optional path to an unpackaged metadata manifest, relative to the app repo root. |
| `package-versions` | Optional space-separated list of exact unlocked or managed 2GP version IDs starting with `04t`, ordered by dependency. Use released versions for a persistent UAT-to-Prod pipeline. |
| `alchemy-config` | Stack entry point, default `alchemy.run.ts`. Pass an empty string to skip Alchemy. |

The app needs a committed npm lockfile and a `salesforce:smoke` npm script. The smoke script runs with `STAGE` and `SF_TARGET_ORG` set and the Salesforce CLI already authenticated. For example, a billing app can read a known record and exercise its callout using an org-specific test case. Put Alchemy in the app's lockfile if the job applies a stack.

- **Unpackaged:** Set `manifest`. Prod validates against the target and quick-deploys that successful validation.
- **Unlocked or managed 2GP:** Set `package-versions`. Build and promote versions separately in the Dev Hub. This job installs the versions you supply. It uses CLI noninteractive install, which approves external sites declared by a package. Review those as part of approving the version. Unlocked upgrades request `DeprecateOnly`; Salesforce still controls component-specific upgrade behavior.
- **Mixed:** Set both inputs. This job installs packages first, then deploys unpackaged metadata that can depend on those packages. If a package requires unpackaged metadata already present in the target org, ship that prerequisite before calling this job.
- **Configuration only:** Omit both Salesforce deployment inputs and provide the stack. The job runs Alchemy and the smoke test.

The job verifies the authenticated org ID before applying changes. If a deployment or install is still in progress when its CLI wait expires, check the Salesforce job status before retrying. A release can have updated Salesforce even if a later Alchemy or smoke-test step fails. Review the failed step and target org before proceeding.

## References

- [Sharing private actions and workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/share-across-private-repositories)
- [GitHub deployment environments and approval availability](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments)
- [Salesforce JWT login](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_org_login_jwt.html)
- [Alchemy shared state](https://alchemy.run/state-store/)
