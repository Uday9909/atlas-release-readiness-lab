# Release Readiness Pack — Atlas Service

Fill in every section below as you work through the three tasks. Keep entries short
and factual. Do not paste secret values anywhere in this file.

## 1. Release metadata

- Release version: 1.4.0
- Git commit hash: 4970a00e5bd62a4d2f675d5fe74748cd0a700329 (tagged commit)
- Git tag: v1.4.0
- Docker image name and version: atlas-service:v1.4.0
- Date/time: 2026-10-08
- Prepared by: Udaybir Singh

## 2. Risk assessment (Task 1 — Docker)

- Issue found (what failed and how you noticed): `docker build` aborted at the
  `COPY` step. The Dockerfile copied `requirements-prod.txt`, a file that does not
  exist in the build context (the repo only ships `requirements.txt`).
- Impact if shipped as-is: no image could be produced at all — the release pipeline
  would fail at the first build step and nothing could be deployed.
- Evidence (build error summary): `COPY failed: file not found in build context or
  excluded by .dockerignore: stat requirements-prod.txt: file does not exist`.
- Fix applied: changed the instruction to `COPY requirements.txt .` so the existing
  pinned dependency file is copied to the image working directory.
- Residual risk after fix: low. Build succeeds and the image runs; no other
  Dockerfile defects were present (`EXPOSE 5000` matches the app port).

## 3. Pipeline stages (Task 2 — Kubernetes)

Describe what happens at each stage and where it was validated:

- Validate: `docker build` (local and the `release-check` CI job) compiles the image;
  Kubernetes manifests are checked before apply. Validated locally: `docker build`
  succeeded and manifests were applied to a local kind cluster.
- Build: `docker build -t atlas-service:v1.4.0 .` produces image
  `atlas-service:v1.4.0` (image id `sha256:2e83c7a8…`). Validated locally and by the
  `release-check` job's build step.
- Deploy: `production-deploy` job runs after the `production` environment approval
  and applies the manifests (`kubectl apply -f k8s/`). Validated locally against a
  kind cluster — Deployment, Secret and Service created.
- Smoke test: `curl http://localhost:8080/health` for the container, and the same
  request through the Service (`kubectl port-forward svc/atlas-service 8081:80`).
  Both returned HTTP 200 with `{"status":"ok","version":"1.4.0"}`.

## 4. Environment gate

- Where does human approval occur in the pipeline? On the `production-deploy` job,
  which declares `environment: production`. The job is held until a reviewer
  approves the deployment in GitHub.
- Which GitHub environment enforces it? `production`.
- Why is this gate placed before production and not before `release-check`?
  `release-check` only validates (checks out, builds the image) and touches no live
  environment or credentials, so it is safe to run unattended. The irreversible
  action is the production deploy, so the human approval belongs immediately before
  that job — approvals placed earlier would still leave the deploy unguarded.

## 5. Secret handling (Task 3 — Workflow)

- Issue found in `.github/workflows/release.yml`: a step named "Show registry token"
  ran `echo "Using token ${{ secrets.REGISTRY_TOKEN }}"`, printing the token into the
  workflow log.
- Why was it unsafe? Workflow logs are retained and readable by anyone with access
  to the repository, so the token would be exposed and could be harvested, even if
  GitHub attempts to mask it. Secrets must never be emitted to logs.
- Fix applied (describe the safe pattern used, do not paste secret values): the
  `echo` step was removed. The token is now consumed only where it is actually
  required — as the `password` input of a `docker/login-action@v3` step, which passes
  it to the tool without printing it. The `${{ secrets.REGISTRY_TOKEN }}` reference
  appears exactly once, in that credential field.
- Confirm no password, token, or private key is committed anywhere in the repo:
  confirmed. A scan of the working tree for private-key blocks, cloud keys and
  literal `password`/`token` assignments returned no matches; `k8s/secret.yaml`
  contains only the placeholder `API_TOKEN: replace-me`.

## 6. Validation evidence

List commands run and a one-line result for each (attach screenshots separately
in your PDF submission):

- `docker build ...`: succeeded — image `atlas-service:v1.4.0` built (id
  `sha256:2e83c7a8…`).
- `curl http://localhost:8080/health`: HTTP 200 `{"status":"ok","version":"1.4.0"}`
  from the running container (`-p 8080:5000`).
- `kubectl apply --dry-run=client -f k8s/`: requires a cluster for API discovery on
  this kubectl version, so it was validated on a live cluster instead — `kubectl
  apply -f k8s/` created all three objects and `kubectl wait` reported the Deployment
  available.
- `kubectl diff -f k8s/` (or note if a cluster was unavailable): a cluster was
  available (local kind). No diff — the manifests are applied and current.
- `kubectl get deployment,pods,service` (or note if a cluster was unavailable):
  `atlas-deployment` 1/1 Available, pod `Running`, `atlas-service` ClusterIP with
  selector `app=atlas`; endpoints `10.244.0.5:5000` (Service → Pod routing verified).
- `git tag` / `git show v1.4.0 --stat`: tag `v1.4.0` created on the fixed release
  commit; see metadata in section 1.

## 7. Rollback plan

- Rollback trigger (what condition would cause a rollback): the new image fails its
  health check or produces sustained errors after go-live — e.g. `/health` not
  returning 200, elevated 5xx rate, or a failed smoke test in production.
- Rollback action (previous known-good image tag and how it is redeployed): redeploy
  the previous known-good image `atlas-service:v1.3.0`, for example
  `kubectl set image deployment/atlas-deployment atlas-service=atlas-service:v1.3.0`
  followed by `kubectl rollout status deployment/atlas-deployment`, or
  `kubectl rollout undo` if the prior revision is desired.

## 8. Kubernetes vs Terraform

- One sentence: why would Terraform manage infrastructure provisioning while
  Kubernetes manages the application workload here? Terraform provisions the
  long-lived infrastructure (cluster, networking, IAM) declaratively, while
  Kubernetes owns the runtime application workload — scheduling, scaling and
  self-healing the Atlas containers on top of that infrastructure.

## 9. Final decision

- Decision: **Ready**
- Reason: all three release-blocking issues are fixed and verified — the image builds
  and serves HTTP 200 on `/health`, the Service selector matches the Deployment Pod
  label so traffic routes (endpoints populated), and the workflow no longer logs the
  registry token while keeping the `production` approval gate intact.
