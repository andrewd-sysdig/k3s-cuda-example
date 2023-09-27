# k3s build with cuda support
Single node 

# Provision VM
For this example I'm using an Azure VM Size: Standard NC6s v3 (6 vcpus, 112 GiB memory) which has a Tesla-V100-PCIE-16GB with RHEL8

```
cat /etc/os-release
NAME="Red Hat Enterprise Linux"
VERSION="8.6 (Ootpa)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="8.6"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Red Hat Enterprise Linux 8.6 (Ootpa)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:8::baseos"
HOME_URL="https://www.redhat.com/"
DOCUMENTATION_URL="https://access.redhat.com/documentation/red_hat_enterprise_linux/8/"
BUG_REPORT_URL="https://bugzilla.redhat.com/"

REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 8"
REDHAT_BUGZILLA_PRODUCT_VERSION=8.6
REDHAT_SUPPORT_PRODUCT="Red Hat Enterprise Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8.6"
```

# Confirm NVIDIA card detected
```
lspci |grep -e VGA -ie NVIDIA
0001:00:00.0 3D controller: NVIDIA Corporation GV100GL [Tesla V100 PCIe 16GB] (rev a1)
```

# Update OS & Kernel to latest
```
sudo dnf -y update
```
## Reboot for new kernel to take effect
```
sudo reboot
```
# Install Kernel header/tools (needed for nvidia to build kernel module)
## https://learn.microsoft.com/en-us/azure/virtual-machines/linux/n-series-driver-setup#centos-or-red-hat-enterprise-linux
```
$ sudo dnf -y install kernel kernel-tools kernel-headers kernel-devel
```

# Install nvidia driver 
## Add extra packages for enterprise linux - https://access.redhat.com/solutions/3358
```
sudo rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

## Install Dynamic Kernel Module Support for Nvidia drivers
```
sudo dnf -y install dkms
```

## Finally install the cuda-drivers - would be nice to do headless but haven't figured out how in RHEL
```
sudo dnf -y install cuda-drivers
```

## Confirm drivers are working ok
```
nvidia-smi --query-gpu=gpu_name --format=csv,noheader --id=0 | sed -e 's/ /-/g'
Tesla-V100-PCIE-16GB
```

# Install and setup Kubernetes
## Install k3s
```
curl -ksL get.k3s.io | sh -
```

## Confirm nvidia has been added as a conatiner runtime class to containerd
```
sudo grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."nvidia"]
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes."nvidia".options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
```

## Install and setup kubectl, autocomplete and k alias for user
```
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown 1000:1000 ~/.kube/config
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc 
echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

## Install helm
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Apply new nvidia RuntimeClass 
```
k apply -f nvidia-runtimeclass.yaml
```

## Create nvidia device plugin
***Note: this is different to the one on the nvidia github as it adds the runtimeclass into it which is needed to make it use the nvidia conatiner runtime***
```
kubectl apply -f nvidia-device-plugin.yaml
```

## Test with nvidia smi command
```
kubectl apply -f nvidia-smi-test.yaml
kubectl logs nvidia-test 
Wed Sep 27 06:42:14 2023       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.104.12             Driver Version: 535.104.12   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla V100-PCIE-16GB           Off | 00000001:00:00.0 Off |                    0 |
| N/A   26C    P0              24W / 250W |      0MiB / 16384MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+
kubectl delete pod nvidia-test
```