# Branch 6 — Jenkins CD Pipeline

## Goal

This branch extends the Jenkins CI pipeline with Continuous Delivery (CD) by deploying the Docker image produced by the CI pipeline to an Amazon EC2 instance.

The previous branches established the Jenkins pipeline, shared library, dynamic versioning, and GitHub webhook integration.

This branch adds the deployment layer:

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── Build application
   ├── Build Docker image
   ├── Push Docker image
   │
   ▼
Docker Hub
   │
   │ Docker image
   ▼
Amazon EC2
   │
   ├── Stop previous container
   ├── Remove previous container
   └── Run new container
          │
          ▼
     Java Application
        Port 8080
```

## Deployment Architecture

The CI pipeline is responsible for building and publishing the Docker image. The CD extension takes that published image and deploys it to an Amazon EC2 instance.

The complete flow is:

```text
GitHub
   │
   │ Source code
   ▼
Jenkins
   │
   ├── Build Java application
   ├── Build Docker image
   ├── Login to Docker Hub
   └── Push Docker image
          │
          ▼
      Docker Hub
          │
          │ Docker image
          ▼
     Amazon EC2
          │
          ├── SSH connection from Jenkins
          ├── Stop previous container
          ├── Remove previous container
          └── Run new Docker image
                  │
                  ▼
             Application
               :8080
```

The Docker image is not copied directly from Jenkins to the EC2 instance.

Instead, Jenkins pushes the image to Docker Hub. During deployment, Docker on the EC2 instance retrieves the required image from Docker Hub if it is not already available locally.

## 1. Prepare the Amazon EC2 Instance

The deployment target for this project is an Amazon EC2 instance running Amazon Linux 2023.

The example environment used while developing this project was:

| Setting          | Value             |
| ---------------- | ----------------- |
| Operating System | Amazon Linux 2023 |
| Instance Type    | `t3.micro`        |
| AWS Region       | `us-east-1`       |

> The specific AMI ID and public IP address used during development are intentionally not included in this documentation. When following this guide, use the current Amazon Linux 2023 AMI available in your AWS region and your own EC2 public IPv4 address.

The EC2 instance must be configured so that Jenkins can connect to it through SSH and users can access the deployed application through the internet.

## 2. EC2 Networking Requirements

The EC2 instance should be placed in a subnet that has internet connectivity.

At a high level, the networking configuration should include:

```text
Internet
   │
   ▼
Internet Gateway
   │
   ▼
VPC
   │
   ▼
Public Subnet
   │
   ▼
EC2 Instance
```

An Internet Gateway must be attached to the VPC.

The route table associated with the EC2 instance's subnet should contain a default route:

```text
Destination: 0.0.0.0/0
Target:      Internet Gateway
```

This allows traffic from the subnet to reach the internet.

The EC2 instance also needs a public IPv4 address so that Jenkins and web browsers can reach it.

> An Internet Gateway alone does not make an EC2 instance publicly reachable. The subnet needs an appropriate route to the Internet Gateway, the instance needs a public IPv4 address, and the Security Group must allow the required inbound traffic.

## 3. Configure the EC2 Security Group

The EC2 instance needs a Security Group that allows the traffic required by this deployment.

For this project, the following inbound rules were used:

| Protocol |   Port | Source      | Purpose                          |
| -------- | -----: | ----------- | -------------------------------- |
| TCP      |   `22` | `0.0.0.0/0` | SSH access                       |
| TCP      | `8080` | `0.0.0.0/0` | Public access to the application |

Port `22` allows Jenkins to establish an SSH connection to the EC2 server.

Port `8080` allows users to access the Java application running inside the Docker container.

> For production environments, SSH access should normally be restricted to trusted source IP addresses instead of allowing `0.0.0.0/0`.

## 4. Install Docker on Amazon Linux 2023

Connect to the EC2 instance using the `ec2-user` account.

Update the installed packages:

```bash
sudo yum update
```

Install Docker:

```bash
sudo yum install docker
```

Start Docker:

```bash
sudo systemctl start docker
```

Enable Docker so that it starts automatically when the instance boots:

```bash
sudo systemctl enable docker
```

Add `ec2-user` to the Docker group:

```bash
sudo usermod -aG docker ec2-user
```

Log out of the EC2 instance:

```bash
exit
```

Then SSH into the instance again.

The new login session is required for the updated Docker group membership to take effect.

After reconnecting, Docker commands can be executed as `ec2-user` without using `sudo`.

Authenticate the EC2 instance with Docker Hub so it can pull the private application image during deployment:

```bash
docker login
```
## 5. Configure Jenkins SSH Credentials

Jenkins needs SSH access to the EC2 instance so that the deployment stage can execute Docker commands remotely.

In Jenkins, create a new credential with the following configuration:

| Setting       | Value                                      |
| ------------- | ------------------------------------------ |
| Type          | `SSH Username with private key`            |
| Scope         | `Global`                                   |
| Username      | `ec2-user`                                 |
| Credential ID | `ec2-server-key`                           |
| Private Key   | Private key generated for the EC2 key pair |

The private key should be added to Jenkins securely. Do not commit the private key to GitHub or include it in the repository.

The credential ID is then referenced by the Jenkins shared-library deployment function.

## 6. Deploy the Docker Image to EC2

The deployment logic is implemented in the Jenkins shared library in:

```text
jenkins-shared-library/
└── vars/
    └── ec2Deploy.groovy
