---
title: "Connecting a Hosted UI Website to an AWS EC2 Instance: A Step-by-Step Guide"
seoTitle: "Connecting a Hosted UI Website with AWS EC2 Instance"
seoDescription: "Discover the complete guide to connecting your hosted UI website to AWS EC2, including secure HTTPS setup, EC2 configuration, and troubleshooting tips."
datePublished: Mon Apr 07 2025 08:21:22 GMT+0000 (Coordinated Universal Time)
cuid: cm96syea3000b0ajg8mri795q
slug: connecting-a-hosted-ui-website-to-an-aws-ec2-instance-a-step-by-step-guide
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1730090223184/e8d88af3-f29e-45f9-bc90-266d659a6171.webp
tags: ec2, aws, https, web-development, beginners, web-server, keploy, ssltls

---

In the world of web applications, linking a hosted user interface (UI) with a secure backend on an AWS EC2 instance is essential for creating a seamless, scalable, and secure user experience. This guide covers each step in the process, from launching and configuring an EC2 instance to implementing [HTTPS](https://keploy.io/blog/community/mocking-httpx-requests-with-respx-a-comprehensive-guide) for secure communication. By the end of this tutorial, you’ll have a practical understanding of how to connect a hosted UI to an EC2 backend, troubleshoot common issues, and maximize the power of AWS for a robust setup.

AWS EC2 instance types are categorized based on optimized use cases like compute, memory, storage, or GPU performance.  
Choosing the right instance type ensures efficient resource allocation and cost-effective application performance.

---

### **Step 1: Launch and Configure the EC2 Instance**

The first step in setting up a backend server on AWS is creating an EC2 instance suited to handle your application’s workload. With AWS, you have full control over the instance, including scaling resources as traffic grows. Here’s how we set up the instance:

1. **Select an Instance Type**: Head over to the EC2 Dashboard in AWS, where you can launch a new instance. The `t2.micro` instance type is often a good choice for smaller projects or testing environments, but as application needs grow, selecting a larger instance type may be necessary for enhanced performance.
    
2. **Assign an Elastic IP**: Once the instance is launched, assign an Elastic IP to ensure that it maintains a static public IP address, even if the instance restarts. This makes it easy for your hosted UI to consistently access the backend without needing to update the endpoint.
    
3. **Set Up Security Groups**: AWS security groups act as a virtual firewall for your EC2 instance. Configuring inbound and outbound rules in these security groups is crucial to ensure that only trusted traffic reaches your server. We configured rules to allow incoming traffic on:
    
    * **Port 80 (HTTP)**: Used to redirect traffic to HTTPS.
        
    * **Port 443 (HTTPS)**: Enables secure encrypted connections.
        
    * **Application-Specific Port (6879)**: The port where our application’s backend listens for requests.
        

By setting up these rules, we created a secure pathway for communication between our hosted UI and the EC2 instance.

---

### **Step 2: Retrieve Instance Metadata with IMDSv2**

To simplify dynamic configurations, AWS provides the Instance Metadata Service (IMDS), which is especially useful for obtaining information about the instance itself, like its public IP. AWS recently enhanced the security of this service with IMDSv2, which requires a session token to access metadata.

We created a script that retrieves the public IP address of the instance. Here’s a quick look at how it’s done:

```javascript
goCopy codefunc getPublicIP() (string, error) {
    tokenReq, err := http.NewRequest("PUT", "http://169.254.169.254/latest/api/token", nil)
    if err != nil {
        return "", err
    }
    tokenReq.Header.Set("X-aws-ec2-metadata-token-ttl-seconds", "21600")

    client := &http.Client{}
    tokenResp, err := client.Do(tokenReq)
    if err != nil {
        return "", err
    }
    defer tokenResp.Body.Close()

    token, err := io.ReadAll(tokenResp.Body)
    if err != nil {
        return "", err
    }

    req, err := http.NewRequest("GET", "http://169.254.169.254/latest/meta-data/public-ipv4", nil)
    if err != nil {
        return "", err
    }
    req.Header.Set("X-aws-ec2-metadata-token", string(token))

    resp, err := client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }

    return string(body), nil
}
```

By accessing metadata directly within the instance, this approach ensures that necessary details are always available, even if instance information changes.

---

### **Step 3: Generate and Embed** [**SSL Certificates for HTTPS**](https://keploy.io/blog/community/ssl-problem-unable-to-get-local-issuer-certificate)

A key part of connecting a UI to a backend is ensuring secure data transmission, which we accomplish by configuring HTTPS on the EC2 instance. We generated SSL certificates using OpenSSL and embedded them directly within the code.

#### **Generate SSL Certificates with OpenSSL**

We created a self-signed certificate and private key, which were then embedded within the application. This way, there’s no need to handle the certificates externally.

```javascript
bashCopy codeopenssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt -subj "/CN=yourdomain.com"
```

#### **Embed Certificates in the Code**

With Go’s `embed` package, we could easily include the certificate and key within the application:

```javascript
goCopy code//go:embed certs/server.crt
var serverCrtFile []byte

//go:embed certs/server.key
var serverKeyFile []byte
```

Embedding the certificates directly in the code makes deployment more straightforward, especially for EC2 instances that may be frequently restarted or replaced.

---

### **Step 4: Configure** [**HTTPS with TLS in Go**](https://keploy.io/blog/community/ebpf-for-tls-traffic-tracing-secure-efficient-observability)

Once the certificates were embedded, the next step was to configure HTTPS in our Go server to serve secure connections. We set up the server to serve HTTPS requests only when a public IP was available, using TLS to provide strong encryption.

Here’s how we configured the Go server to listen for secure connections on port 443:

```javascript
goCopy codeif ip != "" {
    cert, err := tls.X509KeyPair(serverCrtFile, serverKeyFile)
    if err != nil {
        log.Fatalf("Failed to load TLS certificates: %v", err)
    }

    httpSrv.TLSConfig = &tls.Config{
        Certificates: []tls.Certificate{cert},
        MinVersion:   tls.VersionTLS12,
    }

    if err := httpSrv.ListenAndServeTLS("", ""); err != http.ErrServerClosed {
        log.Fatalf("GraphQL server failed to start with HTTPS: %v", err)
    }
} else {
    if err := httpSrv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatalf("GraphQL server failed to start with HTTP: %v", err)
    }
}
```

With this configuration, the server listens over HTTPS, ensuring that all communications between the UI and the backend are encrypted.

---

### **Step 5: Testing and Troubleshooting Common Issues**

After setting up the server, thorough testing was essential to confirm that everything worked as expected.

#### **Using OpenSSL for Certificate Validation**

To ensure the certificate was properly configured, we validated it using OpenSSL:

```javascript
bashCopy codeopenssl x509 -in server.crt -text -noout
```

#### **Testing the Connection with CURL**

Using `curl`, we confirmed that the server was accessible over HTTPS:

```javascript
bashCopy codecurl -vk https://<public_ip>:6879/healthz
```

#### **Common Issues and Solutions**

1. **TLS Handshake Errors**:
    
    * This usually indicates a mismatch in TLS versions between the client and server. To resolve this, we enforced TLS 1.2 on the server.
        
2. **Connection Refused**:
    
    * This can happen if ports aren’t correctly opened. We confirmed that our security group allowed inbound traffic on all necessary ports (80, 443, 6879).
        
3. **Certificate Warnings**:
    
    * Since we used a self-signed certificate, some browsers and tools may show a warning. For production setups, consider using a certificate issued by a trusted Certificate Authority (CA).
        

---

### **Final Integration and** [**Testing with the UI**](https://keploy.io/blog/community/top-tools-for-static-analysis-in-python)

With the EC2 backend fully configured, we tested end-to-end integration with the UI to ensure seamless communication between the frontend and backend. By securing each step, from security groups to TLS encryption, we created a setup that ensures both security and reliability.

AWS offers a robust EC2 [API](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing) [thr](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing)ough which developers can control instances programmatically. You can use the API to launch, stop, start, terminate, and resize instance types without resorting to the AWS Console. This is particularly convenient for automation, scaling, and integrating EC2 management with custom applications or DevOps pipelines. All these tools such as the AWS CLI, SDKs, and CloudFormation operate using this API in the background to efficiently and securely communicate with EC2 resources.

## **How to monitor frequency** [**of**](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing) [**rep**](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing)**lacement of aws ec2 instances?**

Here's a brief 6-line tutorial to monitor the frequency of EC2 instance replacement:

* Tag instances with launch timestamps via instance metadata or automation tools.
    
* Turn on AWS CloudTrail to record EC2 instance start/terminate events.
    
* Monitor instance lifecycle changes using Amazon CloudWatch Logs/Events.
    
* Build a CloudWatch Dashboard with custom metrics or filters for instance IDs.
    
* Export logs to S3 and query with Athena or utilize AWS Config to monitor changes over time.
    
* Visualize trends (dailies, weeklies) to see trends in instance replacement frequency.
    

## **How to delete ec2 instances from aws?**

1. Go to the **EC2 Dashboard** in t[he](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing) AWS Management Console.
    
2. Select the instance you want [to](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing) delete, then click **Actions &gt; Instance State &gt; Terminate instance**.
    
3. Confirm the termination — th[e i](https://keploy.io/blog/community/test-case-generation-for-faster-api-testing)nstance will be permanently deleted along with its ephemeral data.
    

---

### **Conclusion**

Connecting a hosted UI with an AWS EC2 instance requires a thoughtful approach to security, scalability, and configuration. From launching the instance and configuring security groups to embedding certificates and troubleshooting, each step adds value to the final setup. The result is a secure, reliable, and scalable environment where users can interact with the backend through a hosted UI, confident that their data is protected. With AWS’s powerful infrastructure and EC2’s flexibility, this setup provides a robust foundation for web applications ready to grow and adapt to future needs.

## **FAQ’s**

### **1\. When I modify the instance type of an EC2 instance?**

When you modify the instance type, AWS temporarily stops the instance.

You can then change its type and restart it with the new type.

All the data in the root volume is preserved unless deleted intentionally.

Ensure that the new instance type is supported by the current AMI and networking.

### **2\. Will my Elastic IP be the same after instance type change?**

Yes, if you have an Elastic IP associated, it will remain attached during the type change.

Elastic IPs persist through instance stops and starts.

Just make sure the instance is in the same region.

Without an Elastic IP, the public IP will change on restart.

### **3\. How do I select the correct EC2 instance type for my application?**

Evaluate your workload requirements: compute-intensive, memory-hungry, or storage-intensive.

Consult AWS's instance type recommendations and perform benchmarking if necessary.

Begin with general-purpose types such as `t3` for development or low load.

Observe performance and grow vertically (change type) or horizontally (increase instances).

### **4\. Can I automate changing EC2 instance types depending on usage?**

Yes, you can do it automated with CloudWatch alarms and Lambda functions.

Set thresholds on CPU, memory, or network metrics.

When the threshold is breached, fire a Lambda script to terminate, reconfigure, and restart the instance.

Optimizes cost and performance with automated intervention.