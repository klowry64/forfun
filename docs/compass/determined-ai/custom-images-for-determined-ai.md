---
description: Build your own images for running machine learning models on Determined AI
---

# Custom Images for Determined AI

## Prerequisites

### Install Determined AI

Before continuing with the rest of this guide, the Determined AI application must be installed in your namespace.

<table data-card-size="large" data-view="cards"><thead><tr><th></th><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td>--> <strong>Install Determined AI in your namespace</strong></td><td></td><td></td><td><a href="./#get-started">#get-started</a></td></tr></tbody></table>

### Standard images

Determined AI provides a few useful [standard default base images](https://docs.determined.ai/latest/training/setup-guide/custom-env.html#default-images). It is strongly recommended to use one of these official Determined images as a base image, using the `FROM` instruction in your image's Dockerfile. For example:

```docker
# Determined Image
FROM determinedai/environments:cuda-11.3-pytorch-1.10-tf-2.8-gpu-0.19.10
```

#### Determined AI default base images

| Environment | Image                                                                | Framework      |
| ----------- | -------------------------------------------------------------------- | -------------- |
| CPUs        | `determinedai/environments:py-3.8-pytorch-1.10-tf-2.8-cpu-0.19.4`    | PyTorch 1.10   |
| Nvidia GPUs | `determinedai/environments:cuda-11.3-pytorch-1.10-tf-2.8-gpu-0.19.4` | Cuda 11.3      |
| AMD GPUs    | `determinedai/environments:rocm-5.0-pytorch-1.10-tf-2.7-rocm-0.19.4` | TensorFlow 2.8 |

{% hint style="danger" %}
**Warning**

Determined AI warns AGAINST installing TensorFlow, PyTorch, Horovod, or Apex packages, which conflict with Determined-installed packages.

This can cause issues for people looking to build custom models that pin different versions of PyTorch, TensorFlow, Horovod, and so forth. CoreWeave provides a repository and a guide from our efforts to build custom images on the Determined AI platform.

At this time, the Determined AI team is working on building images for `CUDA==11.8` with the latest PyTorch that supports it using official NVIDIA base images.
{% endhint %}

{% hint style="info" %}
**Note**

The below guidelines and repository provide insight into how to go about this process. The build process will vary based on requirements; trial and error may be required to get your custom image to work on the Determined AI platform.
{% endhint %}

### Python dependencies

The Determined AI Python package pins certain version of packages required to run their setup harness and to execute their launcher. All dependencies are listed in [Determined's provided `setup.py` file](https://github.com/determined-ai/determined/blob/master/harness/setup.py).

{% hint style="warning" %}
**Important**

One of the dependencies Determined AI installs is `protobuf`. This  package may not be compatible with some custom images, which can cause issues during runtime. This can be mitigated by pinning the package version to `protobuf==3.19.4`.
{% endhint %}

### DeepSpeed

Determined AI uses a fork of the standard [DeepSpeed library](https://www.deepspeed.ai/) to handle node failures, node communication, and to host file management automatically. If you are using a Machine Learning model that requires DeepSpeed, use [Determined's fork](https://github.com/determined-ai/environments/blob/cd272fdb92824814aff76066b79edd38680b995e/Makefile#L220) of the library to ensure proper functionality on the Determined platform.

## Examples

{% hint style="info" %}
**Note**

The example Dockerfiles provided here are compatible with `CUDA==11.7`.
{% endhint %}

### PyTorch 1.13

<details>

<summary>Click to expand - PyTorch 1.13 Dockerfile</summary>

```docker
FROM ghcr.io/coreweave/nccl-tests:11.7.1-devel-ubuntu20.04-nccl2.14.3-1-45d6ec9

ENV DET_PYTHON_EXECUTABLE="/usr/bin/python3.8"
ENV DET_SKIP_PIP_INSTALL="SKIP"

# Run updates and install packages for build
RUN echo "Dpkg::Options { "--force-confdef"; "--force-confnew"; };" > /etc/apt/apt.conf.d/local
RUN apt-get -qq update && \
    apt-get -qq install -y --no-install-recommends software-properties-common && \
    add-apt-repository ppa:deadsnakes/ppa -y && \
    add-apt-repository universe && \
    apt-get -qq update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y curl tzdata build-essential daemontools && \
    apt-get install -y --no-install-recommends \
       python3.8 \
       python3.8-distutils \
       python3.8-dev \
       python3.8-venv \
       git && \
    apt-get clean

# python3.8 -m ensurepip --default-pip && \
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python3.8 get-pip.py
RUN python3.8 -m pip install --no-cache-dir --upgrade pip

RUN python3.8 -m pip install torch torchvision torchaudio
RUN python3.8 -m pip install --no-cache-dir install packaging

#### Python packages
RUN python3.8 -m pip install --no-cache-dir determined==0.19.9
RUN python3.8 -m pip install --no-cache-dir pybind11
RUN python3.8 -m pip install --no-cache-dir protobuf==3.19.4
RUN update-alternatives --install /usr/bin/python3 python /usr/bin/python3.8 2
RUN echo 2 | update-alternatives --config python
```

</details>

## Additional resources

* [Determined AI: Custom Images](https://docs.determined.ai/latest/training/setup-guide/custom-env.html#custom-images)
* [Determined AI: Image Build Repository](https://github.com/determined-ai/environments)