```

The application repository's `Jenkinsfile` calls the shared-library function:

```groovy
stage("deploy and run image on EC2 server") {
    steps {
        script {
            ec2Deploy()
        }
    }
}
```

The `ec2Deploy()` function connects to the EC2 instance through SSH and executes the Docker deployment commands remotely.

The shared-library implementation is:

```groovy
def call() {
    echo "Deploying & running server on EC2..."

    def ec2Server = 'ec2-user@<ec2-public-ip>'

    def dockerCmd = "docker stop demo-app || true && docker rm demo-app || true && docker run --name demo-app -d -p 8080:8080 ${env.IMAGE_NAME}"

    sshagent(credentials: ['ec2-server-key']) {
        sh "ssh -o StrictHostKeyChecking=no ${ec2Server} '${dockerCmd}'"
    }
}
```

### What the deployment function does

The function performs the following operations:

1. Connects to the EC2 instance using SSH.
2. Uses the Jenkins credential `ec2-server-key` to authenticate.
3. Stops the existing `demo-app` container.
4. Removes the existing `demo-app` container.
5. Starts a new container using the Docker image stored in `env.IMAGE_NAME`.
6. Maps EC2 port `8080` to container port `8080`.

The `|| true` expressions prevent the deployment from failing when the previous container does not exist.

The EC2 public IPv4 address is represented by `<ec2-public-ip>` in this documentation. Do not commit a personal, temporary, or production server IP address to the public repository.

## 7. Add the Deployment Stage to Jenkins

The Jenkinsfile calls the shared-library deployment function in a dedicated deployment stage:

```groovy
stage("deploy and run image on EC2 server") {
    steps {
        script {
            ec2Deploy()
        }
    }
}
```

At this point, the pipeline extends beyond Continuous Integration.

The pipeline now performs both CI and CD:

```text
CI
│
├── Build application
├── Build Docker image
└── Push image to Docker Hub
        │
        ▼
CD
│
└── Deploy image to EC2
```

## 8. How EC2 Gets the Docker Image

Jenkins does not transfer the Docker image directly to EC2.

The image is first pushed to Docker Hub:

```text
Jenkins
   │
   │ docker push
   ▼
Docker Hub
```

The deployment command then runs the image on EC2:

```bash
docker run --name demo-app -d -p 8080:8080 <docker-image>
```

If the image is not already present on the EC2 instance, Docker automatically pulls it from Docker Hub.

The resulting flow is:

```text
Jenkins
   │
   │ Push image
   ▼
Docker Hub
   │
   │ Pull image
   ▼
EC2 Docker
   │
   ▼
demo-app container
```

## 9. Verify the Deployment

After Jenkins completes the deployment stage, connect to the EC2 instance and verify that the container is running:

```bash
docker ps
```

The `demo-app` container should be listed with port `8080` mapped to port `8080` on the EC2 instance.

The application can then be accessed from a browser using the EC2 instance's public IPv4 address:

```text
http://<ec2-public-ip>:8080
```

The application is now running on Amazon EC2 as a Docker container and is accessible through the internet.

## Result

The project has now evolved from a Jenkins CI pipeline into a CI/CD pipeline:

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── Build
   ├── Package
   ├── Docker Build
   ├── Docker Push
   │
   ▼
Docker Hub
   │
   ▼
Amazon EC2
   │
   └── Docker Container
          │
          ▼
     Java Application
        Port 8080
```

The CI pipeline produces and publishes the Docker image, while the CD pipeline deploys that image to the EC2 server.

