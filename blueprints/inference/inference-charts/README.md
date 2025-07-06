# AI on EKS Inference Charts

Chart Name: `ai-on-eks-inference-charts`

This Helm chart provides deployment configurations for AI/ML inference workloads on both GPU and AWS Neuron (
Inferentia/Trainium) hardware.

## Overview

The chart supports the following deployment types:

- GPU-based VLLM deployments
- GPU-based Ray-VLLM deployments
- Neuron-based VLLM deployments
- Neuron-based Ray-VLLM deployments

### VLLM vs Ray-VLLM

**VLLM Deployments** (`framework: vllm`):

- Direct VLLM deployment using Kubernetes Deployment
- Simpler architecture, faster startup
- Uses `vllm/vllm-openai` image
- Suitable for single-node inference

**Ray-VLLM Deployments** (`framework: rayVllm`):

- VLLM deployed on Ray Serve for distributed inference
- More complex architecture with head and worker nodes
- Uses `rayproject/ray` image
- Supports autoscaling and distributed workloads
- Includes observability integration with Prometheus and Grafana
- Requires additional parameters: `rayVersion`, `vllmVersion`, `pythonVersion`

## Prerequisites

- Kubernetes cluster with GPU or AWS Neuron nodes
- Helm 3.0+
- For GPU deployments: NVIDIA device plugin installed
- For Neuron deployments: AWS Neuron device plugin installed
- Hugging Face Hub token (stored as a Kubernetes secret named `hf-token`)

## Installation

### Create Hugging Face Token Secret

Before installing the chart, create a Kubernetes secret with your Hugging Face token:

```bash
kubectl create secret generic hf-token --from-literal=token=your_huggingface_token
```

## Configuration

The following table lists the configurable parameters of the inference-charts chart and their default values.

| Parameter                                                                | Description                             | Default                                                                     |
|--------------------------------------------------------------------------|-----------------------------------------|-----------------------------------------------------------------------------|
| `global.image.pullPolicy`                                                | Global image pull policy                | `IfNotPresent`                                                              |
| `inference.accelerator`                                                  | Accelerator type to use (gpu or neuron) | `gpu`                                                                       |
| `inference.framework`                                                    | Framework type to use (vllm or rayVllm) | `vllm`                                                                      |
| `inference.serviceName`                                                  | Name of the inference service           | `inference`                                                                 |
| `inference.serviceNamespace`                                             | Namespace for the inference service     | `default`                                                                   |
| `inference.modelServer.image.repository`                                 | Model server image repository           | `vllm/vllm-openai`                                                          |
| `inference.modelServer.image.tag`                                        | Model server image tag                  | `latest`                                                                    |
| `inference.modelServer.vllmVersion`                                      | VLLM version (for Ray deployments)      | Not set                                                                     |
| `inference.modelServer.pythonVersion`                                    | Python version (for Ray deployments)    | Not set                                                                     |
| `inference.modelServer.deployment.replicas`                              | Number of replicas                      | `1`                                                                         |
| `inference.modelServer.deployment.minReplicas`                           | Minimum number of replicas (for Ray)    | `1`                                                                         |
| `inference.modelServer.deployment.maxReplicas`                           | Maximum number of replicas (for Ray)    | `2`                                                                         |
| `inference.modelServer.deployment.instanceType`                          | Node selector for instance type         | Not set                                                                     |
| `inference.modelServer.deployment.topologySpreadConstraints.enabled`     | Enable topology spread constraints      | `true`                                                                      |
| `inference.modelServer.deployment.topologySpreadConstraints.constraints` | List of topology spread constraints     | See default configuration                                                   |
| `inference.modelServer.deployment.podAffinity.enabled`                   | Enable pod affinity                     | `true`                                                                      |
| `inference.rayOptions.rayVersion`                                        | Ray version to use                      | `2.47.0`                                                                    |
| `inference.rayOptions.autoscaling.enabled`                               | Enable Ray native autoscaling           | `false`                                                                     |
| `inference.rayOptions.autoscaling.upscalingMode`                         | Ray autoscaler upscaling mode           | `Default`                                                                   |
| `inference.rayOptions.autoscaling.idleTimeoutSeconds`                    | Idle timeout before scaling down        | `60`                                                                        |
| `inference.rayOptions.autoscaling.actorAutoscaling.minActors`            | Minimum number of actors                | `1`                                                                         |
| `inference.rayOptions.autoscaling.actorAutoscaling.maxActors`            | Maximum number of actors                | `1`                                                                         |
| `inference.rayOptions.observability.rayPrometheusHost`                   | Ray Prometheus host URL                 | `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090` |
| `inference.rayOptions.observability.rayGrafanaHost`                      | Ray Grafana host URL                    | `http://kube-prometheus-stack-grafana.monitoring.svc.cluster.local`         |
| `inference.rayOptions.observability.rayGrafanaIframeHost`                | Ray Grafana iframe host URL             | `http://localhost:3000`                                                     |
| `vllm.logLevel`                                                          | Log level for VLLM                      | `debug`                                                                     |
| `vllm.port`                                                              | VLLM server port                        | `8004`                                                                      |
| `service.type`                                                           | Service type                            | `ClusterIP`                                                                 |
| `service.port`                                                           | Service port                            | `8000`                                                                      |
| `fluentbit.image.repository`                                             | Fluent Bit image repository             | `fluent/fluent-bit`                                                         |
| `fluentbit.image.tag`                                                    | Fluent Bit image tag                    | `3.2.2`                                                                     |

