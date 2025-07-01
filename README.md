# KubeRay with TPUs User Guide

This page contains instructions for how to set up Ray on GKE with TPUs.

## Prerequisites

Please follow the official [Google Cloud documentation](https://cloud.google.com/tpu/docs/tpus-in-gke) for an introduction to TPUs. In particular, please ensure that your GCP project has sufficient quotas to provision the cluster, see [this link](https://cloud.google.com/tpu/docs/tpus-in-gke#ensure-quotas) for details.

For addition useful information about TPUs on GKE (such as topology configurations and availability), see [this page](https://cloud.google.com/kubernetes-engine/docs/concepts/tpus).

In addition, please ensure the following are installed on your local development environment:

* Helm (v3.9.3)
* Kubectl

## Version Compatibility with KubeRay

Here's which versions of this webhook are compatible with which versions of KubeRay. Reading from
the bottom, the KubeRay version stays the same in all subsequent webhook versions until the next
row's webhook version.

| webhook version | KubeRay version |
|-----------------|-----------------|
| unreleased      | 1.4.0           |
| 1.0.0           | 1.1.1           |

## Manually Installing the TPU Initialization Webhook

The TPU Initialization Webhook automatically bootstraps the TPU environment for TPU clusters. The webhook needs to be installed once per GKE cluster and requires a KubeRay Operator running v1.1+ and GKE cluster version of 1.28+. The webhook requires [cert-manager](https://github.com/cert-manager/cert-manager) to be installed in-cluster to handle TLS certificate injection. cert-manager can be installed in both GKE standard and autopilot clusters using the following helm commands:

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install --create-namespace --namespace cert-manager --set installCRDs=true --set global.leaderElection.namespace=cert-manager cert-manager jetstack/cert-manager
```

After installing cert-manager, it may take up to two minutes for the certificate to become ready.

Installing the webhook:

1. `git clone https://github.com/ai-on-gke/kuberay-tpu-webhook`
1. `cd kuberay-tpu-webhook`
1. `make deploy`
    1. this will create the webhook deployment, configs, and service in the "ray-system" namespace
    1. to change the namespace, edit the "namespace" value in each .yaml in deployments/ and certs/
1. `make deploy-cert`

The webhook can also be installed using the [Helm chart](https://github.com/ai-on-gke/kuberay-tpu-webhook/tree/main/helm-chart), enabling users to easily edit the webhook configuration. This helm package is stored on Artifact Registry and can be installed with the following commands:

1. Ensure you are authenticated with gcloud:
    1. `gcloud auth login`
    1. `gcloud auth configure-docker us-docker.pkg.dev`
1. `helm install kuberay-tpu-webhook oci://us-docker.pkg.dev/ai-on-gke/kuberay-tpu-webhook-helm/kuberay-tpu-webhook`

The above command can be edited with `-f` or `--set` flags to pass in a custom values file or key-value pair respectively for the chart (i.e. `--set tpuWebhook.image.tag=v1.2.3-gke.0`).

For common errors encountered when deploying the webhook, see the [Troubleshooting guide](https://github.com/ai-on-gke/kuberay-tpu-webhook/tree/main/Troubleshooting.md).

## Creating the KubeRay Cluster

You can find sample TPU cluster manifests for [single-host](https://github.com/ray-project/kuberay/blob/master/ray-operator/config/samples/ray-cluster.tpu-v4-singlehost.yaml) and [multi-host](https://github.com/ray-project/kuberay/blob/master/ray-operator/config/samples/ray-cluster.tpu-v4-multihost.yaml) here.

For a quick-start guide to using TPUs with KubeRay, see [Use TPUs with KubeRay](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/tpu.html).

## Running Sample Workloads

1. Save the following to a local file (e.g. `test_tpu.py`):

```python
import ray

ray.init(
    runtime_env={
        "pip": [
            "jax[tpu]",
            "-f https://storage.googleapis.com/jax-releases/libtpu_releases.html",
        ]
    }
)


@ray.remote(resources={"TPU": 4})
def tpu_cores():
    import jax
    return "TPU cores:" + str(jax.device_count())

num_workers = 4
result = [tpu_cores.remote() for _ in range(num_workers)]
print(ray.get(result))
```

1. `kubectl port-forward svc/RAYCLUSTER-NAME-head-svc dashboard &` where `RAYCLUSTER-NAME` is the
   `.metadata.name` of the RayCluster in the cluster manifest you used.
1. `RAY_ADDRESS=http://localhost:8265 ray job submit --runtime-env-json='{"working_dir": "."}' -- python test_tpu.py`

For a more advanced workload running Stable Diffusion on TPUs, see [here](https://cloud.google.com/kubernetes-engine/docs/add-on/ray-on-gke/tutorials/deploy-ray-serve-stable-diffusion-tpu). For an example of serving a LLM with TPUs, RayServe, and KubeRay, see [here](https://cloud.google.com/kubernetes-engine/docs/tutorials/serve-lllm-tpu-ray).
