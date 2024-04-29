# Putting `ilab` in a Container AND making it go fast

Containerization of `ilab` allows for portability and ease of setup. With this,
users can now run lab on OpenShift to test the speed of `ilab train` and `generate`
using dedicated GPUs. This guide shows you how to put the `ilab` CLI, all of its
dependencies, and your GPU into a container for an isolated and easily reproducible
experience.

## Steps to build an image then run a container

**Containerfile:**

```dockerfile
FROM nvcr.io/nvidia/cuda:12.3.2-devel-ubi9
RUN dnf install -y python3.11 git python3-pip make automake gcc gcc-c++
WORKDIR /instructlab
RUN python3.11 -m ensurepip
RUN dnf install -y gcc
RUN rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
RUN dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo && dnf repolist && dnf config-manager --set-enabled cuda-rhel9-x86_64 && dnf config-manager --set-enabled cuda && dnf config-manager --set-enabled epel && dnf update -y
RUN python3.11 -m pip install --force-reinstall nvidia-cuda-nvcc-cu12
RUN export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/usr/local/cuda/extras/CUPTI/lib64" \
    && export CUDA_HOME=/usr/local/cuda \
    && export PATH="/usr/local/cuda/bin:$PATH" \
    && export XLA_TARGET=cuda120 \
    && export XLA_FLAGS=--xla_gpu_cuda_data_dir=/usr/local/cuda
RUN CMAKE_ARGS="-DLLAMA_CUBLAS=on" python3.11 -m pip install --force-reinstall --no-cache-dir llama-cpp-python
RUN python3.11 -m pip install git+https://github.com/instructlab/instructlab.git@stable
CMD ["/bin/bash"]
```

Or image: TBD (am I allowed to have a public image with references to lab in it?)

This `Containerfile` is based on Nvidia CUDA image, which lucky for us plugs
directly into Podman via their `nvidia-container-toolkit`! The `ubi9` base
image does not have most packages installed. The bulk of the `Containerfile` is
spent configuring your system so `ilab` can be installed and run properly.
`ubi9` as compared to `ubuntu` cannot install the entire `nvidia-12-4` toolkit.
This did not impact performance during testing.

```shell
1. podman build -f <Containerfile_Path>
2. curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo |   sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
3. sudo yum-config-manager --enable nvidia-container-toolkit-experimental
4. sudo dnf install -y nvidia-container-toolkit
5. sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
6. nvidia-ctk cdi list
    Example output: 
    INFO[0000] Found 2 CDI devices
    nvidia.com/gpu=0
    nvidia.com/gpu=all
7. podman run --device nvidia.com/gpu=0  --security-opt=label=disable -it <IMAGE_ID>
```

Voila! You now have a container with CUDA and GPUs enabled!

### Sources

[Nvidia Container Toolkit Install Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

[Podman Support for Container Device Interface](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html)