### Model Parameters

The chart provides configuration for various model parameters:

| Parameter                                   | Description                      | Default                     |
|---------------------------------------------|----------------------------------|-----------------------------|
| `modelParameters.modelId`                   | Model ID from Hugging Face Hub   | `NousResearch/Llama-3.2-1B` |
| `modelParameters.gpuMemoryUtilization`      | GPU memory utilization           | `0.8`                       |
| `modelParameters.maxModelLen`               | Maximum model sequence length    | `8192`                      |
| `modelParameters.maxNumSeqs`                | Maximum number of sequences      | `4`                         |
| `modelParameters.maxNumBatchedTokens`       | Maximum number of batched tokens | `8192`                      |
| `modelParameters.tokenizerPoolSize`         | Tokenizer pool size              | `4`                         |
| `modelParameters.maxParallelLoadingWorkers` | Maximum parallel loading workers | `2`                         |
| `modelParameters.pipelineParallelSize`      | Pipeline parallel size           | `1`                         |
| `modelParameters.tensorParallelSize`        | Tensor parallel size             | `1`                         |
| `modelParameters.enablePrefixCaching`       | Enable prefix caching            | `true`                      |
| `modelParameters.numGpus`                   | Number of GPUs to use            | `1`                         |

**Note**: Model parameters are automatically converted to environment variables in SCREAMING_SNAKE_CASE format (e.g.,
`modelId` becomes `MODEL_ID`, `maxNumSeqs` becomes `MAX_NUM_SEQS`).

## Supported Models

The chart includes pre-configured values files for the following models:

### GPU Models

- **DeepSeek R1 Distill Llama 8B**: `values-deepseek-r1-distill-llama-8b-ray-vllm-gpu.yaml` (Ray-VLLM)
- **Llama 3.2 1B**: `values-llama-32-1b-vllm.yaml` (VLLM) and `values-llama-32-1b-ray-vllm.yaml` (Ray-VLLM)
- **Llama 4 Scout 17B**: `values-llama-4-scout-17b-vllm.yaml` (VLLM)
- **Mistral Small 24B**: `values-mistral-small-24b-ray-vllm.yaml` (Ray-VLLM)

### Neuron Models

- **DeepSeek R1 Distill Llama 8B**: `values-deepseek-r1-distill-llama-8b-vllm-neuron.yaml` (VLLM)
- **Llama 2 13B**: `values-llama-2-13b-ray-vllm-neuron.yaml` (Ray-VLLM)
- **Llama 3 70B**: `values-llama-3-70b-ray-vllm-neuron.yaml` (Ray-VLLM)
- **Llama 3.1 8B**: `values-llama-31-8b-vllm-neuron.yaml` (VLLM) and `values-llama-31-8b-ray-vllm-neuron.yaml` (
  Ray-VLLM)

