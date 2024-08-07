---
title: "BitBucket Self-Hosting : Running ebpf/Privileged programs"
seoTitle: "Overcoming Bitbucket CI/CD Limitations with Docker-in-Docker and Linux"
seoDescription: "Explore how to overcome Bitbucket CI/CD pipeline limitations with Docker-in-Docker and Linux shell scripts. Learn to handle root privileges, mount volumes,"
datePublished: Wed Aug 07 2024 07:35:55 GMT+0000 (Coordinated Universal Time)
cuid: clzjjbyb4001v08l7e5unhp9t
slug: bitbucket-self-hosting-running-ebpfprivileged-programs
canonical: https://keploy.io/blog/technology/bitbucket-self-hosting-running-ebpfprivileged-programs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1721114797783/56e833e5-9b1d-49f1-ba16-c30d09225bd8.png
tags: linux, docker, self-hosted, bitbucket, ebpf, bitbucket-pipelines, bitbucket-self-hosted-runner

---

Bitbucket is a robust tool for source code management and continuous integration/continuous deployment (CI/CD). It offers flexibility in setting up pipelines, but there are limitations, particularly around root privileges and mounting volumes. This blog explores these limitations and provides detailed solutions using Docker-in-Docker (DinD) and Linux shell scripts.

## What is Bitbucket?

Bitbucket is a Git repository management solution designed for professional teams. It provides source code hosting, version control, and CI/CD capabilities. With Bitbucket Pipelines, you can automate the build, test, and deployment processes, ensuring your code moves through the development stages seamlessly.

**Bitbucket Pipelines:**

* **Automate Workflows:** Automate your CI/CD workflows directly in Bitbucket.
    
* **Customizable Pipelines:** Define custom pipelines using YAML configuration files.
    
* **Integration with Jira:** Seamlessly integrate with Jira for tracking and managing issues.
    

Bitbucket Pipelines enable developers to define and manage their build, test, and deployment workflows using YAML files. These pipelines run in Docker containers, providing a consistent environment for your code. However, when working with these pipelines, you might encounter some limitations, particularly if your tasks require elevated permissions or volume mounts.

## Understanding Docker-in-Docker (DinD)

Docker-in-Docker (DinD) allows you to run Docker commands within a Docker container. This is particularly useful for CI/CD pipelines where you need to build, test, or deploy Docker images. However, DinD has some limitations, especially around mounting volumes.

**Concept of DinD:**

* **Nested Docker:** Run Docker inside a Docker container.
    
* **Isolation:** Provides an isolated environment for running Docker commands.
    
* **Limitations:** Volume mounting is not supported, and it requires privileged mode to run.
    

Running Docker commands within a Docker container can be incredibly powerful, but it also introduces complexity. The main challenge with DinD is the need for privileged mode, which allows the inner Docker daemon to operate. Without this mode, the Docker daemon inside the container cannot function correctly, leading to build failures.

## How to work with Bitbucket Pipelines?

Bitbucket Pipelines provide a seamless integration of CI/CD into your repository. By defining your build, test, and deployment steps in a `bitbucket-pipelines.yml` file, you can automate your workflows and ensure consistency across your development process. The default `.yml` looks like :

```yaml
# This is an example Starter pipeline configuration
# Use a skeleton to build, test and deploy using manual and parallel steps
# -----
# You can specify a custom docker image from Docker Hub as your build environment.

image: node:22

pipelines:
  default:
    - parallel:
        - step:
            name: Build and Test
            caches:
              - node
            script:
              - npm install
              - npm test
        - step:
            name: Deploy
            script:
              - npm run deploy
```

In this example:

* **image:** Specifies the Docker image to use for the pipeline. By default it uses `atlassians/default-image`
    
* **caches:** Caches dependencies between pipeline runs to speed up the process.
    
* **script:** Contains the commands to execute in each step.
    

Bitbucket Pipelines support multiple steps and parallel execution, allowing you to break down complex workflows into manageable parts. By defining different steps for build, test, and deployment, you can ensure each phase is completed successfully before moving to the next.

## Setting Up Docker-in-Docker in Bitbucket

To use DinD in Bitbucket Pipelines, you need to set up a privileged Docker service. Here's how you can configure it:

```yaml
image: atlassian/default-image:2

definitions:
  services:
    docker:
      image: docker:dind

pipelines:
  default:
    - step:
        runs-on:
          - 'self.hosted'
        name: Build and Test
        services:
          - docker
        script:
          - docker build -t my-app .
          - docker run my-app
```

In this setup:

* **image:** Specifies the Docker version to use.
    
