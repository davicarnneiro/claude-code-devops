## ADDED Requirements

### Requirement: Validate Node.js dependencies on every push and PR
The system SHALL run `npm ci` in `src/` on every `push` to any branch and every `pull_request`, using Node.js with npm cache enabled.

#### Scenario: Push to any branch triggers validation
- **WHEN** a developer pushes to any branch
- **THEN** the `validate` job runs `npm ci` in `src/` and succeeds or fails based on dependency installation

#### Scenario: Pull request triggers validation
- **WHEN** a pull request is opened or updated
- **THEN** the `validate` job runs `npm ci` in `src/` before the PR can be merged

#### Scenario: npm cache accelerates validation
- **WHEN** the `validate` job runs with an existing cache
- **THEN** npm dependencies are restored from cache, reducing job duration

### Requirement: Build and push Docker image only on push to main
The system SHALL build the Docker image from `./src` context and push it to Docker Hub as `davicarneiro/evento-kube-news:<run_number>` only when a `push` event targets the `main` branch, and only after the `validate` job succeeds.

#### Scenario: Push to main after validation triggers build
- **WHEN** a push to `main` occurs and the `validate` job succeeds
- **THEN** the `build-and-push` job builds the Docker image and pushes to Docker Hub with tag `github.run_number`

#### Scenario: Push to non-main branch does not trigger build
- **WHEN** a push to a branch other than `main` occurs
- **THEN** the `build-and-push` job does not run

#### Scenario: Build failure prevents CD from running
- **WHEN** the `build-and-push` job fails
- **THEN** the `call-cd` job does not run and the deploy does not happen

### Requirement: Pass image tag to CD workflow via workflow_call
The system SHALL invoke the CD workflow (`cd.yml`) via `workflow_call` after `build-and-push` succeeds, passing `github.run_number` as `image-tag` input, with `secrets: inherit`.

#### Scenario: Successful build triggers CD via workflow_call
- **WHEN** the `build-and-push` job completes successfully on a push to `main`
- **THEN** the `call-cd` job invokes `cd.yml` with `image-tag: github.run_number` and `secrets: inherit`

#### Scenario: CD is not invoked on PRs or non-main branches
- **WHEN** the workflow runs in a PR or non-main branch context
- **THEN** the `call-cd` job does not execute
