# Branch 2 — Create Jenkins CI Pipeline

## Goal

Create a Jenkins Continuous Integration (CI) pipeline that automatically retrieves a Java/Maven application from GitHub, builds the application, creates a Docker image, and publishes the image to a private Docker Hub repository.

This branch builds on the Jenkins installation completed in:

[Branch 1 — Install Jenkins](https://github.com/saifaljuboori/jenkins-cicd-demo/tree/01-install-jenkins)

## Pipeline Overview

The completed pipeline will implement the following flow:

```text
GitHub
   │
   │ Checkout source code
   ▼
Jenkins
   │
   ├── Build application with Maven
   │
   ├── Build Docker image
   │
   ├── Login to Docker Hub
   │
   └── Push Docker image
   │
   ▼
Private Docker Hub Repository
```

## Project Scope

This branch covers:

* Preparing the Java/Maven application
* Configuring Maven in Jenkins
* Making Docker available to Jenkins
* Creating Jenkins credentials for GitHub
* Creating a Jenkins Pipeline job
* Connecting Jenkins to the application Git repository
* Creating a Jenkinsfile
* Building the Java application with Maven
* Building a Docker image
* Creating Docker Hub credentials in Jenkins
* Authenticating Jenkins with Docker Hub
* Pushing the Docker image to a private Docker Hub repository
* Verifying the completed CI pipeline

## 1. Application Repository

The CI pipeline in this branch uses the following Java/Maven application as its build source:

[https://github.com/saifaljuboori/java-maven-app](https://github.com/saifaljuboori/java-maven-app?utm_source=chatgpt.com)

The repository is a Java application built with Maven and is used as the application source for the Jenkins pipeline.

### 1.1 Repository Structure

The application repository contains the main Maven project files:

```text
java-maven-app/
├── src/
│   └── main/
│       ├── java/
│       └── resources/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── script.groovy
```

The Jenkins pipeline will check out this repository and execute the Maven and Docker commands against the checked-out application source.

### 1.2 Maven Configuration

The project uses Maven through its `pom.xml`.

The Maven project is configured with:

* **Group ID:** `com.example`
* **Artifact ID:** `java-maven-app`
* **Java version:** 17
* **Spring Boot:** 3.5.5

The Maven build produces the application JAR under the `target/` directory.

For example:

```text
target/
└── java-maven-app-<version>.jar
```

The exact version may change as the project evolves.

### 1.3 Docker Image

The application repository already contains a `Dockerfile`.

The Dockerfile uses Amazon Corretto 17 and copies the Maven-generated JAR from `target/` into the Docker image:

```dockerfile
FROM amazoncorretto:17-alpine-jdk

EXPOSE 8080

COPY ./target/java-maven-app-*.jar /usr/app/

WORKDIR /usr/app

CMD java -jar java-maven-app-*.jar
```

Therefore, the Jenkins pipeline must build the Maven application **before** building the Docker image.

The resulting pipeline order is:

```text
Checkout application
        │
        ▼
Maven build
        │
        ▼
target/java-maven-app-<version>.jar
        │
        ▼
Docker build
        │
        ▼
Docker image
```

### 1.4 Application Repository Prerequisite

No changes to the `java-maven-app` repository are required for this branch.

The application repository already contains the Maven configuration and Dockerfile required by the Jenkins CI pipeline.

The next step is to configure Jenkins with the tools required to build this application.

## 2. Configure Maven in Jenkins

The Java application used in this project is a Maven project. Jenkins therefore needs access to a Maven installation so that the pipeline can build the application.

Jenkins can manage Maven installations through its global tool configuration. This avoids requiring Maven to be manually installed inside the Jenkins container.

### 2.1 Open Jenkins Tool Configuration

From the Jenkins dashboard, navigate to:

**Manage Jenkins → Tools**

Locate the **Maven installations** section.

Click:

**Add Maven**

Configure the Maven installation with a name that can be referenced from the Jenkinsfile.

For this project, use:

```text
my-maven
```

Enable:

**Install automatically**

Then select an available Maven version.

> The Maven installation name is important because the Jenkinsfile will use this name to identify the configured Maven installation.

### 2.2 Maven Installation in Jenkins

After saving the configuration, Jenkins should have a Maven installation similar to:

```text
Manage Jenkins
    └── Tools
         └── Maven installations
              └── Maven
```

The configured Maven installation will be made available to the pipeline when it is declared in the Jenkinsfile.

For example:

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {
        // pipeline stages
    }
}
```

The name in the Jenkinsfile must match the name configured under **Maven installations**.

### 2.3 Why Maven Is Configured in Jenkins

The Maven installation is managed by Jenkins rather than being manually installed inside the Jenkins container.

This provides a predictable build environment for the pipeline and allows the Maven version used by the pipeline to be controlled from Jenkins.

The relationship is:

```text
Jenkins
   │
   ▼
Manage Jenkins → Tools
   │
   ▼
Maven installation
   │
   ▼
Jenkinsfile
   │
   ▼
tools {
    maven 'Maven'
}
   │
   ▼
Maven build
```

### 2.4 Result

At this point, Jenkins has a configured Maven installation that can be used by the Java/Maven CI pipeline.

The next step is to configure Jenkins to access the Docker Engine on the DigitalOcean server so that Jenkins can build Docker images.

## 3. Make Docker Available to Jenkins

Jenkins is running as a Docker container, while Docker Engine is installed directly on the DigitalOcean Droplet.

To allow Jenkins to build Docker images, the Jenkins container needs:

* Access to the host Docker socket
* The Docker CLI installed inside the Jenkins container
* Permission for the Jenkins user to use Docker

This setup uses the **Docker-out-of-Docker (DooD)** approach.

### 3.1 Mount the Docker Socket

The Jenkins container must be started with the host Docker socket mounted:

```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

If Jenkins is being created from scratch, the complete command is:

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

The Docker socket provides the Jenkins container with access to the Docker Engine running on the host.

> If Jenkins is already running without the Docker socket mounted, recreate the container with the socket mount. The `jenkins_home` Docker volume preserves Jenkins data when the container is recreated.

### 3.2 Verify the Docker Socket

On the DigitalOcean Droplet, verify that the Docker socket exists:

```bash
ls -l /var/run/docker.sock
```

Then verify that the socket is visible inside the Jenkins container:

```bash
docker exec jenkins ls -l /var/run/docker.sock
```

The socket should be available at:

```text
/var/run/docker.sock
```

### 3.3 Check for the Docker CLI

Mounting the Docker socket does **not** install the Docker CLI inside the Jenkins container.

Check whether the Docker CLI is available:

```bash
docker exec -it jenkins docker --version
```

If the command returns an error such as:

```text
docker: command not found
```

the Docker CLI must be installed inside the Jenkins container.

### 3.4 Install the Docker CLI

Enter the Jenkins container:

```bash
docker exec -it jenkins bash
```

Update the package index:

```bash
apt-get update
```

Install the Docker package:

```bash
apt-get install -y docker.io
```

Verify the installation:

```bash
docker --version
```

The exact Docker version may differ from the version used in this project because package repositories are continuously updated.

Exit the Jenkins container:

```bash
exit
```

Verify the Docker CLI again:

```bash
docker exec -it jenkins docker --version
```

### 3.5 Verify Docker Access as the Jenkins User

Jenkins normally executes pipeline commands as the `jenkins` user.

First verify that the Jenkins user exists:

```bash
docker exec -it --user jenkins jenkins id
```

Then verify that the Jenkins user can execute the Docker CLI:

```bash
docker exec -it --user jenkins jenkins docker --version
```

Finally, verify that the Jenkins user can communicate with the host Docker Engine:

```bash
docker exec -it --user jenkins jenkins docker ps
```

If `docker ps` succeeds, Jenkins has the required Docker access.

The final setup is:

```text
DigitalOcean Droplet
        │
        ▼
Docker Engine
        │
        │ /var/run/docker.sock
        ▼
Jenkins Container
        │
        ├── Jenkins
        │
        └── Docker CLI
                │
                ▼
        Host Docker Engine
```

Jenkins is now ready to execute Docker commands from the CI pipeline.

The next step is to configure Jenkins credentials for accessing the GitHub application repository.

## 4. Configure GitHub Credentials in Jenkins

Jenkins needs credentials to authenticate with GitHub when connecting to the application repository.

The application repository used by this project is:

```text id="y2qu8m"
https://github.com/saifaljuboori/java-maven-app.git
```

### 4.1 Create a GitHub Personal Access Token

Create a GitHub Personal Access Token (PAT) for Jenkins.

From GitHub, open:

**Settings → Developer settings → Personal access tokens**

Create a token that allows Jenkins to access the required repository.

For this project, the token should provide the minimum permissions required for the Git operations Jenkins will perform.

> Never commit the token to the Git repository or place it directly inside the Jenkinsfile.

GitHub recommends using fine-grained personal access tokens where possible and limiting them to the required repositories and permissions.

### 4.2 Add the Credential to Jenkins

From the Jenkins dashboard, navigate to:

**Manage Jenkins → Credentials**

Open the appropriate credentials store and domain, then select:

**Add Credentials**

Configure the credential as:

| Field           | Value                             |
| --------------- | --------------------------------- |
| **Kind**        | Username with password            |
| **Username**    | Your GitHub username              |
| **Password**    | Your GitHub Personal Access Token |
| **ID**          | `github-credentials`              |
| **Description** | GitHub credentials for Jenkins    |

The important value is the credential ID:

```text id="xujmbr"
github-credentials
```

The Jenkinsfile can later reference this credential by its ID without exposing the actual username or token.

### 4.3 Credential Usage

The Jenkins pipeline can reference the stored credential using Jenkins' credential system.

For example:

```groovy id="v7l4r9"
withCredentials([
    usernamePassword(
        credentialsId: 'github-credentials',
        usernameVariable: 'USERNAME',
        passwordVariable: 'PASSWORD'
    )
]) {
    // authenticated Git operations
}
```

The actual credential values are provided by Jenkins at runtime rather than being stored in the Jenkinsfile.

### 4.4 Verify the Credential

After creating the credential, confirm that it appears in Jenkins under:

```text id="s1g7t6"
Manage Jenkins
    └── Credentials
         └── github-credentials
```

Do not display or copy the token after it has been created.

The GitHub credential is now available for the Jenkins pipeline.

The next step is to create the Jenkins **Pipeline job** and connect it to the `java-maven-app` GitHub repository.

## 5. Create the Jenkins Pipeline Job

The Jenkins Pipeline job connects Jenkins to the GitHub application repository and instructs Jenkins to use the `Jenkinsfile` stored in that repository.

The application repository is:

```text id="w7t4c1"
https://github.com/saifaljuboori/java-maven-app.git
```

### 5.1 Create a New Pipeline Job

From the Jenkins dashboard, select:

**New Item**

Enter a name for the job.

For example:

```text id="1c8j4y"
java-maven-app-pipeline
```

Select:

**Pipeline**

Then click:

**OK**

### 5.2 Configure the Pipeline

In the job configuration page, scroll down to the **Pipeline** section.

Set:

**Definition:**

```text id="0j6f1v"
Pipeline script from SCM
```

Then configure the source control settings.

**SCM:**

```text id="4v0k1p"
Git
```

**Repository URL:**

```text id="y4h7n2"
https://github.com/saifaljuboori/java-maven-app.git
```

### 5.3 Configure GitHub Credentials

Under **Credentials**, select the GitHub credential created in the previous step:

```text id="0y2q6p"
github-credentials
```

Jenkins will use this credential when accessing the Git repository.

### 5.4 Configure the Branch

Specify the branch containing the application source and Jenkinsfile.

For this project:

```text id="2u9d6s"
*/master
```

> If the application's default branch changes, use the branch containing the Jenkinsfile and application source.

### 5.5 Configure the Jenkinsfile Path

Set:

**Script Path:**

```text id="3g6f9w"
Jenkinsfile
```

This tells Jenkins to retrieve the Jenkinsfile from the application repository rather than defining the pipeline directly inside the Jenkins job configuration.

The configuration should now conceptually look like:

```text id="f7v5c3"
Pipeline
│
├── Definition
│     └── Pipeline script from SCM
│
├── SCM
│     └── Git
│
├── Repository URL
│     └── https://github.com/saifaljuboori/java-maven-app.git
│
├── Credentials
│     └── github-credentials
│
├── Branch
│     └── */master
│
└── Script Path
      └── Jenkinsfile
```

### 5.6 Save the Pipeline Job

Click:

**Save**

Jenkins now knows where to find the application source and where to find the pipeline definition.

At this point, the relationship between GitHub and Jenkins is:

```text id="9h3f8n"
GitHub
   │
   │ java-maven-app repository
   │
   ▼
Jenkins Pipeline Job
   │
   │ reads
   ▼
Jenkinsfile
```

The Jenkinsfile will define the actual CI stages executed by Jenkins.

The next step is to create the Jenkinsfile for this branch and define the Maven build and Docker image stages.

## 6. Create the Jenkinsfile

The Jenkins Pipeline job is configured to load its pipeline definition from the application's GitHub repository.

The pipeline definition is stored in:

```text
Jenkinsfile
```

at the root of the `java-maven-app` repository.

The Jenkinsfile defines the CI process used by this branch.

### 6.1 Pipeline Stages

The Branch 2 pipeline contains the following stages:

```text
GitHub Repository
        │
        ▼
   Checkout Source
        │
        ▼
    Maven Build
        │
        ▼
   Docker Build
        │
        ▼
 Docker Hub Login
        │
        ▼
   Docker Push
```

The pipeline does not deploy the application in this branch.

Deployment is outside the scope of this CI pipeline.

### 6.2 Jenkinsfile

Create a file named:

```text
Jenkinsfile
```

in the root of the `java-maven-app` repository.

Use the following pipeline:

```groovy
pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {

        stage('build jar') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('build docker image') {
            steps {
                sh 'docker build -t saljuboori/demo-app:1.0.0 .'
            }
        }

        stage('push docker image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )
                ]) {
                    sh 'docker login -u $USERNAME -p $PASSWORD'
                    sh 'docker push saljuboori/demo-app:1.0.0'
                }
            }
        }
    }
}
```

> The Docker Hub repository name and Jenkins credential ID must match the values configured in your Jenkins environment. The image tag `1.0.0` is intentionally static in this branch. Dynamic versioning is introduced in Branch 4.

### 6.3 Maven Build

The first stage builds the Java application:

```groovy
stage('build jar') {
    steps {
        sh 'mvn clean package'
    }
}
```

The command:

```bash
mvn clean package
```

performs the Maven build and creates the application JAR.

The generated JAR is created under:

```text
target/
```

The Dockerfile uses this generated JAR when building the Docker image.

### 6.4 Build the Docker Image

The next stage builds the Docker image:

```groovy
stage('build docker image') {
    steps {
        sh 'docker build -t saljuboori/demo-app:1.0.0 .'
    }
}
```

The `.` tells Docker to use the current Jenkins workspace as the build context.

The resulting image is:

```text
saljuboori/demo-app:1.0.0
```

### 6.5 Push the Docker Image

Before Jenkins can push the image, it must authenticate with Docker Hub.

The Docker Hub credentials are stored securely in Jenkins.

The credential used by this pipeline is:

```text
dockerhub-credentials
```

The pipeline retrieves the credentials using:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'dockerhub-credentials',
        usernameVariable: 'USERNAME',
        passwordVariable: 'PASSWORD'
    )
]) {
    // Docker commands
}
```

The credentials are then used to authenticate with Docker Hub:

```bash
docker login -u $USERNAME -p $PASSWORD
```

After authentication, Jenkins pushes the image:

```bash
docker push saljuboori/demo-app:1.0.0
```

The image is then available in the configured Docker Hub repository.

> Never place a Docker Hub password or access token directly in the Jenkinsfile.

### 6.6 Complete CI Flow

The Branch 2 CI pipeline is:

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── mvn clean package
   │        │
   │        ▼
   │     JAR artifact
   │
   ├── docker build
   │        │
   │        ▼
   │   Docker image
   │
   ├── docker login
   │
   └── docker push
            │
            ▼
        Docker Hub
```

