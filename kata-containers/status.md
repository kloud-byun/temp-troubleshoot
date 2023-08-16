# Kata Containers Installation and Configuration on RHEL8.6 (Kata stable-v3.0)

### Set Up
```bash
# install kata-containers, containerd, fc binary, ctr, etcs
bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/kata-containers/main/utils/kata-manager.sh) -k 3.0.0 -t"

# instal nerdctl
wget https://github.com/containerd/nerdctl/releases/download/v1.5.0/nerdctl-1.5.0-linux-amd64.tar.gz
sudo tar Cxzvvf /usr/local/bin nerdctl-1.5.0-linux-amd64.tar.gz

# install dependencies for nerdctl
##################################
# rootlesskit
curl -sSL https://github.com/rootless-containers/rootlesskit/releases/download/v1.1.1/rootlesskit-$(uname -m).tar.gz | sudo tar Cxzv /bin
# slirp4netns
curl -o slirp4netns --fail -L https://github.com/rootless-containers/slirp4netns/releases/download/v1.2.0/slirp4netns-$(uname -m)
chmod +x slirp4netns
sudo mv slirp4netns /usr/local/bin/
# buildkit
wget https://github.com/moby/buildkit/releases/download/v0.12.1/buildkit-v0.12.1.linux-amd64.tar.gz
sudo tar -xzvf buildkit-v0.12.1.linux-amd64.tar.gz -C /usr/local/bin/ --strip-components=1
# CNI plugin
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.3.0.tgz
# runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64
chmod +x runc.amd64
sudo mv runc.amd64 /usr/local/bin/runc

# setup nerdctl rootless mode
containerd-rootless-setuptool.sh install
containerd-rootless-setuptool.sh install-buildkit
```

