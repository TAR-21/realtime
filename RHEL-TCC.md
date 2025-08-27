## A Guide to Enabling Intel TCC on RHEL and Verifying Real-Time Performance with a Podman Container

This document provides a complete procedure for setting up RHEL for Real Time on a machine with an Intel TCC-supported CPU and verifying its real-time performance using an application inside a Podman container.

-----

### Part 1: Host OS Real-Time Environment Setup

**Objective**: To tune the hardware and OS to reserve specific CPU cores as a "sanctuary" for real-time tasks.

#### **Step 1. BIOS/UEFI Configuration (Hardware-Level Preparation)**

**Objective**: To switch the CPU itself into a "full performance mode" optimized for real-time processing before the OS boots.

A typical PC's CPU is designed to balance performance and power consumption. For example, it automatically enters sleep modes (C-states) to save power when idle and dynamically changes its clock frequency (P-states / Turbo Boost) as needed.

However, for real-time processing, these power-saving features and dynamic performance adjustments are sources of "unpredictable latency" (jitter). The time it takes to wake from sleep or for the clock frequency to stabilize is a significant source of noise when precision in the nanosecond range is required.

**Enabling "Intel® TCC Mode"** is the act of eliminating or optimizing these sources of variability at the hardware level.

**Specific Steps**:

1.  **Accessing the BIOS/UEFI**: Immediately after powering on the PC, press the specific key displayed on the manufacturer's logo screen (often `F2`, `Del`, `F10`, or `Esc`) to enter the setup utility.
2.  **Finding the Setting**: BIOS menus vary greatly by manufacturer and model, but the setting is typically located in one of the following sections:
      * `Advanced` -\> `CPU Configuration`
      * `Performance` / `Overclocking`
      * `Platform Settings`
3.  **Enabling and Saving**: Change the "Intel TCC Mode" or a similarly named option like "Real-Time Mode" to **Enabled**. Save the configuration (often with the `F10` key) and reboot.

Only after completing this step is the CPU ready to respond to real-time requests from the OS with a consistent, predictable response time.

#### **Step 2. OS Preparation (OS-Level Preparation)**

**Objective**: To install a special OS kernel designed to handle real-time tasks with the highest priority.

Standard operating system kernels (like those in Windows, macOS, and regular Linux) are built with "fairness" as a primary goal. They are good at sharing CPU time fairly among various tasks—a web browser, a music player, a background virus scan—to make it seem like all applications are running smoothly.

However, this "fairness" is a problem for real-time systems where some tasks absolutely cannot be late.

**Installing "RHEL for Real Time"** is the process of replacing the standard "fairness-based" kernel with a **real-time kernel (`kernel-rt`)**, which is built on the philosophy that "priority is everything."

**Main Features of the Real-Time Kernel**:

  * **Fully Preemptible**: A high-priority task can interrupt a lower-priority task almost anywhere, even in the middle of a kernel operation that would be non-interruptible in a standard kernel. This dramatically reduces task latency.
  * **Priority-Based Scheduling**: The scheduler makes decisions based on the absolute priority number assigned to a task, rather than trying to be "fair."

**Specific Steps**:
RHEL for Real Time is less a completely separate OS and more of an add-on package group that provides real-time capabilities to RHEL.

1.  **Enable the Repository**: First, you must tell the package manager (`dnf`) where to find the real-time packages.
    ```bash
    sudo subscription-manager repos --enable rhel-9-for-x86_64-rt-rpms
    ```
2.  **Install the Package Group**: Next, install the real-time kernel itself along with a set of related utilities.
    ```bash
    sudo dnf groupinstall "RT"
    ```
3.  **Reboot and Verify**: After the installation, reboot the system and confirm that you are running on the new real-time kernel.
    ```bash
    uname -r
    ```
    If the output version string ends with **`-rt`**, the installation was successful.

This step establishes the OS-level foundation for prioritizing real-time tasks above all else.

#### **Step 3. Kernel Tuning (CPU Core Isolation)**

  - Use the `grubby` command to isolate specific CPU cores from the OS's general-purpose scheduler.
  - **Example (Isolating cores 2 & 3)**:
    ```bash
    sudo grubby --update-kernel=ALL --args="isolcpus=2-3 nohz_full=2-3 rcu_nocbs=2-3"
    ```
  - **Apply and Verify**:
    Reboot the system. After rebooting, run `cat /proc/cmdline` to confirm that the parameters have been applied.

#### **Step 4. Applying the `tuned` Profile**

  - Optimize the entire OS for real-time operations.
  - **Command**:
    ```bash
    sudo tuned-adm profile realtime
    ```
  - **Verify**:
    Run `tuned-adm active` and check that the output shows `Current active profile: realtime`.

-----

### Part 2: Creating the Real-Time Verification Container

**Objective**: To build the standard real-time measurement tool, `cyclictest`, from its source code and package it into a container image.

#### **Step 1. Create the `Containerfile`**

  - Create a file named `Containerfile` with the following content. This allows you to install the `rt-tests` suite without relying on special subscriptions.
    ```dockerfile
    # Use UBI 9 as the base image
    FROM registry.redhat.io/ubi9/ubi:latest

    # STEP 1: Install the tools and libraries required to build rt-tests
    RUN dnf install -y \
        git \
        make \
        gcc \
        procps-ng \
        numactl-devel \
        libcap-devel \
        && dnf clean all

    # STEP 2: Download, build, and install the rt-tests source code
    RUN git clone https://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git && \
        cd rt-tests && \
        make all && \
        make install

    # Set the default command
    CMD ["cyclictest", "--help"]
    ```

#### **Step 2. Build the Podman Image**

  - Using the `Containerfile` above, build a container image named `rt-test-app`.
    ```bash
    sudo podman build -t rt-test-app .
    ```

-----

### Part 3: Performance Measurement and Verification

**Objective**: To prove that the performance of the real-time task running on isolated cores does not degrade, even while the rest of the OS is under heavy load.

#### **Step 1. Install the Stress Tool (on the Host OS)**

  - Install the `stress-ng` load generation tool on the host OS.
    ```bash
    sudo dnf install -y epel-release
    sudo dnf install -y stress-ng
    ```

#### **Step 2. Running the Performance Test (Using Two Terminals)**

  - **Terminal ① - Start `cyclictest`**:
    Launch `cyclictest` with real-time privileges, pinned to the isolated cores (2,3).

    ```bash
    podman run --rm --cap-add=SYS_NICE --cpuset-cpus="2,3" --ulimit memlock=-1:-1 rt-test-app chrt -f 80 cyclictest -t1 -p 80 -i 1000 -m
    ```

  - **Terminal ② - Apply Load with `stress-ng`**:
    Apply heavy load to the **non-isolated CPU cores (e.g., 0,1)**, where `cyclictest` is not running.

    ```bash
    taskset -c 0,1 stress-ng --cpu 2 --vm 2 --timeout 120s
    ```

#### **Step 3. Evaluating the Results**

  - While `stress-ng` is running, observe the `Max` (maximum latency) value in the output of `cyclictest` in Terminal ①.
  - **Success Criteria**: The `Max` value should **remain consistently low (typically between a few to \~20 microseconds)**, even while the host OS is under heavy load.
  - If this is the case, it serves as proof that the kernel tuning has successfully protected the real-time task from the influence of other tasks on the system.