* **options:** Enables Docker support in the pipeline.
    
* **services:** Adds the Docker service with privileged mode enabled.
    
* **script:** Contains the commands to build and run your Docker image.
    

This configuration ensures that your pipeline can run Docker commands by enabling Docker support and setting the service to run in privileged mode. The `script` section includes the actual commands for building and running your Docker images.

Running privileged containers allows you to perform actions that require elevated permissions, such as using `sudo`. Here’s how you can enable privileged mode in your pipeline:

```yaml
image: atlassian/default-image:2

definitions:
  services:
    docker:
      image: docker:dind

pipelines:
  default:
    - step:
        runs-on:
          - self.hosted
        name: Build with Privileges
        services:
          - docker
        script:
          - docker build -t my-privileged-app .
          - docker run --privileged my-privileged-app
        services:
          - docker
```

In this example:

* **privileged:** Set to true to enable privileged mode for the Docker service.
    
* **\--privileged:** Ensures the container runs with elevated permissions.
    

Privileged mode is crucial for running certain commands and tasks that require higher levels of access. By configuring your pipeline to use a privileged Docker service, you can overcome many of the limitations imposed by the default security settings.

## Using Linux Shell Scripts

If you need to mount volumes or perform other privileged operations, using Linux shell scripts is the way to go. This method provides greater flexibility and control over your build environment.

### **Setting Up a Linux Shell Script Pipeline:**

1. **Add**`linux.shell` to **your Bitbucket pipeline :**
    

```yaml
image: atlassian/default-image:2

pipelines:
  default:
    - step:
        runs-on:
          - self.hosted
          - linux.shell

        name: Setup Environment and Build
        script:
          - npm install -y
          - npm run build && npm run serve
```

In this example:

* **atlassian/default-image:2:** Provides a base image with necessary tools for running shell scripts.
    

Using `Linux shell` scripts in Bitbucket Pipelines allows you to leverage the full power of the Linux environment. You can run `sudo` commands, install additional packages, and mount volumes as needed, providing a high degree of customization for your build process.

## Practical Use Cases and Tips

1. **Customizing the Build Environment:**
    
    * If your project requires specific tools or dependencies that are not available in default Docker images, using Linux shell scripts with sudo privileges allows you to customize your environment.
        
2. **Persistent Data Storage:**
    
    * Mounting volumes is crucial for scenarios requiring persistent data storage or sharing data between different pipeline steps. This approach is not possible with DinD, making the Linux shell script method more versatile.
        
3. **Security Considerations:**
    
    * Always be cautious with sudo commands and ensure your scripts are secure. Avoid running unnecessary commands with elevated privileges to reduce security risks.
        

## Advanced Configuration with DinD

While DinD is powerful, it requires careful setup to ensure it runs smoothly. Here are some advanced tips for configuring DinD in Bitbucket Pipelines:

1. **Optimizing Memory Usage:**
    
    * Running Docker inside Docker can be resource-intensive. Allocate sufficient memory to the Docker service to prevent build failures.
        
2. **Handling Nested Containers:**
    
    * Be mindful of the nested nature of DinD. Ensure your inner Docker containers have access to the necessary resources and network settings.
        
3. **Debugging Build Failures:**
    
    * Use logging and debugging tools to identify issues within your DinD setup. Proper logging can help trace problems and optimize your pipeline configuration.
        

## Using Linux Shell Scripts for Maximum Flexibility

Linux shell scripts provide unmatched flexibility for customizing your build environment. Here’s a more detailed example of how you can leverage shell scripts in your pipeline:

**Advanced Shell Script Setup:**

1. **Create an environment setup script (env-setup.sh):**
    

```sh
#!/bin/bash

# Update package list and install dependencies
sudo apt-get update
sudo apt-get install -y curl git

# Mount a volume
sudo mount /dev/sdx /mnt/my-volume

# Set environment variables
export MY_VAR="LINUX_SHELL"

# Run custom commands
./custom-script.sh
```

2. **Reference the setup script in your Bitbucket pipeline:**
    

```yaml
image: ubuntu:20.04

pipelines:
  default:
    - step:
       runs-on:
          - self.hosted
          - linux.shell

        name: Environment Setup
        script:
          - chmod +x env-setup.sh
          - ./env-setup.sh
```

In this setup:

* **env-setup.sh:** Script to update the package list, install dependencies, mount a volume, set environment variables, and run custom commands.
    
* **ubuntu:20.04:** Base image providing a full Linux environment.
    

## Conclusion

