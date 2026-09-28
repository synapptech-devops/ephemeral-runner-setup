# GitHub Actions Runner Controller on k3s

This guide describes how to deploy the modern **GitHub Actions Runner Controller (ARC)** on Kubernetes/k3s using **Autoscaling Runner Scale Sets**.

The resulting setup provides organization-level, ephemeral Linux GitHub Actions runners that automatically scale from zero when jobs are queued.

## Architecture

```text
GitHub Actions
      │
      │ Job targets:
      │ runs-on: arc-org-runners
      ▼
GitHub Actions Service
      │
      ▼
ARC AutoscalingListener
      │
      ▼
ARC Controller
      │
      ▼
EphemeralRunnerSet
      │
      ▼
Kubernetes Pod
┌──────────────────────────┐
│ GitHub Actions Runner    │
│                          │
│ Executes one job         │
└──────────────────────────┘
      │
      ▼
Job completes
      │
      ▼
Runner pod is deleted
```

With no queued jobs, the runner scale set scales down to zero pods.

---

## Configuration

This guide uses the following configuration:

| Setting                 | Value                        |
| ----------------------- | ---------------------------- |
| ARC version             | `0.14.2`                     |
| GitHub organization     | `synapptech-devops`          |
| Controller namespace    | `arc-systems`                |
| Runner namespace        | `arc-runners`                |
| Controller Helm release | `arc`                        |
| Runner scale set        | `arc-org-runners`            |
| Authentication          | GitHub Personal Access Token |
| Runner OS               | Linux                        |

---

## Prerequisites

The following tools must be installed:

- Kubernetes or k3s
- `kubectl`
- Helm 3
- GitHub CLI (`gh`)
- Access to the GitHub organization
- A GitHub PAT with the permissions required to manage organization runners

Verify the environment:

```powershell
kubectl get nodes
helm version
gh --version
gh auth status
```

All Kubernetes nodes should report `Ready`.

---

## 1. Authenticate Helm to GitHub Container Registry

The ARC Helm charts are distributed through GitHub Container Registry (`ghcr.io`).

Authenticate Helm using the existing GitHub CLI credentials:

```powershell
gh auth token | helm registry login ghcr.io `
  --username YOUR_GITHUB_USERNAME `
  --password-stdin
```

Expected result:

```text
Login Succeeded
```

> [!NOTE]
> This authentication is used by Helm to pull the ARC charts from GHCR. It is separate from the GitHub API authentication used by ARC itself.

---

## 2. Install the ARC Controller

Install the ARC controller into the `arc-systems` namespace:

```powershell
helm install arc `
  --namespace "arc-systems" `
  --create-namespace `
  --version 0.14.2 `
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Verify the controller:

```powershell
kubectl get pods -n arc-systems
```

Expected:

```text
NAME                                      READY   STATUS
arc-gha-rs-controller-xxxxxxxxxx-xxxxx    1/1     Running
```

Verify that the ARC CRDs were installed:

```powershell
kubectl get crd | Select-String "actions.github.com"
```

The installation should include CRDs such as:

```text
autoscalinglisteners.actions.github.com
autoscalingrunnersets.actions.github.com
ephemeralrunners.actions.github.com
ephemeralrunnersets.actions.github.com
```

---

## 3. Create the Runner Namespace

Create a separate namespace for runner resources:

```powershell
kubectl create namespace arc-runners
```

Verify:

```powershell
kubectl get namespace arc-runners
```

The ARC resources are intentionally separated:

```text
arc-systems
├── ARC controller
└── AutoscalingListener

arc-runners
├── AutoscalingRunnerSet
├── EphemeralRunnerSet
└── Ephemeral runner pods
```

---

## 4. Create the GitHub Authentication Secret

Read the GitHub PAT interactively:

```powershell
$pat = Read-Host "Enter GitHub PAT"
```

Create the Kubernetes secret:

```powershell
kubectl create secret generic arc-github-secret `
  --namespace arc-runners `
  --from-literal=github_token="$pat"
