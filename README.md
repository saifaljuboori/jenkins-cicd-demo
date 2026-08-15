# Branch 3 — Jenkins Shared Library

## Goal

Extract the reusable Jenkins CI pipeline logic from Branch 2 into a Jenkins Shared Library.

The application pipeline will continue to build the Java/Maven application and Docker image, authenticate with Docker Hub, and push the image. The difference is that the reusable pipeline logic will be maintained in a dedicated Shared Library repository rather than being implemented entirely inside the application's `Jenkinsfile`.

This branch builds on the CI pipeline completed in:

[Branch 2 — Create Jenkins CI Pipeline](https://github.com/saifaljuboori/jenkins-cicd-demo/tree/02-jenkins-ci-pipeline)

## Pipeline Overview

The completed pipeline will implement the following flow:

```text
Java/Maven Application Repository
             │
             │ Jenkinsfile
             ▼
          Jenkins
             │
             │ loads Shared Library
             ▼
    Jenkins Shared Library
             │
             ├── Maven build
             │
             ├── Docker image build
             │
             ├── Docker Hub login
             │
             └── Docker image push
             │
             ▼
      Private Docker Hub Repository
```

## Project Scope

This branch covers:

- Creating a Jenkins Shared Library repository
- Creating the Shared Library directory structure
- Configuring the Shared Library in Jenkins
- Moving reusable CI pipeline logic into the Shared Library
- Updating the application `Jenkinsfile` to use the Shared Library
- Keeping application-specific pipeline configuration in the application repository
- Running the Jenkins pipeline using the Shared Library
- Verifying that the Maven build still works
- Verifying that the Docker image is still built
- Verifying that the Docker image is still pushed to Docker Hub

The goal is to make the CI pipeline logic reusable while preserving the working behavior established in Branch 2.

## 3. Configure the Jenkins Shared Library

The Shared Library used by this branch already exists in a separate GitHub repository:

https://github.com/saifaljuboori/jenkins-shared-library

The repository contains the reusable Jenkins pipeline logic that will be consumed by the application pipeline.

### 3.1 Shared Library Structure

The Shared Library currently contains:

```text
jenkins-shared-library/
├── src/
│   └── com/example/
│       └── Docker.groovy
│
└── vars/
    ├── buildJar.groovy
    ├── buildDockerImage.groovy
    ├── dockerLogin.groovy
    └── dockerPush.groovy
```

The `vars/` directory exposes reusable functions that can be called directly from a Jenkins Pipeline.

The `Docker.groovy` class contains the reusable Docker operations used by the pipeline.

### 3.2 Configure the Library in Jenkins

From the Jenkins dashboard, navigate to:

**Manage Jenkins → System**

Locate:

**Global Pipeline Libraries**

Click:

**Add**

Configure the Shared Library with:

| Setting                    | Value                                                         |
| -------------------------- | ------------------------------------------------------------- |
| **Name**                   | `jenkins-shared-library`                                      |
| **Default version**        | `main`                                                        |
| **Retrieval method**       | Modern SCM                                                    |
| **Source Code Management** | Git                                                           |
| **Project Repository**     | `https://github.com/saifaljuboori/jenkins-shared-library.git` |

The library is configured to use the `main` branch of the Shared Library repository.

The configuration establishes the following relationship:

```text
Jenkins
   │
   ▼
Global Pipeline Libraries
   │
   ▼
jenkins-shared-library
   │
   ├── vars/
   │     ├── buildJar()
   │     ├── buildDockerImage()
   │     ├── dockerLogin()
   │     └── dockerPush()
   │
   └── src/
         └── com/example/Docker.groovy
```

Click **Save** after configuring the library.

At this point, Jenkins can load the Shared Library and make its reusable pipeline functions available to Jenkins Pipelines.

The application `Jenkinsfile` will be updated in the next step to use these shared functions.

## 4. Update the Application Jenkinsfile

Branch 2 contained the CI pipeline implementation directly inside the application `Jenkinsfile`.

In Branch 3, the reusable CI logic is moved into the Jenkins Shared Library.

The application repository keeps the application-specific configuration, while the Shared Library provides the reusable pipeline functions.

### 4.1 Shared Library Functions

The existing Shared Library provides the following reusable functions:

```text
buildJar()
buildDockerImage(imageName)
dockerLogin()
dockerPush(imageName)
```

These functions are implemented in the `jenkins-shared-library` repository.

### 4.2 Update the Application Jenkinsfile

The `java-maven-app` repository is updated to use the Shared Library.

The `Jenkinsfile` now loads the library:

```groovy
@Library('jenkins-shared-library') _
```

and calls the reusable functions provided by the library.

The resulting `Jenkinsfile` is:

```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {

        stage('build jar') {
            steps {
                buildJar()
            }
        }

        stage('build docker image') {
            steps {
                buildDockerImage('saljuboori/demo-app:1.0.0')
            }
        }

        stage('push docker image') {
            steps {
                dockerLogin()
                dockerPush('saljuboori/demo-app:1.0.0')
            }
        }
    }
}
```

### 4.3 What Changed from Branch 2?

In Branch 2, the Jenkinsfile contained the implementation of the CI commands directly.

For example:

```groovy
sh 'mvn clean package'
```

was replaced by:

```groovy
buildJar()
```

The Docker build command:

```groovy
sh 'docker build -t saljuboori/demo-app:1.0.0 .'
```

was replaced by:

```groovy
buildDockerImage('saljuboori/demo-app:1.0.0')
```

Docker authentication and pushing are also delegated to the Shared Library:

```groovy
dockerLogin()
dockerPush('saljuboori/demo-app:1.0.0')
```

The implementation of these functions is no longer maintained inside the application repository.

### 4.4 Application-Specific Configuration

The Docker image name remains in the application `Jenkinsfile`:

```text
saljuboori/demo-app:1.0.0
```

This is intentional.

The Shared Library provides reusable CI logic, while the application repository remains responsible for application-specific values.

This allows the same Shared Library to be reused by other applications without hard-coding their Docker image names into the library.

### 4.5 Commit the Updated Jenkinsfile

After updating the `Jenkinsfile`, commit and push the changes to the application repository:

```bash
git add Jenkinsfile
git commit -m "Use Jenkins shared library"
git push origin master
```

The application pipeline is now configured to use the reusable Jenkins Shared Library.

The next step is to run the Jenkins pipeline and verify that the Shared Library is loaded correctly and that the existing CI behavior from Branch 2 still works.

## 5. Run and Verify the Jenkins Pipeline

After updating the application `Jenkinsfile` to use the Shared Library, the Jenkins pipeline should be executed to verify that the reusable functions work correctly.

The purpose of this verification is to confirm that the CI behavior established in Branch 2 is preserved after moving the reusable logic into the Shared Library.

### 5.1 Run the Pipeline

From the Jenkins dashboard, open the existing application pipeline job:

```text
java-maven-app-pipeline
```

Select:

**Build Now**

Jenkins retrieves the updated `Jenkinsfile` from the `java-maven-app` repository and loads the configured Shared Library.

The pipeline should execute the following stages:

```text
build jar
     │
     ▼
build docker image
     │
     ▼
push docker image
```

### 5.2 Verify the Shared Library Is Loaded

Open the completed build and select:

**Console Output**

The pipeline should successfully load:

```text
jenkins-shared-library
```

The build should then execute the functions provided by the library:

```text
buildJar()
buildDockerImage()
dockerLogin()
dockerPush()
```

A successful build confirms that Jenkins can retrieve and use the Shared Library configured in the previous step.

### 5.3 Verify the Maven Build

The `build jar` stage should execute the reusable `buildJar()` function.

The function runs the Maven build and should produce the application JAR under:

```text
target/
```

The Maven build must complete successfully.

This confirms that moving the Maven build logic into the Shared Library did not change the application build behavior.

### 5.4 Verify the Docker Image Build

The `build docker image` stage should execute:

```groovy
buildDockerImage('saljuboori/demo-app:1.0.0')
```

The Shared Library then performs the Docker build using the supplied image name.

The resulting image should be:

```text
saljuboori/demo-app:1.0.0
```

This confirms that the Docker build logic is being executed from the Shared Library.

### 5.5 Verify Docker Hub Authentication and Push

The final stage executes:

```groovy
dockerLogin()
dockerPush('saljuboori/demo-app:1.0.0')
```

The Shared Library handles Docker Hub authentication and pushes the image to the configured Docker Hub repository.

The Docker Hub credential remains managed by Jenkins and is not stored directly in the application `Jenkinsfile`.

The image should be successfully pushed with the tag:

```text
1.0.0
```

### 5.6 Verify the Docker Hub Repository

Open the private Docker Hub repository:

```text
saljuboori/demo-app
```

Verify that the image:

```text
1.0.0
```

is available.

This confirms that the Shared Library successfully preserved the Docker publishing behavior from Branch 2.

### 5.7 Branch 3 Result

The Branch 3 pipeline now has the following architecture:

```text
java-maven-app
      │
      │ Jenkinsfile
      ▼
Jenkins
      │
      │ loads
      ▼
jenkins-shared-library
      │
      ├── buildJar()
      ├── buildDockerImage()
      ├── dockerLogin()
      └── dockerPush()
      │
      ▼
Docker Hub
```

The application repository contains the application-specific pipeline configuration, while the reusable CI implementation is maintained in the Shared Library.

The CI behavior remains the same as Branch 2:

```text
GitHub
   │
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

The main difference is that the reusable pipeline logic is now centralized in the Shared Library and can be reused by other Jenkins pipelines.

## 6. Branch 3 Complete

Branch 3 successfully moves the reusable Jenkins CI logic established in Branch 2 into a dedicated Jenkins Shared Library.

The Shared Library is maintained separately in:

https://github.com/saifaljuboori/jenkins-shared-library

### 6.1 What Was Achieved

The Branch 3 implementation now:

* Uses a dedicated Jenkins Shared Library repository
* Loads the Shared Library from Jenkins
* Moves reusable Maven build logic into `buildJar()`
* Moves reusable Docker build logic into `buildDockerImage()`
* Moves reusable Docker authentication logic into `dockerLogin()`
* Moves reusable Docker push logic into `dockerPush()`
* Keeps application-specific configuration in the application repository
* Updates the application `Jenkinsfile` to use the Shared Library
* Preserves the working CI behavior established in Branch 2
* Successfully builds the Java/Maven application
* Successfully builds the Docker image
* Successfully pushes the Docker image to Docker Hub

### 6.2 Branch 2 vs Branch 3

The main architectural difference is where the reusable CI implementation is maintained.

**Branch 2:**

```text
java-maven-app
└── Jenkinsfile
    ├── Maven commands
    ├── Docker build commands
    ├── Docker login
    └── Docker push
```

**Branch 3:**

```text
java-maven-app
└── Jenkinsfile
    └── Calls Shared Library functions
            │
            ▼
jenkins-shared-library
├── vars/
│   ├── buildJar.groovy
│   ├── buildDockerImage.groovy
│   ├── dockerLogin.groovy
│   └── dockerPush.groovy
│
└── src/
    └── com/example/
        └── Docker.groovy
```

This separation allows the same CI implementation to be reused by multiple application repositories.

### 6.3 Static Image Tag

As in Branch 2, this branch intentionally uses the static Docker image tag:

```text
saljuboori/demo-app:1.0.0
```

Dynamic application versioning and dynamic Docker image tagging are introduced in Branch 4.

### 6.4 Next Branch

Branch 3 establishes reusable CI pipeline logic through a Jenkins Shared Library.

The next branch will build on this foundation by introducing **dynamic application versioning and Docker image tagging**.

