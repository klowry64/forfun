---
description: >-
  Use Argo Workflows to run a full pipeline which ends with distributed
  finetuning of GPT-NeoX-20B.
---

# Finetune GPT-NeoX-20B with Argo Workflows

Similarly to the [Finetuning Machine Learning Models](../finetuning-machine-learning-models.md) tutorial, the following walkthrough provides an example of using Argo Workflows to finetune a smaller model (GPT-J) on a smaller dataset. If you are new to finetuning and Argo Workflows, this is a great place to start.

This example uses two A100 nodes (16 total GPUs) using NVIDIA's [NVLINK](https://www.nvidia.com/en-us/data-center/nvlink/) and [Infiniband](https://www.nvidia.com/en-us/networking/products/infiniband/) technologies for highly performant distributed training.

## Source code

Throughout the rest of this document, referenced files may be found in CoreWeave's `kubernetes-cloud` repo in the `kubeflow/training-operator/gpt-neox` folder.

{% embed url="https://github.com/coreweave/kubernetes-cloud/tree/master/kubeflow/training-operator/gpt-neox" %}
Link to the source code referenced in this example.
{% endembed %}

In this folder, you will find numbered YAML files:

```
01-pvc.yaml
02-finetune-role.yaml
03-wanbd-secret.yaml
04-finetune-workflow.yaml
```

These should be deployed in the order in which they are numbered. The first three YAML files (`01-pvc.yaml`, `02-finetune-role.yaml`, and `03-wanbd-secret.yaml`) deploy the Kubernetes resources that are required by the Argo workflow, which is deployed using the fourth file (`04-finetune-workflow.yaml`).

## Understanding the Finetune Workflow

The Argo workflow for finetuning is defined in the `04-finetune-workflow.yaml` file. This file consists of three important sections:

