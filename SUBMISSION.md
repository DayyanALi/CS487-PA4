# PA4 Submission: TaskFlow Pipeline

## Student Information

| Field | Value |
|---|---|
| Name | Dayyan Ali |
| Roll Number | 26100007 |
| Resource Group | `rg-sp26-26100007` |
| Region Used | `swedencentral` |

---

## Task 1: App Service Web App

### Forked Repository
![Forked repository](docs/forked-repo.png)

This is my forked PA4 repository used for all assignment changes and submission files.

### App Service Overview
![Web App overview](docs/web-app-overview.png)

The frontend Web App `pa4-26100007` is deployed in the assignment resource group and serves the TaskFlow UI.

### Deployment Center / GitHub Actions
![Deployment Center](docs/webapp-deployment-center.png)

The Web App is connected to my GitHub fork through Deployment Center/GitHub Actions for CI/CD.

### Live Web App and Environment Variables
![Web App running](docs/web-app-running.jpeg)

The live TaskFlow frontend is reachable through Azure App Service.

![Web App environment variables](docs/web-app-env-vars.jpeg)

The Web App has the Function start and status settings used to call the Durable Function backend.

---

## Task 2: Azure Container Registry

### ACR Overview
![ACR overview](docs/acr-overview.jpeg)

The Azure Container Registry `pa426100007` is created in the resource group using the Basic SKU.

### Docker Builds and Pushes
![Docker build 1](docs/docker-build-1.jpeg)

![Docker build 2](docs/docker-build-2.jpeg)

These screenshots show local Docker builds for the validator API, report job, and Function App images.

![Docker pushes](docs/docker-pushes.jpeg)

The container images were tagged and pushed to ACR.

### ACR Repositories
![ACR repositories](docs/acr-repositories.png)

The registry contains the required `validate-api`, `report-job`, and `func-app` repositories.

---

## Task 3: Durable Function Implementation

### Completed Function Code
[function_app.py](function-app/function_app.py)

The Durable orchestrator receives an order, calls `validate_activity`, rejects invalid orders, and calls `report_activity` for valid orders. The report activity creates an ACI report job and returns the generated Blob Storage report URL.

---

## Task 4: Function App Container Deployment

### Function App Overview
![Function App overview](docs/function-app-overview.jpeg)

The Function App `pa4-26100007-func` is deployed as a Linux containerized Function App.

### Orchestration Smoke Test
![Task 4 curl start](docs/task4-curl-start.png)

The HTTP starter returned a Durable orchestration instance id and status query URL, proving the deployed Function App could start workflows.

### Expected Early Failure
![Task 4 failed status](docs/task4-status-failed.jpeg)

The early smoke test reached the orchestrator and failed before downstream wiring was complete, which confirmed the function code was deployed and executing.

---

## Task 5: AKS Validator

### Kubernetes Pods, Service, and Health Check
![AKS pods service health](docs/task5-pods-service-health.png)

The validator pod is running in AKS and the LoadBalancer service exposes the validator endpoint.

### Validator API Tests
![Validator tests](docs/task5-validator-tests.png)

The validator accepts a normal order and rejects an invalid order where quantity exceeds the limit.

### Function App VALIDATE_URL
![Function VALIDATE_URL](docs/function-validate-url.png)

The Function App is configured with the AKS validator URL so `validate_activity` can call the `/validate` endpoint.

---

## Task 6: ACI Report Job

### Reports Blob Container
![Blob reports container](docs/blob-reports-container.jpeg)

The `reports` Blob container stores generated PDF reports.

### Manual ACI Run
![ACI state](docs/task6-aci-state.jpeg)

The manual report-job container completed successfully and exited after generating the report.

### Managed Identity
![Function managed identity](docs/function-managed-identity.jpeg)

The user-assigned managed identity `mi-pa4-26100007` is attached to the Function App for secretless Azure operations.

### Report App Settings
![Function report settings](docs/function-report-env-vars.jpeg)

The Function App has the report image, ACR, storage, resource group, subscription, and identity settings needed by `report_activity`.

---

## Task 7: End-to-End Pipeline

### Happy Path UI
![Valid order before submit](docs/task7-valid-before-submit.png)

The valid order was submitted from the live Web App.

![Valid order running](docs/task7-valid-running.png)

The Web App shows the Durable orchestration running with an instance id.

![Valid order completed](docs/task7-valid-completed.png)