## Topology Spread Constraints

The chart includes optional topology spread constraints to control how pods are distributed across your cluster. By
default, the chart is configured to prefer scheduling replicas in the same availability zone for reduced network latency
and cost optimization.

### Default Configuration

```yaml
inference:
  modelServer:
    deployment:
      topologySpreadConstraints:
        enabled: true
        constraints:
          # Prefer same AZ as head pod (soft constraint)
          - maxSkew: 1
            topologyKey: topology.kubernetes.io/zone
            whenUnsatisfiable: ScheduleAnyway
            labelSelector:
              matchLabels: { }
          # Require workers to be grouped together (hard constraint)
          - maxSkew: 1
            topologyKey: topology.kubernetes.io/zone
            whenUnsatisfiable: DoNotSchedule
            labelSelector:
              matchLabels: { }
      podAffinity:
        enabled: true
        # Strong preference for same AZ (helps Karpenter understand intent)
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: topology.kubernetes.io/zone
              labelSelector:
                matchLabels: { }
```

**Note**: For Ray deployments, the default configuration uses two constraints:

1. **Head Co-location**: Workers prefer to be in the same AZ as the head pod (soft constraint)
2. **Worker Grouping**: All worker pods must be scheduled together in the same AZ (hard constraint)

This ensures optimal performance while maintaining high availability - workers will try to co-locate with the head, but
if that's not possible, they'll at least be grouped together for consistent inter-worker communication.

### Disabling Topology Constraints

To disable topology spread constraints entirely:

```yaml
inference:
  modelServer:
    deployment:
      topologySpreadConstraints:
        enabled: false
      podAffinity:
        enabled: false
```

### Ray-Specific Behavior

For Ray deployments (`framework: rayVllm`), the topology constraints work differently:

- **Head Group**: Uses the first constraint to establish zone preference
- **Worker Group**: Uses both constraints:
    1. First constraint (soft): Tries to co-locate with head pod
    2. Second constraint (hard): Ensures all workers are grouped together

**Scheduling Logic**:

1. Head pod schedules in any available zone
2. Workers try to schedule in the same zone as head
3. If head's zone is full, workers schedule together in another zone
4. Workers are never split across multiple zones

### Karpenter Compatibility

The chart uses **both topology spread constraints and pod affinity** for Karpenter compatibility:

- **Topology Spread Constraints**: Control pod distribution at the scheduler level
- **Pod Affinity**: Help Karpenter understand co-location intent during node provisioning

**Troubleshooting Steps**:

1. **Soft constraints first**: Always start with `whenUnsatisfiable: ScheduleAnyway`
2. **Check node availability**: Verify nodes exist in your target AZ
3. **Monitor Karpenter logs**: Check why it's provisioning in different AZs

## Ray Native Autoscaling

For Ray-VLLM deployments, you can enable Ray's native autoscaling feature which automatically scales worker nodes based
on workload demand. This is more efficient than Kubernetes HPA as it understands Ray's internal workload distribution.

### Autoscaling Configuration

| Parameter                                                     | Description                                     | Default   |
|---------------------------------------------------------------|-------------------------------------------------|-----------|
| `inference.rayOptions.autoscaling.enabled`                    | Enable Ray native autoscaling                   | `false`   |
| `inference.rayOptions.autoscaling.upscalingMode`              | Ray autoscaler upscaling mode                   | `Default` |
| `inference.rayOptions.autoscaling.idleTimeoutSeconds`         | How long to wait before scaling down idle nodes | `60`      |
| `inference.rayOptions.autoscaling.actorAutoscaling.minActors` | Minimum number of actors                        | `1`       |
| `inference.rayOptions.autoscaling.actorAutoscaling.maxActors` | Maximum number of actors                        | `1`       |

### Example Autoscaling Configuration