```

Clear the PowerShell variable:

```powershell
$pat = $null
```

Verify that the secret exists:

```powershell
kubectl get secret arc-github-secret -n arc-runners
```

Expected:

```text
NAME                TYPE     DATA
arc-github-secret   Opaque   1
```

> [!WARNING]
> Do not print or decode the secret simply to verify it. Checking that the Kubernetes Secret exists is sufficient.

---

## 5. Install the Organization Runner Scale Set

Install the runner scale set:

```powershell
helm install arc-org-runners `
  --namespace "arc-runners" `
  --version 0.14.2 `
  --set-string githubConfigUrl="https://github.com/synapptech-devops" `
  --set-string githubConfigSecret="arc-github-secret" `
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

The important values are:

```text
githubConfigUrl
└── https://github.com/synapptech-devops

githubConfigSecret
└── arc-github-secret
```

Because `githubConfigUrl` points to the organization rather than an individual repository, this creates an **organization-level runner scale set**.

---

## 6. Verify the Runner Scale Set

Check the `AutoscalingRunnerSet`:

```powershell
kubectl get autoscalingrunnersets -n arc-runners
```

Expected:

```text
NAME
arc-org-runners
```

Verify the configured GitHub URL:

```powershell
kubectl get autoscalingrunnerset arc-org-runners `
  -n arc-runners `
  -o jsonpath='{.spec.githubConfigUrl}'
```

Expected:

```text
https://github.com/synapptech-devops
```

---

## 7. Verify the Autoscaling Listener

Check the controller namespace:

```powershell
kubectl get pods -n arc-systems -o wide
```

Both the controller and listener should eventually be running:

```text
NAME                                      READY   STATUS
arc-gha-rs-controller-xxxxxxxxxx-xxxxx    1/1     Running
arc-org-runners-xxxxxxxx-listener         1/1     Running
```

Check the `AutoscalingListener` resource:

```powershell
kubectl get autoscalinglisteners -A
```

The listener should reference:

```text
AUTOSCALINGRUNNERSET NAMESPACE   arc-runners
AUTOSCALINGRUNNERSET NAME        arc-org-runners
```

---

## 8. Verify the GitHub Connection

Find the listener pod:

```powershell
kubectl get pods -n arc-systems
```

Inspect its logs:

```powershell
kubectl logs `
  -n arc-systems `
  <LISTENER_POD_NAME> `
  --tail=50
```

For example:

```powershell
kubectl logs `
  -n arc-systems `
  arc-org-runners-xxxxxxxx-listener `
  --tail=50
```

Healthy listener logs should contain messages similar to:

```text
refreshing token
getting runner registration token
getting Actions tenant URL and JWT
Starting listener
Handling initial session statistics
Calculated target runner count
Getting next message
```

At this point ARC is connected to GitHub and waiting for jobs.

---

## 9. Verify Scale-to-Zero

Check the runner namespace:

```powershell
kubectl get pods -n arc-runners
```

When no jobs are queued, it is normal to see:

```text
No resources found in arc-runners namespace.
```

The default configuration allows ARC to scale to zero:

```text
No queued jobs
      │
      ▼
Desired runners = 0
      │
      ▼
No runner pods
```

A missing runner pod while the system is idle is therefore **expected behavior**, not an error.

---

## 10. Configure GitHub Actions Workflows

Workflows must target the ARC runner scale set by name.

Replace traditional self-hosted labels such as:

```yaml
runs-on: [self-hosted, Linux, X64]
```

with:

```yaml
runs-on: arc-org-runners
```

For example:

```yaml
name: ARC Test

on:
  workflow_dispatch:

jobs:
  test:
    runs-on: arc-org-runners

    steps:
      - uses: actions/checkout@v7

      - name: Runner information
        run: |
          echo "Hello from ARC"
          echo "Runner: $RUNNER_NAME"
          echo "OS: $RUNNER_OS"
          uname -a
```

The relationship is:

```text
ARC scale set
arc-org-runners
      │
      ▼