### 6.7 Commit the Jenkinsfile

After creating the Jenkinsfile, commit and push it to the application's repository:

```bash
git add Jenkinsfile
git commit -m "Add Jenkins CI pipeline"
git push origin master
```

The Jenkins Pipeline job can now retrieve the Jenkinsfile from GitHub.

The next step is to configure the Docker Hub credentials in Jenkins and verify that Jenkins can successfully build and push the Docker image.

## 7. Configure Docker Hub Credentials

The Jenkins pipeline needs permission to push the Docker image to Docker Hub.

This project uses a private Docker Hub repository.

The image produced by the pipeline is:

```text
saljuboori/demo-app:1.0.0
```

### 7.1 Create the Docker Hub Repository

Sign in to Docker Hub and create a new repository.

Navigate to:

**My Hub → Repositories → Create repository**

Configure the repository with:

| Setting             | Value        |
| ------------------- | ------------ |
| **Namespace**       | `saljuboori` |
| **Repository name** | `demo-app`   |
| **Visibility**      | Private      |

The resulting repository is:

```text
saljuboori/demo-app
```

Docker Hub supports both public and private repositories. A private repository is only accessible to the account and users with permission to access it.

### 7.2 Create a Docker Hub Personal Access Token

Jenkins should not use the Docker Hub account password for CI authentication.