Self-hosting in Bitbucket offers flexibility but comes with challenges around sudo privileges and volume mounting. By understanding the limitations and capabilities of Docker-in-Docker and Linux shell scripts, you can choose the best approach for your project's needs. Whether you prioritize the ease of running Docker commands with DinD or the versatility of mounting volumes and running custom scripts with Linux shell, Bitbucket Pipelines can be tailored to fit your development workflow.

For more detailed information and references, check out the [Bitbucket documentation on running Docker commands](https://support.atlassian.com/bitbucket-cloud/docs/run-docker-commands-in-bitbucket-pipelines/) and the [Atlassian Community discussion on privileged flags](https://community.atlassian.com/t5/Bitbucket-questions/How-to-run-anything-that-requires-privileged-flag-in-my-self/qaq-p/1982343).

## FAQs

### **What is the primary difference between using Docker-in-Docker (DinD) and Linux shell scripts in Bitbucket Pipelines?**

The main difference lies in the capabilities and limitations of each method. Docker-in-Docker (DinD) allows you to run Docker commands within a Docker container, providing an isolated environment for building and testing Docker images. However, DinD does not support volume mounting and requires privileged mode to function correctly. On the other hand, Linux shell scripts allow you to run `sudo` commands, mount volumes, and customize your build environment extensively, providing greater flexibility and control.

### **Why do I need privileged mode to use Docker-in-Docker (DinD) in Bitbucket Pipelines?**

Privileged mode is necessary for Docker-in-Docker because it grants the inner Docker daemon the required permissions to operate inside a container. Without privileged mode, the Docker daemon inside the container would lack the necessary permissions to run, resulting in build failures. Enabling privileged mode allows the inner Docker instance to function as if it were running on a host machine.

### **Can I mount volumes using Docker-in-Docker (DinD) in Bitbucket Pipelines?**

No, mounting volumes is not supported with Docker-in-Docker in Bitbucket Pipelines. This limitation is due to the nested nature of DinD, which prevents the inner Docker instance from accessing and mounting external volumes. If your build process relies on volume mounts, using Linux shell scripts is a better option as it allows you to mount volumes and perform other privileged operations.

### **How can I run**`sudo` commands in Bitbucket Pipelines?

To run `sudo` commands in Bitbucket Pipelines, you need to use a Linux shell script while self hosting. Create a script with the necessary `sudo` commands and reference it in your pipeline configuration. For example:

```sh
#!/bin/bash

# Example of a setup script that requires sudo
sudo apt-get update
sudo apt-get install -y some-package

# Mount a volume
sudo mount /dev/sdx /mnt/my-volume

# Run your build commands
./build.sh
```

Then, reference the script in your `bitbucket-pipelines.yml` file:

```yaml
image: atlassian/default-image:2

pipelines:
  default:
    - step:
        name: Setup Environment and Build
        script:
          - chmod +x setup.sh
          - ./setup.sh
```

### **What are the security considerations when using privileged mode and**`sudo` commands in Bitbucket Pipelines?

Security is a critical consideration when using privileged mode and `sudo` commands. Here are some best practices:

* **Minimize Elevated Permissions:** Only use privileged mode and `sudo` commands when absolutely necessary.
    
* **Secure Scripts:** Ensure your scripts are secure and do not contain sensitive information or commands that could be exploited.
    
* **Limit Access:** Restrict access to your Bitbucket repository and pipeline configurations to trusted team members.
    
* **Regular Audits:** Perform regular security audits to identify and mitigate potential risks associated with elevated permissions.
    

### **How can I troubleshoot issues with Docker-in-Docker or Linux shell scripts in Bitbucket Pipelines?**

Troubleshooting issues with Docker-in-Docker or Linux shell scripts involves a few steps:

* **Check Logs:** Review the pipeline logs for error messages and stack traces to identify the root cause of the problem.
    
* **Enable Debugging:** Add debugging commands (e.g., `echo` statements) to your scripts to output the values of variables and the results of commands.
    
* **Test Locally:** Run your Docker commands or shell scripts locally to ensure they work outside of Bitbucket Pipelines.
    
* **Consult Documentation:** Refer to the [Bitbucket documentation on running Docker commands](https://support.atlassian.com/bitbucket-cloud/docs/run-docker-commands-in-bitbucket-pipelines/) and the [Atlassian Community discussions](https://community.atlassian.com/t5/Bitbucket-questions/How-to-run-anything-that-requires-privileged-flag-in-my-self/qaq-p/1982343) for guidance and solutions to common issues.
    
* **Seek Help:** If you’re unable to resolve the issue, seek help from the Bitbucket Community or support channels, providing detailed information about your configuration and the errors encountered.