# A Complete Guide to Building an Intel TCC/Real-Time Environment on RHEL and Implementing EtherCAT Communication in a Podman Container

This document provides a complete procedure for setting up RHEL for Real Time on a machine with an Intel TCC-supported CPU, installing and configuring the IgH EtherCAT Master, and finally, controlling EtherCAT communications from within an application running in a Podman container.

-----

### Part 1: Host OS Real-Time Environment Setup

**Objective**: To tune the hardware and OS to reserve specific CPU cores as a "sanctuary" for real-time tasks.

#### **1.1. BIOS/UEFI Configuration (Hardware-Level Preparation)**

**Objective**: To switch the CPU itself into a "full performance mode" optimized for real-time processing before the OS boots.

A typical PC's CPU is designed to balance performance and power consumption. Features like power-saving modes and dynamic performance scaling can cause "unpredictable latency" (jitter) for real-time processing. Enabling "Intel® TCC Mode" eliminates or optimizes these sources of variability at the hardware level.

**Steps**:

1.  Immediately after powering on the PC, press the specific key for your manufacturer (often `F2`, `Del`, `F10`, or `Esc`) to enter the setup utility.
2.  Navigate to a menu such as `Advanced` -\> `CPU Configuration` or `Performance`.
3.  Change the "**Intel® TCC Mode**" or a similarly named option to **Enabled**, then save the configuration and reboot.

#### **1.2. OS Preparation (OS-Level Preparation)**

**Objective**: To install a special OS kernel designed to handle real-time tasks with the highest priority.

While standard OS kernels prioritize "fairness" among all tasks, a real-time kernel (`kernel-rt`) operates on the philosophy that "priority is everything."

**Steps**:

1.  **Enable the Repository**: Enable the repository containing the real-time packages.
    ```bash
    sudo subscription-manager repos --enable rhel-9-for-x86_64-rt-rpms
    ```
2.  **Install the Package Group**: Install the real-time kernel and related tools.
    ```bash
    sudo dnf groupinstall "RT"
    ```
3.  **Reboot and Verify**: Reboot the system and confirm that the output of the `uname -r` command includes the `-rt` suffix.

#### **1.3. Kernel Tuning (CPU Core Isolation)**

Use the `grubby` command to isolate specific CPU cores from the OS's general-purpose scheduler.

**Example (Isolating cores 2 & 3)**:

```bash
sudo grubby --update-kernel=ALL --args="isolcpus=2-3 nohz_full=2-3 rcu_nocbs=2-3"
```

After running the command, reboot the system and verify the setting has been applied by running `cat /proc/cmdline`.

#### **1.4. Applying the `tuned` Profile**

Optimize the entire OS for real-time operations.

```bash
sudo tuned-adm profile realtime
```

Verify the profile is active by running `tuned-adm active`.

-----

### Part 2: IgH EtherCAT Master Installation & Configuration (Host OS)

**Objective**: To install and correctly configure the EtherCAT master software on the host OS.

#### **2.1. Install Build Dependencies**

Install the development tools required to build the source code.

```bash
sudo dnf install kernel-rt-devel-$(uname -r) gcc make autoconf automake libtool wget
```

#### **2.2. Download and Extract Source Code**

Download the EtherCAT master source code.

```bash
cd /usr/src/
wget https://download.etherlab.org/ethercat/ethercat-1.6.7.tar.bz2
tar -xvjf ethercat-1.6.7.tar.bz2
cd ethercat-1.6.7/
```

#### **2.3. Patch Source Code for Compatibility (Critical)**

To build this version of the EtherCAT master on a modern RHEL 9 kernel, two lines of source code must be modified.

1.  **Patch 1 (`cdev.c`)**:

    ```bash
    sudo nano +233 /usr/src/ethercat-1.6.7/master/cdev.c
    ```

    Comment out line 233 by adding `//` to the beginning.
    **Before**: `vma->vm_flags |= VM_DONTDUMP;`
    **After**: `// vma->vm_flags |= VM_DONTDUMP;`

2.  **Patch 2 (`module.c`)**:

    ```bash
    sudo nano +115 /usr/src/ethercat-1.6.7/master/module.c
    ```

    Modify line 115 to remove the extra argument.
    **Before**: `class = class_create(THIS_MODULE, "EtherCAT");`
    **After**: `class = class_create("EtherCAT");`

#### **2.4. Build and Install**

After patching the source code, run the build and install commands.

```bash
./configure --with-linux-dir=/lib/modules/$(uname -r)/build --disable-8139too --enable-generic
make
sudo make modules_install
sudo make install
sudo ldconfig
```

#### **2.5. Configure the EtherCAT Master**

Set the MAC address of the dedicated NIC for EtherCAT communication.

1.  **Find the MAC Address** (for `enp3s0`):
    ```bash
    ip addr show enp3s0
    ```
2.  **Edit the Configuration File**:
    ```bash
    sudo nano /usr/local/etc/ethercat.conf
    ```