GitHub Actions workflow
runs-on: arc-org-runners
```

---

## 11. Test Autoscaling

Before triggering the workflow, watch the runner namespace:

```powershell
kubectl get pods -n arc-runners -w
```

Trigger the workflow from GitHub Actions.

The expected lifecycle is:

```text
GitHub job queued
      │
      ▼
AutoscalingListener receives job
      │
      ▼
Desired runners: 0 → 1
      │
      ▼
EphemeralRunner created
      │
      ▼
Kubernetes runner pod created
      │
      ▼
Runner registers with GitHub
      │
      ▼
Workflow executes
      │
      ▼
Job completes
      │
      ▼
Runner pod is deleted
      │
      ▼
Desired runners: 1 → 0
```

A successful runner will have a generated name similar to:

```text
arc-org-runners-xxxxx-runner-xxxxx
```

Each runner is ephemeral and normally handles a job before being removed.

---

## Runner Image Tooling

The default ARC runner image provides the GitHub Actions runner environment, but it does **not** necessarily contain all development tools required by application pipelines.

For example, workflows that execute:

```powershell
pwsh -NoProfile -File src/toolchain.ps1
```

require PowerShell to be installed in the runner image.

If it is absent, the workflow will fail with an error such as:

```text
pwsh: command not found
Process completed with exit code 127
```

This does **not** indicate an ARC failure. If the workflow reached this point, ARC successfully:

1. Received the GitHub job.
2. Scaled the runner set.
3. Created an ephemeral Kubernetes pod.
4. Registered the runner with GitHub.
5. Started executing the workflow.

The runner image simply lacks an application dependency.

---

## Custom Runner Image

For production workloads, create a custom Linux runner image containing the tools required by the organization's pipelines.

Potential tooling includes:

```text
GitHub Actions Runner
├── PowerShell Core
├── .NET SDK
├── Node.js
├── pnpm
├── Git
└── Container build tooling
```

Keep the ARC infrastructure configuration separate from the development toolchain wherever possible.

This makes the architecture easier to maintain:

```text
ARC
│
├── Kubernetes integration
├── GitHub authentication
├── Runner registration
└── Autoscaling

Custom Runner Image
│
├── PowerShell
├── .NET
├── Node.js
├── pnpm
└── Build tooling

GitHub Workflow
│
└── Application-specific build/test/release logic
```

---

## Useful Commands

Check ARC controller and listener:

```powershell
kubectl get pods -n arc-systems
```

Check runner resources:

```powershell
kubectl get pods -n arc-runners
```

Watch ephemeral runners:

```powershell
kubectl get pods -n arc-runners -w
```

Check runner scale sets:

```powershell
kubectl get autoscalingrunnersets -n arc-runners
```

Check listeners:

```powershell
kubectl get autoscalinglisteners -A
```

Check ephemeral runner sets:

```powershell
kubectl get ephemeralrunnersets -n arc-runners
```

Inspect the scale set:

```powershell
kubectl describe autoscalingrunnerset arc-org-runners -n arc-runners
```

Check controller logs:

```powershell
kubectl logs `
  -n arc-systems `
  deployment/arc-gha-rs-controller `
  --tail=100
```

Check listener logs:

```powershell
kubectl logs `
  -n arc-systems `
  <LISTENER_POD_NAME> `
  --tail=100
```

---

## Uninstall

Remove the runner scale set first:

```powershell
helm uninstall arc-org-runners `
  --namespace arc-runners
```

Then remove the controller:

```powershell
helm uninstall arc `
  --namespace arc-systems
```

Remove the namespaces if they are no longer required:

```powershell
kubectl delete namespace arc-runners
kubectl delete namespace arc-systems
```

---

## Next Steps

The ARC infrastructure is now responsible for:

- Connecting Kubernetes to GitHub Actions
- Receiving organization-level jobs
- Creating ephemeral Linux runners
- Scaling from zero
- Removing runners after jobs finish

The next step is to create a **custom ARC runner image** containing the development and container-build tooling required by the pipelines.
