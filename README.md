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


## Calculate GPU requirements

In order to install OpenSource models such as
[Lllama-2-70b-chat-hf](https://huggingface.co/meta-llama/Llama-2-70b-chat-hf) we need to calculate the GPU requirements.

The amount of GPUs depends on the value of the QUANTIZE flag. In this tutorial, QUANTIZE is set to bitsandbytes-nf4, which means that the model is loaded in 4 bits.

A 70 billion parameter model would require a minimum of 40 GB of GPU memory which equals to 70 billion times 4 bits (70 billion x 4 bits= 35 GB) and considers a 5 GB of overhead. In this case, a single L4 GPU wouldn't have enough memory. Therefore, the examples in this tutorial use two L4 GPU of memory (2 x 24 = 48 GB). This configuration is sufficient for running Falcon 40b or Llama 2 70b in L4 GPUs.

If L4 GPUs are not available or do not meet your performance requirement. Llama2 is known to perform better on H100.Tutorial also provides an example on how to use H100 GPU with [Dynamic Workload Scheduler(DWS)](https://cloud.google.com/blog/products/compute/introducing-dynamic-workload-scheduler)

Please review the section below to check GPU availability and machine sizes and capacity per region.


## GPU Capacity and Availability

[Choose GPU support for your region and GKE cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/gpus)


## Important Considerations
- Make sure to configure terraform backend to use GCS bucket, in order to persist terraform state across different environments.


## Licensing

* The use of the assets contained in this repository is subject to compliance with [Google's AI Principles](https://ai.google/responsibility/principles/)
* See [LICENSE](/LICENSE)