* workflow parameters,
* [the directed-acyclic graph (DAG) definition](https://argoproj.github.io/argo-workflows/walk-through/dag/#dag),
* and the step definitions.

<figure><img src="../../.gitbook/assets/image (3) (1).png" alt="Visualization of the DAG defined by the Argo workflow"><figcaption><p>Visualization of the DAG defined by the Argo workflow</p></figcaption></figure>

### Parameters

At the top of the `04-finetune-workflow.yaml` file, all of the Workflow parameters and their default values are defined. If none of these are changed, the Workflow will download and tokenize the [Hackernews subset of The Pile dataset](https://news.ycombinator.com/item?id=25607809) before finetuning GPT-NeoX-20B on it using two `A100_NVLINK_80GB` nodes in the `LAS1` region.

If you have already downloaded the model weights, dataset, and/or tokenized the dataset, you can skip those steps by changing the respective parameters to `false`:

* `download_checkpoint`
* `download_dataset`
* `tokenize_dataset`

{% hint style="info" %}
**Note**

The screenshot above shows these steps being skipped.
{% endhint %}

The finetuning stage uses [Weights and Biases](https://wandb.ai/site) for extra logging. This is controlled by the three `wandb` parameters. To avoid using Weights and Biases, you can set the values of these parameters to an empty string.

### DAG definition

The DAG is defined in the Workflow file under the `main` template. The graph is organized based on the dependencies of all the stages. For example, the two download stages don't have any dependencies, so they both run in parallel right away.

{% hint style="info" %}
**Note**

It is not demonstrated in this example, but it is possible to have more advanced logic around dependencies, such as running certain stages based on the completion status of another stage.
{% endhint %}

### Source code

The code that performs the dataset tokenization and finetuning is straight from [Eluther AI's `gpt-neox` repo](https://github.com/EleutherAI/gpt-neox). This code is built into the Docker image that is built from [CoreWeave's `ml-containers` repo](https://github.com/coreweave/ml-containers).

### Training config

The config file containing all of the training-related parameters is stored within [a Kubernetes ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/). If you would like to adjust these parameters, the ConfigMap can be edited within the `04-finetune-workflow.yaml` file in the `create-configmap` stage definition.

### Finetuning resources

By default, this Workflow uses two A100 nodes with NVLINK and RDMA over InfiniBand for extremely fast distributed training. These resources are defined in the `Worker` spec of the MPIJob at the bottom of the Workflow's YAML manifest:

```yaml
resources:
  requests:
    cpu: 48
    memory: 256Gi
    nvidia.com/gpu: 8
    rdma/ib: 1
  limits:
    nvidia.com/gpu: 8
    rdma/ib: 1
```

### RDMA over InfiniBand

As is shown in this example, the `rdma/ib` resource requests that Remote Direct Memory Access (RDMA) be performed using InfiniBand. RDMA allows for direct memory access from the memory of one computer into that of another without involving the Operating System of either machine, which is accomplished using InfiniBand packets over Ethernet.

Requesting this resource offers a big boost to distributed training performance, however it is currently **only available for A100 and H100 GPU node types on CoreWeave Cloud.**

<table data-view="cards"><thead><tr><th></th><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td>➡️ <strong>Learn more about InfiniBand on CoreWeave Cloud</strong></td><td></td><td></td><td><a href="../../coreweave-kubernetes/networking/hpc-interconnect.md">hpc-interconnect.md</a></td></tr></tbody></table>

## Setup

Before running the Workflow, a few things need to be created in your namespace.&#x20;

{% hint style="info" %}
**Note**

This guide assumes that you have already followed the process to set up the CoreWeave Kubernetes environment. If you have not done so already, [follow our Getting Started guide](../../coreweave-kubernetes/getting-started/) before proceeding with this guide.
{% endhint %}

### Argo Workflows

To run an Argo workflow, first deploy the Argo Workflows application in your namespace via the CoreWeave's [application Catalog](../../coreweave-kubernetes/serverless/applications-catalog.md).

{% hint style="info" %}
**Additional Information**

For more information on Argo Workflows, see [Workflows](broken-reference).
{% endhint %}

### PVC

The Argo workflow uses Persistent Volume Claim's (PVCs) to store the dataset and model checkpoints. The PVCs are defined in `01-pvc.yaml` and be deployed with `kubectl`:

```bash
kubectl apply -f 01-pvc.yaml
```

{% hint style="info" %}
**Optional**

You can deploy [a FileBrowser application](../../storage/filebrowser.md) attaching the newly created PVCs to be able to inspect their contents in your browser.
{% endhint %}

### Finetune role

This Argo Workflow involves creating new Kubernetes resources in your environment: [a ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/), and an MPIJob. In order to do create these, the Workflow needs to run as a Service Account with the necessary permissions granted to it.

The `finetune` service account and its permissions that will later be used by the workflow is defined in the `02-finetune-role.yaml`. Apply it to your namespace with `kubectl`:

```bash
kubectl apply -f 02-finetune-role.yaml
```

### Weights and Biases secret

If you would like to take advantage of Weights and Biases logging during finetuning, create a Secret that contains your WandB account key. To do this, first [acquire your key from Weights and Biases](https://wandb.ai/authorize) and encode it using base64.

```bash
$ echo -n "example-wanbd-key" | base64

ZXhhbXBsZS13YW5iZC1rZXk=
```

Then, copy the encoded value into line `3` of `03-wandb-secret.yaml`.

When complete, the file should look like this:

```yaml
apiVersion: v1
data:
  token: ZXhhbXBsZS13YW5iZC1rZXk=
kind: Secret
metadata:
  name: wandb-token-secret
type: Opaque
```

Once the file is updated with your encoded account key, apply it to your namespace with `kubectl`:

```bash
kubectl apply -f 03-wandb-secret.yaml
```

## Run the Workflow

Once all of the necessary resources are created, submit the Workflow using the Argo CLI. If it is not already installed, follow [Argo's installation instructions](https://argoproj.github.io/argo-workflows/quick-start/#install-the-argo-workflows-cli) to install the CLI tool.

To submit the Workflow to the Argo server created earlier, use the `argo submit` command. The `-p` flag can be used to set the value for any of the parameters in the YAML file in line.

```bash
argo submit 04-finetune-workflow.yaml \
    -p run_name=finetune_gpt_neox \
    --serviceaccount finetune
```

Once the Workflow is submitted, its progress may be monitored from the Argo Workflows Web UI, which can be accessed via the URL provided in the application's deployment page. Retrieve this page by navigating to the **Applications** page on CoreWeave Cloud, then clicking on the Argo application.

Pod logs may be acquired via CLI using `kubectl logs <pod name>`, or by clicking on the relevant stage in the Argo Workflows Web UI.

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption><p>Argo Workflow right after submission</p></figcaption></figure>

The logs from the finetuning training script are available from the launcher Pod. They can be accessed via `kubectl`:

```bash
kubectl logs finetune-gpt-neox-n6mnd-mpijob-launcher-xz98s
```

{% hint style="info" %}
**Note**

Once complete, the finetuned model checkpoint will be saved in the `neox-checkpoints` PVC. The path is defined as a workflow parameter and defaults to `pvc://neox-checkpoints/20B_finetuned_checkpoint`.
{% endhint %}

### Debugging

The GPT-NeoX code can hang indefinitely if both the Worker Pods are not up and running when the main container in the launcher begins to run. To prevent this, an init container is used in the launcher, which sleeps for 60 seconds. In most cases, this will prevent a hanging issue. However, if the Worker Pods take longer than 60 seconds to spin up - which may occur if they are in a `pending` state due to a lack of resource availability - they might not spin up in time.

One quick way to check if the launcher is properly connected to both of the Worker Pods is by checking the Worker logs. If the most recent line in these logs is `Accepted publickey for root...`, then the launcher is connected. On the other hand, if the most recent log is `Disconnected...`, then the launcher isn't currently connected. This could be the sign of an issue.

In this example, the logs show the launcher properly connected to the worker:

<pre class="language-bash"><code class="lang-bash"><strong>$ kubectl logs finetune-gpt-neox-n6mnd-mpijob-worker-1
</strong>
Server listening on 0.0.0.0 port 22.
Server listening on :: port 22.

Accepted publickey for root from 10.145.231.160 port 38432 ssh2: ECDSA SHA256:mq7qxkWCmx7Srl2iavbJ0Dk7KsBriu1UvYnUCcruAts
</code></pre>

## Clean up

Once the Workflow has finished, the Pods used for all Workflow stages will move to a `Completed` state in order to keep the logs available for viewing. At this point, they are no longer using any compute resources, so will not incur any cost.

The easiest way to clean all of these Pods up is by deleting the Workflow run from the Argo Workflows Web UI. You may also delete them manually using `kubectl delete pod`.

Deleting the Argo Workflow alone won't remove all of the resources. The `mpijob` resource that was used for finetuning, and the `configmap` resource will still exist. To delete them, target them specifically using `kubectl delete`:

```bash
kubectl delete configmap neox-training
kubectl delete mpijob <mpijob-name-created-by-argo> 
```

{% hint style="info" %}
**Note**

The unique name of the `mpijob` can be acquired using `kubectl get mpijob`.
{% endhint %}

Finally, remove the resources that were created prior to running the Workflow. You can delete the Argo Workflows deployment through the CoreWeave's Applications UI. The PVCs, finetune Service Account role, and Weights and Biases secret can be deleted by targeting the files that created them with `kubectl delete`:

```bash
kubectl delete -f 03-wandb-secret.yaml
kubectl delete -f 02-finetune-role.yaml
kubectl delete -f 01-pvc.yaml
```