### Firecracker Setup
```bash
# install firecracker w/ jailer
release_url="https://github.com/firecracker-microvm/firecracker/releases"
version="v1.4.0"
arch=`uname -m`
curl -L ${release_url}/download/${version}/firecracker-${version}-${arch}.tgz -o firecracker.tgz
mkdir firecracker-${version} && tar -xzvf firecracker.tgz -C firecracker-${version} --strip-components=1

# create symbolic links
sudo ln -s $(pwd)/firecracker-${version}/firecracker-${version}-${arch} /usr/local/bin/firecracker
sudo ln -s $(pwd)/firecracker-${version}/jailer-${version}-${arch} /usr/local/bin/jailer

## configure firecracker
# create devmapper script
sudo mkdir -p /scripts/devmapper
echo '#!/bin/bash
set -ex

DATA_DIR=/var/lib/containerd/devmapper
POOL_NAME=devpool

mkdir -p ${DATA_DIR}

# Create data file
sudo touch "${DATA_DIR}/data"
sudo truncate -s 100G "${DATA_DIR}/data"

# Create metadata file
sudo touch "${DATA_DIR}/meta"
sudo truncate -s 10G "${DATA_DIR}/meta"

# Allocate loop devices
DATA_DEV=$(sudo losetup --find --show "${DATA_DIR}/data")
META_DEV=$(sudo losetup --find --show "${DATA_DIR}/meta")

# Define thin-pool parameters.
# See https://www.kernel.org/doc/Documentation/device-mapper/thin-provisioning.txt for details.
SECTOR_SIZE=512
DATA_SIZE="$(sudo blockdev --getsize64 -q ${DATA_DEV})"
LENGTH_IN_SECTORS=$(bc <<< "${DATA_SIZE}/${SECTOR_SIZE}")
DATA_BLOCK_SIZE=128
LOW_WATER_MARK=32768

# Create a thin-pool device
sudo dmsetup create "${POOL_NAME}" \
    --table "0 ${LENGTH_IN_SECTORS} thin-pool ${META_DEV} ${DATA_DEV} ${DATA_BLOCK_SIZE} ${LOW_WATER_MARK}"

cat << EOF
#
# Add this to your config.toml configuration file and restart containerd daemon
#
[plugins]
  [plugins.devmapper]
    pool_name = "${POOL_NAME}"
    root_path = "${DATA_DIR}"
    base_image_size = "10GB"
    discard_blocks = true
EOF' | sudo tee /scripts/devmapper/create.sh
sudo chmod +x /scripts/devmapper/create.sh
sudo /scripts/devmapper/create.sh

# add the generated devmapper config to containerd config
echo '
  [plugins.devmapper]
    pool_name = "devpool"
    root_path = "/var/lib/containerd/devmapper"
    base_image_size = "10GB"
    discard_blocks = true' | sudo tee -a /etc/containerd/config.toml
sudo systemctl restart containerd
sudo ctr plugins ls | grep devmapper # should now show 'ok'

# add a reload script to persist data
echo '#!/bin/bash
set -ex

DATA_DIR=/var/lib/containerd/devmapper
POOL_NAME=devpool

# Allocate loop devices
DATA_DEV=$(sudo losetup --find --show "${DATA_DIR}/data")
META_DEV=$(sudo losetup --find --show "${DATA_DIR}/meta")

# Define thin-pool parameters.
# See https://www.kernel.org/doc/Documentation/device-mapper/thin-provisioning.txt for details.
SECTOR_SIZE=512
DATA_SIZE="$(sudo blockdev --getsize64 -q ${DATA_DEV})"
LENGTH_IN_SECTORS=$(bc <<< "${DATA_SIZE}/${SECTOR_SIZE}")
DATA_BLOCK_SIZE=128
LOW_WATER_MARK=32768

# Create a thin-pool device
sudo dmsetup create "${POOL_NAME}" \
    --table "0 ${LENGTH_IN_SECTORS} thin-pool ${META_DEV} ${DATA_DEV} ${DATA_BLOCK_SIZE} ${LOW_WATER_MARK}"' | sudo tee /scripts/devmapper/reload.sh
sudo chmod +x /scripts/devmapper/reload.sh

# create a systemd service for reloading
echo '
[Unit]
Description=Devmapper reload script

[Service]
ExecStart=/scripts/devmapper/reload.sh

[Install]
WantedBy=multi-user.target' | sudo tee /lib/systemd/system/devmapper_reload.service

# enable it
sudo systemctl daemon-reload
sudo systemctl enable devmapper_reload.service
sudo systemctl start devmapper_reload.service

# configure kata-containers with firecracker
sudo mkdir /etc/kata-containers
sudo cp /opt/kata/share/defaults/kata-containers/configuration-fc.toml /etc/kata-containers/configuration.toml

# configure containerd
echo '#!/bin/bash
KATA_CONF_FILE=/etc/kata-containers/configuration.toml /opt/kata/bin/containerd-shim-kata-v2 $@' | sudo tee /usr/local/bin/containerd-shim-kata-fc-v2
sudo chmod +x /usr/local/bin/containerd-shim-kata-fc-v2
sudo sed -i '/\[plugins.devmapper\]/i\        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata-fc]\n          runtime_type = "io.containerd.kata-fc.v2"\n' /etc/containerd/config.toml
sudo systemctl restart containerd.service
```

### Run
```bash
# pull image
sudo ctr images pull --snapshotter devmapper docker.io/library/ubuntu:latest

### run with qemu (default)
# with ctr
sudo ctr run --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-test uname -a

# with nerdctl
nerdctl run --runtime io.containerd.run.kata.v2 -it --rm ubuntu:latest uname -a

### run with firecracker
# with ctr
sudo ctr run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-test uname -a

# with nerdctl
nerdctl run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -it --rm ubuntu:latest uname -a
```

