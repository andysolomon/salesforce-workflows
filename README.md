# Salesforce workflows

Reusable GitHub Actions workflows for Salesforce application repositories. Salesforce CLI ships source and package versions. Apps can also run an Alchemy stack and a post-deployment smoke test.

## Adopt the workflow

For a metadata-only sandbox deployment, copy [examples/metadata.yml](examples/metadata.yml) into the app's `.github/workflows/` directory. Configure its GitHub Environment, commit `sfdx-project.json` and the manifest, then dispatch the workflow from `main`. The caller supplies the release SHA, stage and manifest. A metadata-only app can omit `package.json` and the npm lockfile.

For promotion with Alchemy and smoke tests, copy [examples/release.yml](examples/release.yml) instead. It deploys unpackaged metadata to Dev on a push to `main`. A manual run on `main` deploys the selected commit and released package versions to UAT, then optionally deploys that same release to Prod. Configure Prod approval before enabling the Prod job. Review the shared workflow release and pin the caller's `uses` reference to a full commit SHA for production use.

Keep pull request CI in the app repo. Run unit tests and type checks there, and make those checks required on the release branch. Call the deploy workflow from trusted refs. GitHub authenticates the runner to Salesforce and runs commands from the **calling app repo**.

### Configure GitHub environments

Create the environments used by your caller. The metadata example uses `integration-sandbox` and the promotion example uses `dev`, `uat`, and `prod`. Set `environment-name` to use your organization’s naming convention. The `stage` input selects sandbox or production deployment behavior. Use `stage: prod` for every production target. Set allowed deployment branches and tags for each. Configure Prod required reviewers where supported by your GitHub plan. Use an approval process appropriate to your repository before enabling production deployments. The reusable job declares the environment and receives its protection rules and credentials.

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

The job serializes deployments to the same GitHub Environment within one calling repo. GitHub concurrency groups are repository-scoped. If multiple app repos write to the same org, coordinate those deployments across repos.

## Test in a scratch org