3.  Find and edit the following two lines, using the MAC address for your hardware.
    ```ini
    # Before
    MASTER0_DEVICE=""
    DEVICE_MODULES=""

    # After (use your specific MAC address)
    MASTER0_DEVICE="cc:82:7f:92:d2:37"
    DEVICE_MODULES="generic"
    ```

#### **2.6. Dedicate the NIC (Disable NetworkManager)**

Prevent NetworkManager from managing the dedicated NIC so EtherCAT can have exclusive control.

1.  Create a configuration file.
    ```bash
    sudo nano /etc/NetworkManager/conf.d/99-unmanaged-devices.conf
    ```
2.  Add the following content (using your specific MAC address).
    ```ini
    [keyfile]
    unmanaged-devices=mac:cc:82:7f:92:d2:37
    ```
3.  Restart NetworkManager.
    ```bash
    sudo systemctl restart NetworkManager
    ```

#### **2.7. Enable the `systemd` Service**

Register and start the EtherCAT service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ethercat
```

Verify it is `active (exited)` by running `systemctl status ethercat.service`.

-----

### Part 3: Podman Container Application Preparation

**Objective**: To build a self-contained container image for the EtherCAT application.

#### **3.1. Prepare Build Files**

Create a working directory (e.g., `/root/test`) and place the following four files inside it.

1.  **`ethercat-1.6.7.tar.bz2`**
    Copy the source archive from `/usr/src/` into your current directory.

    ```bash
    # cd /root/test/
    cp /usr/src/ethercat-1.6.7.tar.bz2 .
    ```

2.  **`simple_test.c`**

    ```c
    #include <stdio.h>
    #include "ecrt.h"

    int main(int argc, char **argv) {
        ec_master_t *master;
        ec_master_info_t master_info;

        master = ecrt_request_master(0);
        if (!master) {
            fprintf(stderr, "Failed to request master 0!\n");
            return -1;
        }
        if (ecrt_master(master, &master_info)) {
            fprintf(stderr, "Failed to get master info.\n");
            ecrt_release_master(master);
            return -1;
        }
        printf("Found %u slaves on the bus.\n", master_info.slave_count);
        ecrt_release_master(master);
        return 0;
    }
    ```

3.  **`Makefile`**

    ```makefile
    all: simple_test
    simple_test: simple_test.c
    	gcc -o simple_test simple_test.c -lethercat
    clean:
    	rm -f simple_test
    ```

4.  **`Containerfile`**

    ```dockerfile
    # Use UBI 9 as the base image
    FROM registry.redhat.io/ubi9/ubi:latest

    # --- Part 1: EtherCAT Master Build ---
    # STEP 1: Install all necessary tools and libraries for the build
    RUN dnf install -y dnf-plugins-core && \
        dnf config-manager --set-enabled rhel-9-for-x86_64-rt-rpms && \
        dnf install -y \
        make gcc gcc-c++ autoconf automake libtool bzip2 diffutils file \
        kernel-rt-devel numactl-devel libcap-devel \
        && dnf clean all

    # STEP 2: Copy the source code archive from the host into the container
    WORKDIR /usr/src/
    COPY ethercat-1.6.7.tar.bz2 .

    # STEP 3: Extract the archive
    RUN tar -xvjf ethercat-1.6.7.tar.bz2

    # STEP 4: Move to the source code directory
    WORKDIR /usr/src/ethercat-1.6.7/

    # STEP 5: Apply compatibility patches to the source code
    RUN sed -i '233s/^/\/\//' master/cdev.c && \
        sed -i 's/class_create(THIS_MODULE, "EtherCAT")/class_create("EtherCAT")/' master/module.c

    # STEP 6: Build and install the EtherCAT master (inside the container)
    RUN KERNEL_SOURCE_DIR=/usr/src/kernels/$(ls /usr/src/kernels) && \
        ./configure --with-linux-dir=${KERNEL_SOURCE_DIR} --disable-8139too --enable-generic && \
        make && \
        make install && \
        echo "/usr/local/lib" > /etc/ld.so.conf.d/ethercat.conf && \
        ldconfig

    # --- Part 2: Sample Application Build ---
    # STEP 7: Move to the application directory
    WORKDIR /app

    # STEP 8: Copy the application source code
    COPY simple_test.c Makefile ./

    # STEP 9: Build the application
    RUN make

    # STEP 10: Specify the command to run
    CMD ["./simple_test"]
    ```

#### **3.2. Build the Container Image**

```bash
podman build -t my-ethercat-app .
```

-----

### Part 4: Final Execution and Verification

**Objective**: To successfully run the containerized application and communicate over the EtherCAT bus.

#### **4.1. Run the Container**

Launch the container with the necessary privileges to access the host's hardware.

```bash
podman run -it --rm \
  --network=host \
  --device=/dev/EtherCAT0 \
  --cap-add=SYS_NICE \
  --cpuset-cpus="2,3" \
  --ulimit memlock=-1:-1 \
  my-ethercat-app
```

#### **4.2. Verify Operation**

If the container starts successfully and prints the message `Found X slaves on the bus.` (where X is the number of slaves), the entire setup is complete and working correctly.