### Results:
```
[vagrant@yellow ~]$ sudo ctr run --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-test uname -a
ctr: failed to create shim task: failed to mount "/run/kata-containers/shared/containers/ubuntu-kata-test/rootfs" to "/run/kata-containers/ubuntu-kata-test/rootfs", with error: ENOENT: No such file or directory: unknown

[vagrant@yellow ~]$ nerdctl run --runtime io.containerd.run.kata.v2 -it --rm ubuntu:latest uname -a
docker.io/library/ubuntu:latest:                                                  resolved       |++++++++++++++++++++++++++++++++++++++| 
index-sha256:ec050c32e4a6085b423d36ecd025c0d3ff00c38ab93a3d71a460ff1c44fa6d77:    done           |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:56887c5194fddd8db7e36ced1c16b3569d89f74c801dc8a5adbf48236fb34564: done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:01f29b872827fa6f9aed0ea0b2ede53aea4ad9d66c7920e81a8db6d1fd9ab7f9:   done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:b237fe92c4173e4dfb3ba82e76e5fed4b16186a6161e07af15814cb40eb9069d:    done           |++++++++++++++++++++++++++++++++++++++| 
elapsed: 3.1 s                                                                    total:  28.2 M (9.1 MiB/s)                                       
FATA[0003] failed to create shim task: Could not create the sandbox resource controller mkdir /sys/fs/cgroup/systemd/vc: permission denied: unknown 

[vagrant@yellow ~]$ sudo ctr run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -t --rm docker.io/library/ubuntu:latest ubuntu-kata-test uname -a
Linux clr-d4d8ba666a874e27b2c86511dda6203b 5.19.2 #2 SMP Sun Oct 9 09:11:55 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

[vagrant@yellow ~]$ nerdctl run --snapshotter devmapper --runtime io.containerd.run.kata-fc.v2 -it --rm ubuntu:latest uname -a
FATA[0000] error unpacking image: failed to stat snapshot sha256:bce45ce613d34bff6a3404a4c2d56a5f72640f804c3d0bd67e2cf0bf97cb950c: snapshotter not loaded: devmapper: invalid argument
```

### current environemnt
```
[vagrant@yellow ~]$ /opt/kata/bin/containerd-shim-kata-v2 --version
Kata Containers containerd shim: id: "io.containerd.kata.v2", version: 3.0.0, commit: e2a8815ba46360acb8bf89a2894b0d437dc8548a

[vagrant@yellow ~]$ kata-runtime --version
kata-runtime  : 3.0.0
   commit   : e2a8815ba46360acb8bf89a2894b0d437dc8548a
   OCI specs: 1.0.2-dev

[vagrant@yellow ~]$ containerd --version
containerd github.com/containerd/containerd v1.7.3 7880925980b188f4c97b462f709d0db8e8962aff

[vagrant@yellow ~]$ nerdctl --version
nerdctl version 1.5.0

[vagrant@yellow ~]$ cat /etc/containerd/config.toml
# 2023-08-10T00:16:34+00:00: Added by bash
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "kata"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
          runtime_type = "io.containerd.kata.v2"

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata-fc]
          runtime_type = "io.containerd.kata-fc.v2"

  [plugins.devmapper]
    pool_name = "devpool"
    root_path = "/var/lib/containerd/devmapper"
    base_image_size = "10GB"
    discard_blocks = true

[vagrant@yellow ~]$ sudo ctr plugins ls | grep devmapper
io.containerd.snapshotter.v1           devmapper                linux/amd64    ok        

[vagrant@yellow ~]$ sudo dmsetup info devpool
Name:              devpool
State:             ACTIVE
Read Ahead:        256
Tables present:    LIVE
Open count:        0
Event number:      0
Major, minor:      253, 3
Number of targets: 1

[vagrant@yellow ~]$ cat /etc/kata-containers/configuration.toml
[hypervisor.firecracker]
path = "/opt/kata/bin/firecracker"
kernel = "/opt/kata/share/kata-containers/vmlinux.container"
image = "/opt/kata/share/kata-containers/kata-containers.img"
enable_annotations = ["enable_iommu"]
valid_hypervisor_paths = ["/opt/kata/bin/firecracker"]
jailer_path = "/opt/kata/bin/jailer"
valid_jailer_paths = ["/opt/kata/bin/jailer"]
kernel_params = ""
default_vcpus = 1
default_maxvcpus = 0
default_bridges = 1
default_memory = 2048
default_maxmemory = 0
block_device_driver = "virtio-mmio"
valid_entropy_sources = ["/dev/urandom","/dev/random",""]
disable_selinux=false

[factory]

[agent.kata]
kernel_modules=[]

[runtime]
internetworking_model="tcfilter"
disable_guest_seccomp=true
sandbox_cgroup_only=false
static_sandbox_resource_mgmt=true
disable_guest_empty_dir=false
experimental=[]
```

### Verify
* After running the container, check processes within another terminal:
    ```bash
    ps aux | grep fire
    ps aux | grep kata
    ```

## Troubleshoot / Debug
* To see options for installing kata-containers without installing anything:
    ```bash
    bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/kata-containers/main/utils/kata-manager.sh) -h"
    ```
