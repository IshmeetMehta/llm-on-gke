# llm-on-gke
# Serve Llama2-70b-chat-hf LLM with multiple GPUs on GKE Standard with DWS


This tutorial shows you how to serve a large language model (LLM) with GPUs in Google Kubernetes Engine (GKE). This tutorial creates a GKE cluster that uses multiple L4 GPUs and specifc node-pools with a single or multiple H100 GPUs and prepares the GKE infrastructure to serve Llama2-70b-chat-hf LLM.

## Create a separate node pool for the GKE cluster with DWS
This step assumes you have created a cluster using terraform provided in the README.md or have an existing cluster.

Create a node pool with Dynamic Workload Scheduler enabled using the gcloud CLI:

    ````bash
    gcloud beta container node-pools create NODEPOOL_NAME \
    --cluster=CLUSTER_NAME \
    --location=LOCATION \
     --enable-queued-provisioning \
    --accelerator type=GPU_TYPE,count=AMOUNT,gpu-driver-version=DRIVER_VERSION \
    --machine-type=MACHINE_TYPE \
    --enable-autoscaling  \
    --num-nodes=0   \
    --total-max-nodes TOTAL_MAX_NODES  \
    --location-policy=ANY  \
    --reservation-affinity=none  \
    --no-enable-autorepair
    `
Replace the following:
````
    NODEPOOL_NAME: The name you choose for the node pool.
    CLUSTER_NAME: The name of the cluster.
    LOCATION: The cluster's Compute Engine region, such as us-central1.
    GPU_TYPE: The GPU type.
    AMOUNT: The number of GPUs to attach to nodes in the node pool.
    DRIVER_VERSION: the NVIDIA driver version to install. Can be one of the following:
    default: Install the default driver version for your GKE version.
    latest: Install the latest available driver version for your GKE version. Available only for nodes that use Container-Optimized OS.
    TOTAL_MAX_NODES: the maximum number of nodes to automatically scale for the entire node pool.
    MACHINE_TYPE: The Compute Engine machine type for your nodes. We recommend that you select an Accelerator-optimized machine type.[ for example nvidia-h100]
````

*Please check availability of the requested GPU type in the region*



## Setup Dynamic Workload Scheduler for Jobs without Kueue

1. Create a provisioing request for your cluster:

        ````yaml
        apiVersion: v10
        kind: PodTemplate
        metadata:
        name: POD_TEMPLATE_NAME
        namespace: NAMESPACE_NAME
        template:
        spec:
            nodeSelector:
                cloud.google.com/gke-nodepool: NODEPOOL_NAME
            tolerations:
                - key: "nvidia.com/gpu"
                operator: "Exists"
                effect: "NoSchedule"
            containers:
                - name: pi
                image: perl
                command: ["/bin/sh"]
                resources:
                    limits:
                    cpu: "700m"
                    nvidia.com/gpu: 1
                    requests:
                    cpu: "700m"
                    nvidia.com/gpu: 1
            restartPolicy: Never
        ---
        apiVersion: autoscaling.x-k8s.io/v1beta1
        kind: ProvisioningRequest
        metadata:
        name: PROVISIONING_REQUEST_NAME
        namespace: NAMESPACE_NAME
        spec:
        provisioningClassName: queued-provisioning.gke.io
        parameters:
            maxRunDurationSeconds: "MAX_RUN_DURATION_SECONDS"
        podSets:
        - count: COUNT
            podTemplateRef:
            name: POD_TEMPLATE_NAME

        ````
    
    * Look for an example provisiong request for in this folder*

2. Apply the provisiong manifest:

        ````bash
        kubectl apply -f <provisioning-request-manifest>
        `

    a. Set the default environment variables:

    ````bash
    export HF_TOKEN=HUGGING_FACE_TOKEN
    ````
    Replace the ```HUGGING_FACE_TOKEN``` with your HuggingFace toke

    b. Create a Kubernetes secret for the HuggingFace token:

     ````bash
    kubectl create secret generic <cluster-name> \
    --from-literal=HUGGING_FACE_TOKEN=${HF_TOKEN} \
    --dry-run=client -o yaml | kubectl apply -f -
    ````

    c. Choose and create the following text-generation-inference.yaml manifest for accerator-type(nvidia-h100) available for your project and quotas:
    Use the provisioning request name to create the manifest.
    
    # Accelerator-type nvindia-h100
        ````yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: llm
        spec:
        replicas: 1
        selector:
            matchLabels:
            app: llm
        template:
            metadata:
            labels:
                app: llm
            spec:
            containers:
            - name: llm
                image: ghcr.io/huggingface/text-generation-inference:1.4.3
                resources:
                requests:
                    cpu: "10"
                    memory: "60Gi"
                    nvidia.com/gpu: "2"
                limits:
                    cpu: "10"
                    memory: "60Gi"
                    nvidia.com/gpu: "2"
                env:
                - name: MODEL_ID
                value: meta-llama/Llama-2-70b-chat-hf
                - name: PORT
                value: "8080"
                - name: HUGGING_FACE_HUB_TOKEN
                valueFrom:
                    secretKeyRef:
                    name: <secret-name>
                    key: HUGGING_FACE_TOKEN
                volumeMounts:
                - mountPath: /dev/shm
                    name: dshm
                - mountPath: /data
                    name: ephemeral-volume
            volumes:
                - name: dshm
                emptyDir:
                    medium: Memory
                - name: ephemeral-volume
                ephemeral:
                    volumeClaimTemplate:
                    metadata:
                        labels:
                        type: ephemeral
                    spec:
                        accessModes: ["ReadWriteOnce"]
                        storageClassName: "premium-rwo"
                        resources:
                        requests:
                            storage: 150Gi
            nodeSelector:
                cloud.google.com/gke-accelerator: "nvidia-h100"
                cloud.google.com/gke-spot: "true"
        ````