The valid order completed and returned a PDF report URL.

### Generated PDF Evidence
![PDF evidence](docs/task7-pdf-evidence.png)

The generated report for `ORD-001` contains the expected order information.

### Reject Path UI
![Reject order before submit](docs/task7-reject-before-submit.png)

The reject path test used quantity `999`, which is above the validator limit.

![Reject result](docs/task7-reject-result.png)

The Web App shows the order was rejected with reason `quantity exceeds limit`.

### Backend Participation Summary
![Task 7 CLI evidence](docs/task7-cli-evidence-summary.png)

This evidence summarizes Blob Storage output, AKS validator traffic, ACI state, and resource group participation for the end-to-end run.

### Resource Group Overview
![Resource group overview](docs/resource-group-overview.jpeg)

The resource group contains the deployed Azure resources used by the TaskFlow pipeline.

---

## Task 8: Write-up and Architecture Diagram

### Architecture Diagram
![Architecture diagram](docs/architecture-diagram.png)

The diagram shows GitHub CI/CD, App Service, Durable Function App, AKS validator, ACI report job, Blob Storage, ACR image pulls, and Managed Identity/IAM relationships.

### Service Selection

**App Service.** App Service is used for the frontend because it provides managed HTTPS web hosting with simple GitHub-based deployment. The Web App is a long-running user-facing application, so it needs a stable URL and low operational overhead. App Service is simpler than Kubernetes for this role and supports the Node frontend cleanly.

**Durable Functions.** Durable Functions are used for the orchestrator because the order workflow has multiple asynchronous steps. The orchestrator starts the workflow, calls validation, branches on the result, starts report generation, and exposes status polling to the frontend. Durable Functions provide checkpointing, persisted state, and status URLs, which would be difficult to implement safely with plain HTTP calls.

**AKS.** AKS hosts the validator because it is a long-running HTTP microservice with a stable endpoint. The validator is called for every order and runs continuously behind a Kubernetes LoadBalancer service. This demonstrates Kubernetes deployments, pods, services, image pull secrets, and microservice orchestration.

**ACI.** Azure Container Instances are used for report generation because the report job is short-lived and only needs to run per valid order. The container starts, generates a PDF, writes it to Blob Storage, and exits. This avoids keeping an always-running worker for a workload that only runs briefly.

### ACI vs AKS

AKS keeps a node running even when no orders are being processed, so idle AKS still has VM cost. ACI has no permanent service in this pipeline; a report container is created per run and exits after work is complete. If many users submit valid orders at once, ACI costs can grow with the number of report containers, while AKS continues to carry its baseline node cost for the validator.

Operationally, AKS is better for long-lived services that need Kubernetes primitives such as Deployments, Services, pods, and rolling updates. ACI is better for simple one-shot jobs where creating a full Kubernetes workload would add unnecessary operational overhead.

### Durable Functions vs Plain HTTP

A plain HTTP chain would make state management harder because the workflow includes validation, conditional branching, report job creation, polling, and final status. Durable Functions persist orchestration state, so the frontend can start a workflow and poll status instead of waiting on one long request. They also make failures easier to observe and recover from because each activity is tracked as part of the orchestration history.

### Cost Review

![Cost Analysis](docs/cost-analysis.png)

The Cost Analysis view is scoped to `rg-sp26-26100007` and shows a total cost of about `$2.46` for the resource group. The highest-cost resources are the App Service plans, followed by ACR, Storage, and AKS. ACI cost is lower for this workload because report containers run briefly and exit, while Blob Storage and ACR usage are small for this assignment.

### Challenges Faced

One issue was App Service quota: the required B1 Basic App Service Plan could not be created in the originally assigned region because the subscription had zero Basic VM quota there. I used the allowed fallback region, Sweden Central, and kept the main application resources there.

Another issue was Function App storage authentication. The subscription security policy blocked key-based storage authentication, so I configured the Function App to use the pre-provisioned managed identity with `AzureWebJobsStorage__accountName`, `AzureWebJobsStorage__credential=managedidentity`, and `AzureWebJobsStorage__clientId`.

A third issue was the manual ACI report job. PowerShell initially stripped quotes from `ORDER_JSON`, causing JSON parsing failures in the container. I fixed this by passing escaped JSON correctly, then enabled storage public network access while keeping key-based auth disabled so the managed-identity upload could reach Blob Storage.