The [GitHub walkthrough (MP4, 1m 30s)](docs/videos/scratch-org-pull-request.mp4) shows this example and follows a completed [Apex Recipes pull request check](https://github.com/trailheadapps/apex-recipes/actions/runs/38726004243/job/116392083619) through scratch org creation, Apex tests, and cleanup.

Copy [examples/scratch-org.yml](examples/scratch-org.yml) into the app's `.github/workflows/` directory. The workflow runs when a pull request targeting `main` is opened, reopened, or updated with new commits. The scratch org job runs for branches in the same repository and skips Dependabot-triggered runs. It creates a one-day scratch org, deploys the manifest with `RunLocalTests`, and attempts to delete the org even if deployment or testing fails. The one-day expiration also applies if the run ends before it can delete the org.

1. Commit `sfdx-project.json`, `config/project-scratch-def.json`, and `manifest/package.xml` in the app repo. Include the Apex classes and tests you want to deploy in the manifest.
2. Create the `scratch-org-ci` GitHub Environment in the app repo. Under **Deployment branches and tags**, choose **Selected branches and tags** and add the branch rule `refs/pull/*/merge` so pull request runs can access the environment. [GitHub environment branch rules](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments#deployment-branches-and-tags)
3. Log in to a Dev Hub locally using `sf org login web --alias dev-hub` with a user authorized to create and delete scratch orgs. Get its authorization URL with `sf org auth show-sfdx-auth-url --target-org dev-hub --json` and save only the `result.sfdxAuthUrl` value as the environment secret `DEVHUB_SFDX_URL`. The URL contains a refresh token. [Salesforce SFDX URL authentication](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_org_login_sfdx-url.html)

## Choose where deployments run

The default is a GitHub-hosted Ubuntu runner. To use a runner available to the calling repository, add a JSON selector under the caller job's `with` block. For example:

```yaml
runs-on: '["self-hosted", "linux", "salesforce-deploy"]'
```

A runner group can be selected with `runs-on: '{"group":"release-runners","labels":"linux"}'`. For a shared organization setting, put the JSON in an Actions organization variable accessible to the caller and pass `runs-on: ${{ vars.SALESFORCE_RUNS_ON }}`. Configure runner access and network routes to the Salesforce org and any registries used by the job. [GitHub runner selectors](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job)

Use a Linux deployment runner with Bash, Git and jq. For self-hosting, provision a fresh runner per job or use an isolated disposable job container. Salesforce CLI saves authentication state in the job user's home directory. Job containers require Docker on the runner, a compatible Linux image with Bash, Git and jq, and an image the runner can pull.

The workflow uses Salesforce CLI 2.159.7 when available on the job path. Otherwise it installs that version with npm in the runner's temporary directory. Node 24 is set up when installing the CLI or running app scripts. Organizations that use the Salesforce CLI image can set this additional input on a Linux runner:

```yaml
# salesforce/cli:2.159.7-slim (linux/amd64)
container-image: docker.io/salesforce/cli@sha256:14e83ce5d8d6357c5197419a857b5e51c978eead5491a996f502aa52992d8019
```

## Deployment inputs

| Input | Value |
| --- | --- |
| `stage` | `dev`, `uat`, or `prod`. Also used as the CLI alias and Alchemy stage. |
| `environment-name` | Calling repository's GitHub Environment for credentials, approvals and deployment concurrency. Defaults to `stage`. |
| `runs-on` | JSON runner selector. Defaults to `["ubuntu-latest"]`. |
| `container-image` | Optional Linux job container image. Omit to run directly on the selected runner. |
| `release-sha` | Full 40-character app commit SHA. Pass the same SHA from UAT to Prod. |
| `manifest` | Optional path to an unpackaged metadata manifest, relative to the app repo root. |
| `package-versions` | Optional space-separated list of exact unlocked or managed 2GP version IDs starting with `04t`, ordered by dependency. Use released versions for a persistent UAT-to-Prod pipeline. |
| `alchemy-config` | Optional Alchemy stack entry point. |
| `smoke-test` | Set to `true` to run the app’s `salesforce:smoke` npm script. Defaults to `false`. |

When upgrading a v1 caller that relied on the previous defaults, set `alchemy-config: alchemy.run.ts` and `smoke-test: true` to retain both steps. Examples target the updated workflow on `main`. For rollout, pin the reviewed commit containing these inputs.

Apps that enable Alchemy or the smoke test need a committed npm lockfile. Enabling the smoke test requires a `salesforce:smoke` npm script. The smoke script runs with `STAGE` and `SF_TARGET_ORG` set and the Salesforce CLI already authenticated. For example, a billing app can read a known record and exercise its callout using an org-specific test case. Put Alchemy in the app's lockfile if the job applies a stack.

- **Unpackaged:** Set `manifest`. Prod validates against the target and quick-deploys that successful validation.
- **Unlocked or managed 2GP:** Set `package-versions`. Build and promote versions separately in the Dev Hub. This job installs the versions you supply. It uses CLI noninteractive install, which approves external sites declared by a package. Review those as part of approving the version. Unlocked upgrades request `DeprecateOnly`; Salesforce still controls component-specific upgrade behavior.
- **Mixed:** Set both inputs. This job installs packages first, then deploys unpackaged metadata that can depend on those packages. If a package requires unpackaged metadata already present in the target org, ship that prerequisite before calling this job.
- **Configuration only:** Omit both Salesforce deployment inputs and provide the stack. The job runs Alchemy and any enabled smoke test.

The job verifies the authenticated org ID before applying changes. If a deployment or install is still in progress when its CLI wait expires, check the Salesforce job status before retrying. A release can have updated Salesforce even if a later Alchemy or smoke-test step fails. Review the failed step and target org before proceeding.

## References

- [Sharing private actions and workflows](https://docs.github.com/en/actions/how-tos/reuse-automations/share-across-private-repositories)
- [GitHub deployment environments and approval availability](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/review-deployments)
- [Salesforce JWT login](https://developer.salesforce.com/docs/platform/salesforce-cli-reference/guide/cli_reference_org_login_jwt.html)
- [Alchemy shared state](https://alchemy.run/state-store/)