In above manifests:

For H100, we donot need to QUANTIZE the model as the accelerator type has total memory of 80GB which is sufficient for the model.

3. Apply the manifest:

    ````bash
    kubectl apply -f text-generation-inference.yaml
    ````
    The output is similar to the following:

    ````
    deployment.apps/llm created
    ````
4. Verify the status of the model:

    ````bash
    kubectl get deploy
    `   

    The output is similar to the following:

        ````
        NAME          READY   UP-TO-DATE   AVAILABLE   AGE
        llm           1/1     1            1           20m
        ````
5. View the logs from the running deployment:

    ````bash
    kubectl logs -l app=llm
    `
    The output is similar to the following:

        ````
        {"timestamp":"2024-03-09T05:08:14.751646Z","level":"INFO","message":"Warming up model","target":"text_generation_router","filename":"router/src/main.rs","line_number":291}
        {"timestamp":"2024-03-09T05:08:19.961136Z","level":"INFO","message":"Setting max batch total tokens to 133696","target":"text_generation_router","filename":"router/src/main.rs","line_number":328}
        {"timestamp":"2024-03-09T05:08:19.961164Z","level":"INFO","message":"Connected","target":"text_generation_router","filename":"router/src/main.rs","line_number":329}
        {"timestamp":"2024-03-09T05:08:19.961171Z","level":"WARN","message":"Invalid hostname, defaulting to 0.0.0.0","target":"text_generation_router","filename":"router/src/main.rs","line_number":343}
        ````
6. Create a Service of type ClusterIP or LoadBalancer and apply the manifest

    Create the following llm-service.yaml manifest

        ````yaml
        apiVersion: v1
        kind: Service
        metadata:
        name: llm-service
        spec:
        selector:
            app: llm
        type: ClusterIP # Replace it with LoadBalancer if you want to expose the service externally
        ports:
            - protocol: TCP
            port: 80
            targetPort: 8080
        ````
    Apply the manifest:

        ````bash
        kubectl apply -f llm-service.yaml
        `
    The output is similar to the following:

    ````
    service.v1.core created
    `   

## Deploy Gradio chat interface to interact with the deployed model

Gradio is a Python library that has a ChatInterface wrapper that creates user interfaces for chatbots.

    1. Create and apply the following gradio-deployment.yaml manifest

        ````yaml
        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: gradio
        labels:
            app: gradio
        spec:
        strategy: 
            type: Recreate
        replicas: 1
        selector:
            matchLabels:
            app: gradio
        template:
            metadata:
            labels:
                app: gradio
            spec:
            containers:
            - name: gradio
                image: us-docker.pkg.dev/google-samples/containers/gke/gradio-app:v1.0.0
                resources:
                requests:
                    cpu: "512m"
                    memory: "512Mi"
                limits:
                    cpu: "1"
                    memory: "512Mi"
                env:
                - name: CONTEXT_PATH
                value: "/generate"
                - name: HOST
                value: "http://llm-service"
                - name: LLM_ENGINE
                value: "tgi"
                - name: MODEL_ID
                value: "llama-2-70b"
                - name: USER_PROMPT
                value: "[INST] prompt [/INST]"
                - name: SYSTEM_PROMPT
                value: "prompt"
                ports:
                - containerPort: 7860
        ---
        apiVersion: v1
        kind: Service
        metadata:
        name: gradio-service
        spec:
        type: LoadBalancer
        selector:
            app: gradio
        ports:
        - port: 80
            targetPort: 7860
        ````

    Apply the manifest:

    ````bash
        kubectl apply -f gradio.yaml
    `

    Find the external IP address of the Service:
    
    ````bash
        kubectl get svc -o wide
    `
    The output is similar to the following:

    ````
    NAME             TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
    gradio-service   LoadBalancer   10.24.29.197   34.172.115.35   80:30952/TCP   125m
    ````

    View the model interface from your web browser by using the external IP address with the exposed port:

    ````bash
        curl http://<external_ip_address>:80
    `

