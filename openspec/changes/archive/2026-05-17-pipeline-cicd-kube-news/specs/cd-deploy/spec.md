## ADDED Requirements

### Requirement: Accept image-tag as input from workflow_call or workflow_dispatch
The system SHALL accept `image-tag` as a required string input when triggered via `workflow_call` (from CI) or `workflow_dispatch` (manual run).

#### Scenario: CD triggered by CI via workflow_call
- **WHEN** `ci.yml` invokes `cd.yml` via `workflow_call` with `image-tag` set to `github.run_number`
- **THEN** the deploy job receives and uses the provided `image-tag`

#### Scenario: CD triggered manually via workflow_dispatch
- **WHEN** a user manually triggers `cd.yml` from the GitHub Actions UI and provides an `image-tag`
- **THEN** the deploy job uses the provided `image-tag` to deploy that specific image version

### Requirement: Authenticate to DOKS cluster using KUBE_CONFIG secret
The system SHALL use `azure/k8s-set-context@v4` with `method: kubeconfig` and the `KUBE_CONFIG` repository secret to authenticate to the DOKS cluster before the deploy step.

#### Scenario: Cluster context set before deploy
- **WHEN** the deploy job runs
- **THEN** `azure/k8s-set-context@v4` configures the kubectl context using the kubeconfig from `KUBE_CONFIG` secret before `azure/k8s-deploy` executes

### Requirement: Apply manifests and update image via azure/k8s-deploy
The system SHALL use `azure/k8s-deploy@v5` to apply all manifests in `k8s/` and substitute the image tag in a single step, targeting the `kube-news` namespace.

#### Scenario: Manifests applied and image substituted
- **WHEN** the deploy step runs
- **THEN** `azure/k8s-deploy@v5` applies all resources in `k8s/`, substitutes the image `davicarneiro/evento-kube-news` with the tag from `inputs.image-tag`, and waits for the rollout to complete

#### Scenario: Image substitution matches by image name
- **WHEN** `azure/k8s-deploy` processes the `images:` parameter
- **THEN** it matches the image name `davicarneiro/evento-kube-news` in the Deployment manifest and replaces its tag with the provided `image-tag`

#### Scenario: Failed rollout fails the deploy job
- **WHEN** pods do not reach Running state within the action's timeout
- **THEN** `azure/k8s-deploy` exits non-zero and the deploy job fails with a visible error

### Requirement: Prevent concurrent deploys
The system SHALL use `concurrency: deploy-main` on the deploy job to ensure only one deploy runs at a time, with the latest run canceling any in-progress run.

#### Scenario: Second deploy cancels in-progress deploy
- **WHEN** a second `cd.yml` run starts while a previous deploy job is still running
- **THEN** the previous run is canceled and the new run proceeds

### Requirement: Deploy runs in production environment
The system SHALL configure the deploy job with `environment: production` to enforce GitHub environment protections and visibility.

#### Scenario: Deploy job linked to production environment
- **WHEN** the deploy job runs
- **THEN** it is associated with the `production` environment in GitHub Actions, enabling environment-level approvals or protections if configured