Create a Docker Hub Personal Access Token (PAT).

In Docker Hub, open the account security/access-token settings and create a new token.

Use a descriptive name such as:

```text
jenkins-ci
```

The token needs permission to push images to the repository.

For a personal Docker Hub repository, a token with **Read & Write** repository access is sufficient for this CI pipeline.

Copy the token when it is generated.

> Docker Hub does not allow you to retrieve the token after it has been generated. Treat the token like a password and keep it secret.

### 7.3 Create the Docker Hub Credential in Jenkins

From the Jenkins dashboard, navigate to:

**Manage Jenkins → Credentials**

Select the appropriate credentials store and domain, then select:

**Add Credentials**

Configure the credential as:

| Field           | Value                                 |
| --------------- | ------------------------------------- |
| **Kind**        | Username with password                |
| **Username**    | Your Docker Hub username              |
| **Password**    | Your Docker Hub Personal Access Token |
| **ID**          | `dockerhub-credentials`               |
| **Description** | Docker Hub credentials for Jenkins    |

The important value is:

```text
dockerhub-credentials
```

The Jenkinsfile references this credential ID:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'dockerhub-credentials',
        usernameVariable: 'USERNAME',
        passwordVariable: 'PASSWORD'
    )
]) {
    // Docker commands
}
```

Jenkins injects the credentials only when the pipeline executes the block.

The actual Docker Hub username and token are therefore not stored directly in the Jenkinsfile.

### 7.4 Verify the Credential

Confirm that the credential appears in Jenkins:

```text
Manage Jenkins
    └── Credentials
         └── dockerhub-credentials
