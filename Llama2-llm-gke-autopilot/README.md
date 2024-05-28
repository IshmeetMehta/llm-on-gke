# llm-on-gke
# Serve Llama2-70b-chat-hf LLM with multiple GPUs on GKE Autopilot

This tutorial shows you how to serve a large language model (LLM) with GPUs in Google Kubernetes Engine (GKE). This tutorial creates a GKE cluster that uses multiple L4 GPUs or single/multiple H100 GPUs and prepares the GKE infrastructure to serve Llama2-70b-chat-hf LLM.

## Create a separate node pool for the GKE cluster
This step assumes you have created a cluster using terraform provided in the README.md or have an existing cluster.

## Setup

1. Configure kubectl to communicate with your cluster:
````bash
gcloud container clusters get-credentials <cluster-name> --region=${REGION}
````

2. Prepare your workload

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

    c. Choose and create the following text-generation-inference.yaml manifest depending on accerator-type available for your project and quotas:
    
    # Accelerator-type nvindia-L4

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
                - name: NUM_SHARD
                value: "2"
                - name: PORT
                value: "8080"
                - name: QUANTIZE
                value: bitsandbytes-nf4
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
                cloud.google.com/gke-accelerator: "nvidia-l4" 
                cloud.google.com/gke-spot: "true"
            ````

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

NUM_SHARD must be 2 because the model requires atleast two NVIDIA L4 GPUs.
QUANTIZE is set to bitsandbytes-nf4 for the accelerator type nvindia-L4 which means that the model is loaded in 4 bit instead of 32 bits. This allows GKE to reduce the amount of GPU memory needed and improves the inference speed. However, the model accuracy can decrease. To learn how to calculate the GPUs to request, see Calculating the amount of GPUs.

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

