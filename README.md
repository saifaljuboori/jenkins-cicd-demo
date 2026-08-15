# Branch 1 — Install Jenkins on DigitalOcean

## Goal

Run Jenkins as a Docker container on an Ubuntu DigitalOcean Droplet and initialize Jenkins for use as a CI/CD server.

## Prerequisites

This project assumes that a DigitalOcean Droplet has already been provisioned and is accessible through SSH.

The DigitalOcean Droplet creation, SSH key configuration, firewall setup, and initial SSH connection are documented in the [digitalocean-java-app-deployment](https://github.com/saifaljuboori/digitalocean-java-app-deployment) repository.

### DigitalOcean Droplet Configuration

The Droplet used for this project was configured with:

* **Operating System:** Ubuntu Linux
* **CPU:** 4 vCPUs
* **Memory:** 8 GB RAM
* **Jenkins Port:** 8080

For this demo environment, a Droplet with at least 2 vCPUs and 4 GB RAM is recommended to provide sufficient resources for Jenkins and the Docker-based CI/CD workloads used in later branches.

## Project Scope

Once the Droplet is available, this branch covers:

* Installing Docker on the Droplet
* Running Jenkins as a Docker container
* Persisting Jenkins data
* Accessing Jenkins through the browser
* Completing the initial Jenkins setup
* Verifying that Jenkins is running successfully

## Architecture

```text
DigitalOcean Droplet
        │
        ▼
Ubuntu Linux
        │
        ▼
Docker Engine
        │
        ▼
Jenkins Container
        │
        ├── Jenkins Web UI :8080
        ├── Jenkins agent port :50000
        └── Persistent Jenkins data
```

## 1. Install Docker Engine

Jenkins will run as a Docker container on the DigitalOcean Droplet. Docker Engine must therefore be installed directly on the Ubuntu server first.

This project uses Docker's **official APT repository** rather than Ubuntu's `docker.io` package. This provides Docker Engine (`docker-ce`), the Docker CLI, containerd, Buildx, and Docker Compose.

### 1.1 Update the package index

```bash
sudo apt-get update
```

### 1.2 Install required packages

Install the packages required to add Docker's official repository:

```bash
sudo apt-get install ca-certificates curl
```

### 1.3 Add Docker's official GPG key

Create the directory used to store APT repository keys:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's official GPG key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
    -o /etc/apt/keyrings/docker.asc
```

Make the key readable by APT:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 1.4 Add Docker's official APT repository

Create the Docker repository configuration:

```bash
echo \
  "Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc" | \
  sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null
```

The command above adds Docker's official Ubuntu repository to the server.

Update the package index again so APT can retrieve packages from the new repository:

```bash
sudo apt-get update
```

### 1.5 Install Docker Engine

Install Docker Engine, the Docker CLI, containerd, Buildx, and Docker Compose:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.6 Verify the Docker installation

Check the Docker version:

```bash
docker --version
```

The exact version may differ from the version used in this project because Docker continuously releases updates.

Verify that the Docker service is running:

```bash
sudo systemctl status docker
```

Docker should show an active/running status.

Finally, run Docker's test container:

```bash
sudo docker run hello-world
```

A successful execution confirms that Docker Engine can start containers correctly.

At this point, Docker Engine is installed directly on the Ubuntu Droplet and is ready to host the Jenkins container.

## 2. Run Jenkins as a Docker Container

Jenkins will run as a Docker container on the Droplet.

The Jenkins container will use:

* A persistent Docker volume for Jenkins data
* Port `8080` for the Jenkins Web UI
* Port `50000` for Jenkins inbound agents
* The host Docker socket so Jenkins can communicate with the Docker Engine installed on the Droplet

### 2.1 Create a Persistent Jenkins Volume

Create a Docker volume for Jenkins:

```bash
docker volume create jenkins_home
```

Verify that the volume was created:

```bash
docker volume ls
```

The output should include:

```text
local     jenkins_home
```

The volume will later be mounted inside the Jenkins container at:

```text
/var/jenkins_home
```

This keeps Jenkins configuration, jobs, plugins, credentials, and other Jenkins data persistent if the container is restarted or recreated.

### 2.2 Run Jenkins

Run the Jenkins container:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

The options above have the following purpose:

| Option                                         | Purpose                                                          |
| ---------------------------------------------- | ---------------------------------------------------------------- |
| `-d`                                           | Runs Jenkins in the background                                   |
| `--name jenkins`                               | Gives the container a predictable name                           |
| `-p 8080:8080`                                 | Makes the Jenkins Web UI available on port `8080`                |
| `-p 50000:50000`                               | Exposes the Jenkins inbound agent port                           |
| `-v jenkins_home:/var/jenkins_home`            | Persists Jenkins data                                            |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Gives Jenkins access to the Docker Engine running on the Droplet |
| `jenkins/jenkins:lts`                          | Uses the Jenkins LTS Docker image                                |

> **Security note:** Mounting `/var/run/docker.sock` gives processes inside the Jenkins container access to the Docker daemon running on the Droplet. Docker daemon access is highly privileged and can provide root-equivalent control over the host. This configuration is used here for the purposes of this demonstration project. Production environments should evaluate more isolated Docker execution approaches.

### 2.3 Verify the Jenkins Container

Check that the Jenkins container has started:

```bash
docker ps
```

The container should appear with a status similar to:

```text
Up ...
```

You can also view the Jenkins startup logs:

```bash
docker logs jenkins
```

Wait for Jenkins to finish starting before continuing.

### 2.4 Retrieve the Initial Administrator Password

Jenkins generates an initial administrator password during the first startup.

Retrieve it from the running container:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the generated password.

Jenkins' official documentation confirms that the initial password can be retrieved from `/var/jenkins_home/secrets/initialAdminPassword` when Jenkins is running in Docker.

### 2.5 Open Jenkins

Open the following URL in a browser:

```text
http://<droplet-public-ip>:8080
```

Replace `<droplet-public-ip>` with the public IP address of the DigitalOcean Droplet.

The Jenkins **Unlock Jenkins** page should appear.

Paste the initial administrator password and continue with the Jenkins setup wizard.

### 2.6 Complete the Jenkins Setup Wizard

When prompted, select:

**Install suggested plugins**

Jenkins will install the recommended plugins required for a standard Jenkins installation.

After the installation completes, create the first Jenkins administrator account.

Once the setup is complete, Jenkins should display the Jenkins dashboard.

At this point, Jenkins is running successfully as a Docker container on the DigitalOcean Droplet.

## 3. Verify Jenkins

After completing the Jenkins setup wizard, verify that Jenkins is running correctly.

### 3.1 Verify the Jenkins Container

On the Droplet, run:

```bash
docker ps
```

The Jenkins container should appear with a status similar to:

```text
Up ...
```

The port mappings should include:

```text
0.0.0.0:8080->8080/tcp
0.0.0.0:50000->50000/tcp
```

### 3.2 Access the Jenkins Dashboard

Open Jenkins in a browser:

```text
http://<droplet-public-ip>:8080
```

The Jenkins dashboard should load successfully.

### 3.3 Branch 1 Complete

At this point, the DigitalOcean Droplet has:

```text
DigitalOcean Droplet
        │
        ▼
Ubuntu Linux
        │
        ▼
Docker Engine
        │
        ▼
Jenkins Container
        │
        ├── Persistent Jenkins data
        ├── Jenkins Web UI :8080
        └── Jenkins agent port :50000
```

Jenkins is now ready to be used as the CI/CD server for the next branches of this project.

The following branches build on this installation:

* **Branch 2** — Create the Java/Maven CI pipeline
* **Branch 3** — Extract reusable pipeline logic into a Jenkins Shared Library
* **Branch 4** — Add dynamic application versioning
* **Branch 5** — Trigger Jenkins automatically using GitHub Webhooks

