# Deployment and DevOps Guide

This guide provides instructions for setting up the CI/CD pipeline, infrastructure, and monitoring for the Collaborative Travel Planning AI System. It is intended for an AI agent to execute.

## 1. Core Tools

- **CI/CD:** [Azure DevOps](https://azure.microsoft.com/en-us/services/devops/) or [GitHub Actions](https://github.com/features/actions).
- **Infrastructure as Code (IaC):** [Terraform](https://www.terraform.io/) or [Azure Bicep](https://docs.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview) to define and manage Azure resources.
- **Containerization:** [Docker](https://www.docker.com/).
- **Container Registry:** [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/).
- **Hosting:**
    -   **Frontend:** [Azure Static Web Apps](https://azure.microsoft.com/en-us/services/app-service/static/).
    -   **Backend:** [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service/) for the monolith and main services, and [Azure Container Apps](https://azure.microsoft.com/en-us/services/container-apps/) or [Azure Functions](https://azure.microsoft.com/en-us/services/functions/) for microservices.
- **Monitoring:** [Azure Monitor](https://azure.microsoft.com/en-us/services/monitor/), [Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview), and [Log Analytics](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-overview).

## 2. Infrastructure Provisioning (IaC)

Define all Azure resources using Terraform or Bicep. This ensures the infrastructure is version-controlled and repeatable.

**Resources to define:**
-   Azure Static Web App for the frontend.
-   Azure Kubernetes Service (AKS) cluster.
-   Azure Container Registry.
-   Azure Database for PostgreSQL.
-   Azure Cosmos DB.
-   Azure Web PubSub, Service Bus, Event Hubs.
-   Azure OpenAI and Cognitive Search services.
-   Azure Key Vault.
-   Azure API Management.
-   Log Analytics Workspace and Application Insights instances.

## 3. CI/CD Pipeline (Azure DevOps)

Create a multi-stage YAML pipeline in Azure DevOps.

### Stage 1: Build & Test

-   **Trigger:** On every push to `main` or `develop` branches, or on pull requests.
-   **Jobs:**
    1.  **Frontend Build:**
        -   Install Node.js and dependencies (`npm ci`).
        -   Run linter, unit tests, and code coverage.
        -   Publish test and coverage results.
        -   Build the Next.js application.
    2.  **Backend Build:**
        -   Install Node.js/Python and dependencies.
        -   Run linter, unit tests, and integration tests.
        -   Perform security scanning (e.g., Trivy for container images, CredScan for credentials).
        -   Build the Docker image for the backend service.
        -   Push the Docker image to Azure Container Registry.

### Stage 2: Deploy to Development

-   **Trigger:** After the Build stage succeeds on the `develop` branch.
-   **Environment:** An Azure DevOps environment named `Development` with approvals if necessary.
-   **Jobs:**
    1.  **Deploy Backend:**
        -   Use the Kubernetes manifest task to deploy the backend image to the development AKS cluster.
        -   Fetch secrets from the development Key Vault.
    2.  **Deploy Frontend:**
        -   The Azure Static Web Apps integration with Azure DevOps/GitHub will handle this automatically. A push to the `develop` branch can be configured to deploy to a staging environment.

### Stage 3: Deploy to Production

-   **Trigger:** After the Build stage succeeds on the `main` branch, and potentially after manual approval.
-   **Environment:** A `Production` environment with mandatory approvals.
-   **Strategy:** Use a canary or blue-green deployment strategy to minimize downtime and risk.
-   **Jobs:**
    1.  **Deploy Backend (Canary):**
        -   Deploy the new version to a small subset of pods in AKS.
        -   Run automated smoke tests or load tests against the canary deployment.
        -   Monitor key metrics (error rate, latency) from Application Insights.
        -   If metrics are healthy, gradually roll out the deployment to all pods.
        -   If issues are detected, automatically roll back.
    2.  **Deploy Frontend:**
        -   A push to the `main` branch will trigger the deployment to the production slot of the Azure Static Web App.

**Reference Code (`PLAN.md`):**
The `azure-pipelines.yml` file in `PLAN.md` provides a detailed template for this pipeline.

## 4. Monitoring and Alerting

-   **Application Insights:**
    -   Instrument both the frontend and backend code to send telemetry to Application Insights.
    -   Track custom events for key user actions (e.g., `TripCreated`, `BookingCompleted`).
    -   Track dependencies to monitor the performance of external services (e.g., Google Maps API, Azure OpenAI).
-   **Azure Monitor:**
    -   Create a unified dashboard in Azure Monitor that visualizes key metrics from all services:
        -   API performance (request rate, latency, error rate).
        -   Database health (CPU, memory, connection count).
        -   AI service usage and costs.
    -   Set up alerts for critical conditions:
        -   High API error rate (> 5%).
        -   High API latency (e.g., p95 > 2s).
        -   Database CPU at > 90% for a sustained period.
        -   AI service budget alerts.
-   **Logging:**
    -   All services should log structured JSON to stdout.
    -   Configure AKS to send container logs to the Log Analytics Workspace for centralized querying.