```

Do not expose the token in the Jenkinsfile, Git repository, terminal output, or screenshots.

At this point, Jenkins has the credentials required to authenticate with Docker Hub and push the Docker image.

The next step is to run the Jenkins pipeline and verify that it can:

1. Clone the application repository
2. Build the Java application with Maven
3. Build the Docker image
4. Authenticate with Docker Hub
5. Push the image to the private Docker Hub repository

## 8. Run and Verify the CI Pipeline

The Jenkins job is now configured with:

* GitHub repository access
* Maven configuration
* Docker CLI access
* Docker socket access
* Docker Hub credentials
* A Jenkinsfile defining the CI pipeline

The next step is to run the pipeline and verify each stage.

### 8.1 Run the Pipeline

From the Jenkins dashboard, open:

```text
java-maven-app-pipeline
```

Select:

**Build Now**

Jenkins will retrieve the Jenkinsfile from the GitHub repository and execute the pipeline.

The build should progress through the configured stages:

```text
build jar
     │
     ▼
build docker image
     │
     ▼
push docker image
```

### 8.2 Verify the Maven Build

Open the completed build and select:

**Console Output**

Look for the Maven build command:

```bash
mvn clean package
```

The Maven build should complete successfully and generate the application JAR under:

```text
target/
```

### 8.3 Verify the Docker Image Build

The next stage should execute:

```bash
docker build -t saljuboori/demo-app:1.0.0 .
```

A successful Docker build should produce an image tagged:

```text
saljuboori/demo-app:1.0.0
```

The Jenkins console output should show that the Docker image was successfully built.

### 8.4 Verify the Docker Hub Authentication

The pipeline then authenticates with Docker Hub using the Jenkins credential:

```text
dockerhub-credentials
```

The credentials themselves should not appear in the console output.

The authentication step should complete successfully.

### 8.5 Verify the Docker Image Push

The final stage pushes the image:

```bash
docker push saljuboori/demo-app:1.0.0
```

A successful push confirms that Jenkins can authenticate with Docker Hub and publish the image.

### 8.6 Verify the Image in Docker Hub

Open the private Docker Hub repository:

```text
saljuboori/demo-app
```

The repository should contain the newly pushed image with the tag:

```text
1.0.0
```

This confirms that the complete CI pipeline successfully published the Docker image.

### 8.7 Verify the Pipeline Result

Return to the Jenkins job page.

The completed build should show:

```text
SUCCESS
```

The complete CI flow is now working:

```text
GitHub
   │
   │ source code + Jenkinsfile
   ▼
Jenkins
   │
   ├── Maven build
   │
   ├── Docker image build
   │
   ├── Docker Hub authentication
   │
   └── Docker image push
             │
             ▼
        Docker Hub
```

### 8.8 Branch 2 Complete

At this point, Branch 2 demonstrates a complete Jenkins CI pipeline that:

* Retrieves the Java/Maven application from GitHub
* Builds the application with Maven
* Builds a Docker image
* Authenticates with a private Docker Hub repository
* Pushes the Docker image to Docker Hub

The image produced by this branch is:

```text
saljuboori/demo-app:1.0.0
```

Branch 2 intentionally uses a **static Docker image tag**.

Dynamic application versioning and Docker image tagging are introduced in **Branch 4**.

The next branch will extract the repeated Jenkins pipeline logic into a **Jenkins Shared Library**, making the pipeline easier to maintain and reuse across projects.

