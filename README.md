# CMPE-283 - Virtual Technologies Assignment-2

## Team Members

1. Nihal Kaul (016697512)
2. Poojan Shah (016583528)

## Contribution

1. Nihal Kaul

- Setup GCP
- Cloned and built linux kernel source
- Updated cpuid.c and vmx.c files to find out total exits and total cycles

2. Poojan Shah

- Setup nested virtual machine using virt-install
- Tested the updated linux kernel source using the cpuid package inside the nested virtual machine
- Debugged errors that occured after updating vmx.c and cpuid.c files

## Getting started guide

### Google Cloud Platform

Setup Google Cloud CLI on your local machine to manage your compute engine resources via commands. Spin up a Linux instance by running the following command,

`gcloud compute instances create [YOUR_INSTANCE_NAME] --machine-type=n2-standard-8 --boot-disk-size=200 --enable-nested-virtualization`

Replace "YOUR_INSTANCE_NAME" with the name of your instance.

In the above command we are not specifying the zone since in our case the zone was configured and set to "us-west1-b" while installing Google Cloud CLI.

We are enabling nested virtualization by adding the flag "--enable-nested-virtualization".

### SSH into your instance

You can ssh into your instance using the following command,

`gcloud compute ssh --zone "YOUR_ZONE" "YOUR_INSTANCE_NAME" --project "YOUR_PROJECT_NAME"`

Once you are inside the linux instance, you need to setup a couple of tools in order to execute and build the linux source tree.

### Install gcc and make

`sudo apt install gcc make`

### Clone a forked version of the linux GitHub repo

`git clone https://github.com/nihalwashere/linux`

### Build the linux source tree

Copy your instance's linux distribution config file to your cloned linux repo

`sudo cp /boot/config-5.10.0-19-cloud-amd64 .config`

### Read the existing config file that was used to build the old kernel, you will be prompted for options in the current kernel source

`make oldconfig`

#### Prepare for architecture

`make prepare`

#### Build the modules

`make -j 8 modules`

#### Build the source

`make -j 8`

#### Install modules

`sudo make INSTALL_MOD_STRIP=1 modules_install`

#### Install source

`sudo make install`

#### Reboot your instance

`sudo reboot`

#### Now you should be able to see your updated linux version when you run the below command

`uname -a`

### Install virt-install

`virt-install` is a command line tool to provision new virtual machines

### Create a nested virtual machine

The below command will help you to create a virtual machine named 'ubuntu-1' with two virtual CPU's and 2 GB of RAM, the OS variant is ubuntu20.04. Since we are operating through the CLI hence we are specifying `--graphics none` and we would like to access the nested virtual machine through the serial console.

`sudo virt-install --name ubuntu-1 --os-variant ubuntu20.04 --vcpus 2 --ram 2048 --location http://ftp.ubuntu.com/ubuntu/dists/focal/main/installer-amd64/ --network bridge=virbr0,model=virtio --graphics none --extra-args='console=ttyS0,115200n8 serial'`

Follow the prompts on your screen to finish setting up your nested virtual machine.

#### List your virtual machines

`virsh list --all`

#### Access your nested virtual machine through the console

`virsh console ubuntu-1`

#### Now you should be able to access the prompt of the nested virtual machine

`ubuntu-1`

### Update your `kvm` module

Follow the guidelines as described in the assignment and make changes to the `linux/arch/x86/kvm/cpuid.c` and `linux/arch/x86/kvm/vmx/vmx.c` files.

After modification, run the below commands to build your kernel source tree for the changes to take effect.

1. `make -j 8 modules`

2. `make -j 8`

3. `sudo make INSTALL_MOD_STRIP=1 modules_install`

### Build your `kvm` module

Check if `kvm` module is already loaded

`lsmod | grep kvm`

Remove `kvm` if already loaded

1. `sudo rmmod  kvm_intel`
2. `sudo rmmod  kvm`

Load updated kvm modules

1. `sudo modprobe kvm`
2. `sudo modprobe kvm_intel`

## Test your code

### Install `cpudid` package on your nested virtual machine

1. `sudo apt-get update`

2. `sudo apt-get install cpuid`

### Test leaf node 0x4FFFFFFC

Run the following command inside your nested virtual machine

`cpuid -1 -l 0x4ffffffc`

You should be able to see an output like below

![alt text](./output1.png)

### Test leaf node 0x4FFFFFFD

Run the following command inside your nested virtual machine

`cpuid -1 -l 0x4ffffffd`

You should be able to see an output like below

![alt text](./output2.png)

### Test leaf node 0x4FFFFFFE

Run the following `C` script inside your nested virtual machine. We are testing the exit type 0 (check input value for ecx).

```
#include <stdio.h>
#include <sys/types.h>

static inline void
__cpuid(unsigned int *eax, unsigned int *ebx, unsigned int *ecx,
unsigned int *edx)
{
    asm volatile("cpuid"
    : "=a" (*eax),
    "=b" (*ebx),
    "=c" (*ecx),
    "=d" (*edx)
    : "0" (*eax), "1" (*ebx), "2" (*ecx), "3" (*edx));
}

int main(int argc, char **argv)
{
    unsigned int eax, ebx, ecx, edx;
    unsigned long long time;

    eax = 0x4FFFFFFE;
    ecx = 0;
    __cpuid(&eax, &ebx, &ecx, &edx);
    printf("######################\n");
    printf("CPUID(0x4FFFFFFE), EXIT=%u, EXIT CYCLES=%u\n", ecx, eax);
}
```

You should be able to see an output like below

![alt text](./exit0.png)

### Test leaf node 0x4FFFFFFF

Run the following `C` script inside your nested virtual machine. We are testing the exit type 0 (check input value for ecx).

```
#include <stdio.h>
#include <sys/types.h>

static inline void
__cpuid(unsigned int *eax, unsigned int *ebx, unsigned int *ecx,
unsigned int *edx)
{
    asm volatile("cpuid"
    : "=a" (*eax),
    "=b" (*ebx),
    "=c" (*ecx),
    "=d" (*edx)
    : "0" (*eax), "1" (*ebx), "2" (*ecx), "3" (*edx));
}

int main(int argc, char **argv)
{
    unsigned int eax, ebx, ecx, edx;
    unsigned long long time;

    eax = 0x4FFFFFFE;
    ecx = 0;
    __cpuid(&eax, &ebx, &ecx, &edx);
    printf("######################\n");
    printf("CPUID(0x4FFFFFFF), EXIT PROCESSING TIME HIGH 32 BITS=%u, EXIT PROCESSING TIME LOW 32 BITS=%u\n", ebx, ecx);
}
```

You should be able to see an output like below

![alt text](./exit0_processing_time.png)

## Questions

1. Comment on the frequency of exits – does the number of exits increase at a stable rate? Or are there
   more exits performed during certain VM operations? Approximately how many exits does a full VM
   boot entail?

- Upon observing, we noticed that the frequency of exits increases gradually. There are more exits whenever a VM is booting. On testing we found that the total number of exits during a full VM boot is `956591`. But of course this is not a hard number and it will change on every VM boot. The total number of exits could be more or less.

![alt text](./full_vm_boot.png)

2. Of the exit types defined in the SDM, which are the most frequent? Least?

- The most frequent exit type is `48` and the least is `47`, however, there are a lot of exit types which are not executed at all and their exit frequency is `0`.

Output for exit reason 47

![alt text](./exit47.png)

Output for exit reason 48

![alt text](./exit48.png)