```yaml
inference:
  framework: rayVllm
  rayOptions:
    autoscaling:
      enabled: true
      upscalingMode: "Aggressive"
      idleTimeoutSeconds: 120  # Wait 2 minutes before scaling down
      actorAutoscaling:
        minActors: 1
        maxActors: 5
```

## Examples

### Deploy GPU Ray-VLLM with DeepSeek R1 Distill Llama 8B model

```bash
helm install deepseek-gpu-inference ./inference-charts --values values-deepseek-r1-distill-llama-8b-ray-vllm-gpu.yaml
```

### Deploy GPU VLLM with Llama 3.2 1B model

```bash
helm install gpu-vllm-inference ./inference-charts --values values-llama-32-1b-vllm.yaml
```

### Deploy GPU Ray-VLLM with Llama 3.2 1B model

```bash
helm install gpu-ray-vllm-inference ./inference-charts --values values-llama-32-1b-ray-vllm.yaml
```

### Deploy Neuron VLLM with DeepSeek R1 Distill Llama 8B model

```bash
helm install deepseek-neuron-inference ./inference-charts --values values-deepseek-r1-distill-llama-8b-vllm-neuron.yaml
```

### Deploy Neuron Ray-VLLM with Llama 2 13B model

```bash
helm install llama2-neuron-inference ./inference-charts --values values-llama-2-13b-ray-vllm-neuron.yaml
```

### Deploy Neuron Ray-VLLM with Llama 3 70B model

```bash
helm install llama3-70b-neuron-inference ./inference-charts --values values-llama-3-70b-ray-vllm-neuron.yaml
```

### Deploy Neuron VLLM with Llama 3.1 8B model

```bash
helm install neuron-vllm-inference ./inference-charts --values values-llama-31-8b-vllm-neuron.yaml
```

### Deploy Neuron Ray-VLLM with Llama 3.1 8B model

```bash
helm install neuron-ray-vllm-inference ./inference-charts --values values-llama-31-8b-ray-vllm-neuron.yaml
```

### Deploy GPU Ray-VLLM with Mistral Small 24B model

```bash
helm install gpu-ray-vllm-mistral ./inference-charts --values values-mistral-small-24b-ray-vllm.yaml
```

### Custom Deployment

You can also create your own values file with custom settings:

```yaml
inference:
  accelerator: gpu  # or neuron
  framework: vllm   # or rayVllm
  serviceName: custom-inference
  serviceNamespace: default

  # Ray-specific options (only for rayVllm framework)
  rayOptions:
    rayVersion: 2.47.0
    autoscaling:
      enabled: false
      upscalingMode: "Default"
      idleTimeoutSeconds: 60
      actorAutoscaling:
        minActors: 1
        maxActors: 1
    observability:
      rayPrometheusHost: http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
      rayGrafanaHost: http://kube-prometheus-stack-grafana.monitoring.svc.cluster.local
      rayGrafanaIframeHost: http://localhost:3000

  modelServer:
    image:
      repository: vllm/vllm-openai
      tag: latest
    deployment:
      replicas: 1
      minReplicas: 1
      maxReplicas: 2
      resources:
        gpu:
          requests:
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1

modelParameters:
  modelId: "NousResearch/Llama-3.2-1B"
  gpuMemoryUtilization: 0.8
  maxModelLen: 8192
  maxNumSeqs: 4
  maxNumBatchedTokens: 8192
  tokenizerPoolSize: 4
  maxParallelLoadingWorkers: 2
  pipelineParallelSize: 1
  tensorParallelSize: 1
  enablePrefixCaching: true
  numGpus: 1
```

Then install the chart with your custom values:

```bash
helm install custom-inference ./inference-charts --values custom-values.yaml
```

## API Endpoints

The deployed service exposes the following OpenAI-compatible API endpoints:

- `/v1/models` - List available models
- `/v1/completions` - Text completion API
- `/v1/chat/completions` - Chat completion API
- `/metrics` - Prometheus metrics endpoint

## Observability

The chart includes Fluent Bit for log collection and exposes Prometheus metrics for monitoring. The Ray-VLLM deployment
also includes configuration for Grafana dashboards.
