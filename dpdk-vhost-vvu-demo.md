# DPDK Vhost with virtio-vhost-user support

## Step-by-step Guide

This guide explains how to use the virtio-vhost-user transport when
running the vhost-scsi example application and the vhost PMD with the
testpmd app. This guide relies on static building though shared library
building works too.

DPDK: [https://github.com/ndragazis/dpdk-next-virtio/tree/virtio-vhost-user](https://github.com/ndragazis/dpdk-next-virtio/tree/virtio-vhost-user)\
QEMU: [https://github.com/ndragazis/qemu/tree/virtio-vhost-user](https://github.com/ndragazis/qemu/tree/virtio-vhost-user)

### 1. Build QEMU and DPDK
```
$ git clone -b virtio-vhost-user git@github.com:ndragazis/qemu.git
$ cd qemu
$ ./configure --target-list=x86_64-softmmu
$ make
```
```
$ git clone -b virtio-vhost-user git@github.com:ndragazis/dpdk-next-virtio.git
$ cd dpdk-next-virtio
$ make install T=x86_64-native-linuxapp-gcc
```

### 2. Vhost_scsi Example Application

#### 2.1 Build the application
```
$ make "RTE_SDK=$PWD" RTE_TARGET=x86_64-native-linuxapp-gcc -C examples/vhost_scsi
```

#### 2.2 Launch the Storage Appliance VM (Slave VM):
```
$ ./qemu/x86_64-softmmu/qemu-system-x86_64 \
  -machine q35,accel=kvm -cpu host -smp 2 -m 4G \
  -drive if=none,file=image.qcow2,format=qcow2,id=bootdisk \
  -device virtio-blk-pci,drive=bootdisk,id=virtio-disk1,bootindex=0,addr=04.0 \
  -chardev socket,id=chardev0,path=vhost-user.sock,server,nowait \
  -device virtio-vhost-user-pci,chardev=chardev0,addr=07.0 \
  -net nic -net user,hostfwd=tcp:127.0.0.1:22223-:22 \
  -display none \
  -monitor stdio
```

#### 2.3 Connect to the Storage Appliance VM with ssh:
```
$ ssh -p 22223 <username>@localhost
```

#### 2.4 Run the vhost_scsi example application:
```
slave:~$ su
# export VVU_DEVICE="0000:00:07.0"
# modprobe vfio enable_unsafe_noiommu_mode=1
# modprobe vfio-pci
# mkdir -p dpdk/
# sshfs -o allow_other <username>@10.0.2.2:<path/to/dpdk/repo/on/host/fs> ./dpdk/
# cd dpdk/
# ./usertools/dpdk-devbind.py -b vfio-pci '00:07.0'
# echo 128 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
slave:~$ sudo ./examples/vhost_scsi/build/vhost-scsi -l 0-1 --pci-whitelist "$VVU_DEVICE" --no-huge -- --virtio-vhost-user-pci "$VVU_DEVICE"
```

#### 2.5 Launch the Compute VM (Master VM):
```
$ ./qemu/x86_64-softmmu/qemu-system-x86_64 \
  -M accel=kvm -cpu host -m 1G \
  -object memory-backend-file,id=mem0,mem-path=/dev/shm/ivshmem,size=1G,share=on \
  -numa node,memdev=mem0 \
  -drive if=virtio,file=image.qcow2,format=qcow2 \
  -chardev socket,id=chardev0,path=vhost-user.sock \
  -net nic -net user,hostfwd=tcp:127.0.0.1:22224-:22
  -display none \
  -monitor stdio
```

#### 2.6 Add a vhost-user-scsi-pci device to the Compute VM through the QEMU Monitor:
```
(qemu) device_add vhost-user-scsi-pci,disable-modern=on,chardev=chardev0
```

#### 2.7 Connect to the Compute VM with ssh:
```
$ ssh -p 22224 <username>@localhost
```

#### 2.8 Check that the Malloc RAM-disk is visible to the Compute VM:
```
master:~$ lsscsi
[1:0:0:0]    disk    INTEL    vhost_scsi_mall  000   /dev/sda
```

### 3. Vhost PMD with Testpmd App

#### 3.1 Launch the Network Appliance VM (Slave VM):
```
$ ./qemu/x86_64-softmmu/qemu-system-x86_64 \
  -machine q35,accel=kvm -cpu host -smp 2 -m 4G \
  -drive if=none,file=image.qcow2,format=qcow2,id=bootdisk \
  -device virtio-blk-pci,drive=bootdisk,id=virtio-disk1,bootindex=0,addr=04.0 \
  -netdev user,id=netdev0,hostfwd=tcp:127.0.0.1:22223-:22 \
  -device e1000,netdev=netdev0,addr=05.0 \
  -netdev user,id=netdev1 \
  -device virtio-net-pci,netdev=netdev1,addr=06.0 \
  -chardev socket,id=chardev0,path=vhost-user.sock,server,nowait \
  -device virtio-vhost-user-pci,chardev=chardev0,addr=07.0 \
  -display none \
  -monitor stdio
```

#### 3.2 Connect to the Network Appliance VM with ssh:
```
$ ssh -p 22223 <username>@localhost
```

#### 3.3 Run vhost PMD with testpmd app (I/O mode):
```
slave:~$ su
# export VVU_DEVICE="0000:00:07.0"
# export VNET_DEVICE="0000:00:06.0"
# modprobe vfio enable_unsafe_noiommu_mode=1
# modprobe vfio-pci
# mkdir -p dpdk/
# sshfs -o allow_other <username>@10.0.2.2:<path/to/dpdk/repo/on/host/fs> ./dpdk/
# cd dpdk/
# ./usertools/dpdk-devbind.py -b vfio-pci --force '00:06.0'
# ./usertools/dpdk-devbind.py -b vfio-pci '00:07.0'
# echo 256 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
# cd x86_64-native-linuxapp-gcc/app/
# ./testpmd -l 0-1 --pci-whitelist "$VVU_DEVICE" \
            --vdev net_vhost0,iface="$VVU_DEVICE",virtio-transport=1 \
            --pci-whitelist "$VNET_DEVICE" -- -i
```

#### 3.4 Launch the Compute VM (Master VM):
```
$ ./qemu/x86_64-softmmu/qemu-system-x86_64 \
  -M accel=kvm -cpu host -m 1G \
  -object memory-backend-file,id=mem0,mem-path=/dev/shm/ivshmem,size=1G,share=on \
  -numa node,memdev=mem0 \
  -drive if=virtio,file=image.qcow2,format=qcow2 \
  -chardev socket,id=chardev0,path=vhost-user.sock \
  -netdev vhost-user,chardev=chardev0,id=netdev0 \
  -device virtio-net-pci,netdev=netdev0 \
```

#### 3.5 Start forwarding packets:
```
testpmd> show config fwd
testpmd> start
testpmd> set verbose 3
```

#### 3.6 Validate that the packets are being forwarded:
```
testpmd> show fwd stats all
```
