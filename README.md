# TCP &asymp; RDMA: CPU-efficient Remote Storage Access with i10
i10 is a new in-kernel remote storage I/O stack for high-performance network and storage hardware. Our i10 design offers a number of benefits:
- First, i10 requires no modifications outside the kernel stack; thus, existing applications can operate directly on top of i10 without any modifications, whatsoever. 
- Second, i10 operates directly on top of the well-understood TCP/IP kernel protocol stack; thus, it require no modifications in the network stack and/or hardware. 
- Third, i10 complies with the [NVMe-over-Fabrics standard](https://nvmexpress.org/resources/specifications/); thus i10 can work with all emerging NVMe storage devices. 
- Fourth and perhaps most interesting benefit of i10 is that, over standard performance benchmarks, it is able to saturate a 100Gbps link using a commodity server with CPU utilization roughly similar to state-of-the-art [NVMe-over-RDMA products](https://kazan-networks.com/achieving-2-8m-iops-with-100gb-nvme-of/). i10 maintains this performance across a wide variety of evaluated workloads including varying read/write ratios, request sizes and device types; moreover, i10 scales well with number of cores and with number of remote storage devices.

## Requirements
i10 is currently implemented as a new fabric of the NVMe-over-Fabrics implementation, so it works like NVMe-over-i10 based on the kernel TCP/IP stack.

i10 has been successfully tested on Ubuntu 16.04 LTS with kernel 4.20.0 --- we use the kernel source (ver 4.20.0) that includes NVMe-over-TCP implementation ([download](http://git.infradead.org/nvme.git/snapshot/eb00c1a1852eb91e1b303aad0cb331318b7b9a0c.tar.gz)) (-> use this link: [download](http://www.cs.cornell.edu/~jaehyun/eb00c1a1852eb91e1b303aad0cb331318b7b9a0c.tar.gz)).

## Setup instructions (with root)
We assume the kernel source tree is downloaded in /usr/src/linux-4.20.0/. You should overwrite the i10 code to the kernel source tree and compile the kernel to use i10. We plan to provide i10 kernel patches for later versions of kernels (e.g., >= ver 5.0.0).

1. Download i10 kernel source code and copy to the kernel source tree:

   ```
   git clone https://github.com/i10-kernel/i10-implementation.git
   cd i10-implementation
   cp -rf drivers include /usr/src/linux-4.20.0/
   cd /usr/src/linux-4.20.0/
   ```

2. Update kernel configuration:

   ```
   cp /boot/config-x.xx.x .config (=> your current kernel version)
   make oldconfig
   ```

3. Make sure i10 modules are included in the kernel configuration:

   ```
   make menuconfig

   - To include i10 host: Device Drivers ---> NVME Support ---> <M> i10: A New Remote Storage I/O Stack (host)
   - To include i10 target: Device Drivers ---> NVME Support ---> <M> i10: A New Remote Storage I/O Stack (target)
   - To use the high resolution timer: General setup ---> Timers subsystem ---> [*] High Resolution Timer Support
   ```

4. Compile and install:

   ```
   make -j24 bzImage
   make -j24 modules
   make modules_install
   make install
   ```

5. Reboot with the new kernel.

6. Repeat above 1--5 steps in a target server.


## Running i10
We assume that the target server has target devices such as NVMe SSD (/dev/nvme0n1) or RAM block device (/dev/ram0). Please refer to "[HowTo Configure NVMe over Fabrics](https://community.mellanox.com/s/article/howto-configure-nvme-over-fabrics)" for more information about NVMe-over-Fabrics configurations.

### Configure i10 target server
1. Load i10 target kernel module:

   ```
   modprobe nvmet
   modprobe i10-target
   ```

2. Create an NVMe-over-i10 subsystem (e.g., nvme_i10):

   ```
   mkdir /sys/kernel/config/nvmet/subsystems/nvme_i10
   cd /sys/kernel/config/nvmet/subsystems/nvme_i10
   
   echo 1 > attr_allow_any_host
   mkdir namespaces/10
   cd namespaces/10
   echo -n /dev/nvme0n1 > device_path
   echo 1 > enable
   
   mkdir /sys/kernel/config/nvmet/ports/1
   cd /sys/kernel/config/nvmet/ports/1
   echo 192.168.xxx.xxx > addr_traddr (=> target IP address)
   echo i10 > addr_trtype
   echo 4420 > addr_trsvcid
   echo ipv4 > addr_adrfam
   
   ln -s /sys/kernel/config/nvmet/subsystems/nvme_i10 /sys/kernel/config/nvmet/ports/1/subsystems/nvme_i10
   ```

### Configure i10 host server
1. Install NVMe utility (nvme-cli):

   ```
   git clone https://github.com/linux-nvme/nvme-cli.git
   cd nvme-cli
   make
   make install
   ```

2. Load i10 host kernel module:

   ```
   modprobe i10-host
   ```

3. Connect to the target subsystem:

   ```
   nvme connect -t i10 -n nvme_i10 -a 192.168.xxx.xxx (=> target IP address) -s 4420 -q nvme_i10_host
   ```

4. Find the remote storage (e.g., /dev/nvme1n1)

   ```
   nvme list
   ```

5. Run test scripts: refer to '[scripts/](https://github.com/i10-kernel/i10-implementation/tree/master/scripts)'.

### Configure the delayed doorbell timeout value (in microsecond)
i10 uses 50us delayed doorbell timer by default. If you want to set the timeout value to, for example, 100us:

```
echo 100 > /sys/module/i10_host/parameters/i10_delayed_doorbell_us
```

Please refer to the [paper](https://www.usenix.org/conference/nsdi20/presentation/hwang) to get an insight how i10 performs with different timeout values.
