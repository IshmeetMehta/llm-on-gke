# llm-on-gke
# Running Opensource LLMs on Google Kubernetes Engine (GKE)

This repository contains assets related to OpenSource LLMs with Google Kubernetes Engineon
[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/docs/integrations/ai-infra).

## Overview

Run optimized OpenSource LLMs with Google Kubernetes Engine (GKE) platform orchestration capabilities. A robust AI/ML platform considers the following layers:

- Infrastructure orchestration that support GPUs and TPUs for Inference and serving workloads at scale
- Flexible integration with distributed computing and data processing frameworks
-

## Infrastructure

The llm-on-gke requires you to create a functional GKE cluster. You can follow the instructions under [infrastructure/README.md](./infrastructure/README.md) to install a Standard or Autopilot GKE cluster.

```bash
.
├── LICENSE
├── README.md
├── infrastructure
│   ├── README.md
│   ├── backend.tf
│   ├── main.tf
│   ├── outputs.tf
│   ├── platform.tfvars
│   ├── variables.tf
│   └── versions.tf
├── modules
│   ├── gke-autopilot-private-cluster
│   ├── gke-autopilot-public-cluster
│   ├── gke-standard-private-cluster
│   ├── gke-standard-public-cluster
└── tutorial.md
```

To deploy new GKE cluster update the `platform.tfvars` file with the appropriate values and then execute below terraform commands:
```
terraform init
terraform apply -var-file platform.tfvars
```


## Applications


## Important Considerations
- Make sure to configure terraform backend to use GCS bucket, in order to persist terraform state across different environments.


## Licensing

* The use of the assets contained in this repository is subject to compliance with [Google's AI Principles](https://ai.google/responsibility/principles/)
* See [LICENSE](/LICENSE)
