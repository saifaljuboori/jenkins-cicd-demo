# Branch 4 — Dynamic Versioning

## Goal

Introduce automatic application versioning into the Jenkins CI/CD pipeline.

Branch 3 established a Jenkins Shared Library containing reusable Maven and Docker pipeline logic.

Branch 4 builds on that foundation by adding reusable version-management functions to the Shared Library.

The pipeline will now:

1. Automatically increment the Maven application version
2. Read the updated Maven version
3. Use that version as the Docker image tag
4. Build and push the versioned Docker image
5. Commit the updated `pom.xml` back to GitHub
6. Add `[skip ci]` to the Jenkins-generated commit to prevent that commit from triggering another CI build when supported by the CI/webhook integration

The result is a pipeline where the application version and Docker image version are generated automatically instead of being hard-coded in the `Jenkinsfile`.

This branch builds on:

[Branch 3 — Jenkins Shared Library](https://github.com/saifaljuboori/jenkins-cicd-demo/tree/03-jenkins-shared-library)

---

## Pipeline Flow

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── Increment Maven version
   │
   ├── Build JAR
   │
   ├── Read application version
   │
   ├── Build Docker image
   │      │
   │      └── Docker tag = application version
   │
   ├── Login to Docker Hub
   │
   ├── Push Docker image
   │
   ├── Deploy application
   │
   └── Commit updated pom.xml
          │
          └── [skip ci]
                 │
                 ▼
               GitHub
```

---

## Versioning Example

If the application starts with:

```xml
<version>1.4.12</version>
```

Jenkins automatically increments the incremental version to:

```xml
<version>1.4.13</version>
```

The same version is then used for the Docker image:

```text
saljuboori/demo-app:1.4.13
```

After the pipeline completes, Jenkins commits the updated `pom.xml` back to GitHub using:

```text
chore: increment application version [skip ci]
```

The `[skip ci]` marker is intended to prevent the Jenkins-generated version commit from starting another CI build.

The resulting relationship is:

```text
Maven version
    │
    ▼
  1.4.13
    │
    ├── Docker image
    │      └── saljuboori/demo-app:1.4.13
    │
    └── Git commit
           └── chore: increment application version [skip ci]
```

---

## What Changed from Branch 3?

Branch 3 used a static Docker image version.

For example:

```text
saljuboori/demo-app:1.0.0
```

Branch 4 removes the hard-coded version from the Docker image name.

Instead, Jenkins obtains the Maven project version dynamically:

```groovy
def version = getVersion()

env.IMAGE_NAME = "saljuboori/demo-app:${version}"
```

The Shared Library now also provides three additional reusable functions:

```text
incrementVersion()
getVersion()
commitVersion()
```

These functions are responsible for version management and committing the updated version back to GitHub.

---

## 1. Extend the Jenkins Shared Library

The existing Jenkins Shared Library from Branch 3 contains the reusable Maven and Docker functions.

Branch 4 extends it with three additional functions:

```text
jenkins-shared-library/
├── src/
│   └── com/example/
│       └── Docker.groovy
│
└── vars/
    ├── buildDockerImage.groovy
    ├── buildJar.groovy
    ├── commitVersion.groovy
    ├── dockerLogin.groovy
    ├── dockerPush.groovy
    ├── getVersion.groovy
    └── incrementVersion.groovy
```

The three new functions are:

```text
incrementVersion()
getVersion()
commitVersion()
```

Because they are located in the `vars/` directory, Jenkins exposes them as reusable Pipeline steps.

---

## 2. Increment the Maven Application Version

The `incrementVersion()` function is implemented in:

```text
vars/incrementVersion.groovy
```

Its implementation is:

```groovy
def call() {
    echo "Incrementing app version..."

    sh "mvn build-helper:parse-version versions:set '-DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion}' versions:commit"
}
```

The function uses Maven to parse the current project version and increment the incremental version.

For example:

```text
1.4.12
  │
  │ increment incremental version
  ▼
1.4.13
```

The major and minor versions remain unchanged while the incremental version is increased automatically.

---

## 3. Read the Current Maven Version

The `getVersion()` function is implemented in:

```text
vars/getVersion.groovy
```

Its implementation is:

```groovy
def call() {
    return sh(
        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
        returnStdout: true
    ).trim()
}
```

This executes Maven's `help:evaluate` goal and retrieves:

```text
project.version
```

from `pom.xml`.

For example, after `incrementVersion()` changes the project version to:

```xml
<version>1.4.13</version>
```

`getVersion()` returns:

```text
1.4.13
```

The Jenkins Pipeline can then use this value to construct the Docker image name dynamically.

---

## 4. Use the Maven Version for the Docker Image

The application `Jenkinsfile` retrieves the current version using:

```groovy
def version = getVersion()
```

The Docker image name is then constructed dynamically:

```groovy
env.IMAGE_NAME = "saljuboori/demo-app:${version}"
```

For example, if Maven reports:

```text
1.4.13
```

Jenkins creates:

```text
saljuboori/demo-app:1.4.13
```

The existing Shared Library Docker functions are then called using this dynamically generated image name:

```groovy
buildDockerImage(env.IMAGE_NAME)
dockerLogin()
dockerPush(env.IMAGE_NAME)
```

This creates a direct relationship between the Maven application version and the Docker image tag.

```text
pom.xml
   │
   │ project.version
   ▼
1.4.13
   │
   ▼
Docker image
   │
   ▼
saljuboori/demo-app:1.4.13
```

---

## 5. Commit the Updated Version Back to GitHub

After the application version has been incremented and the pipeline has completed its build and deployment stages, the updated `pom.xml` is committed back to the application repository.

This logic is implemented in:

```text
vars/commitVersion.groovy
```

The implementation is:

```groovy
def call() {
    echo "Committing updated application version..."

    withCredentials([
        usernamePassword(
            credentialsId: 'github-credentials',
            usernameVariable: 'USERNAME',
            passwordVariable: 'PASSWORD'
        )
    ]) {
        sh '''
            git config user.name "Jenkins"
            git config user.email "jenkins@example.com"

            git add pom.xml
            git commit -m "chore: increment application version [skip ci]"

            git push https://${USERNAME}:${PASSWORD}@github.com/saifaljuboori/java-maven-app.git HEAD:master
        '''
    }
}
```

The function performs these operations:

```text
1. Load GitHub credentials from Jenkins
             │
             ▼
2. Configure Git user as Jenkins
             │
             ▼
3. Stage pom.xml
             │
             ▼
4. Create the version commit
             │
             ▼
5. Push the commit to GitHub
```

The GitHub credential used by the Shared Library is:

```text
github-credentials
```

The credential is retrieved through Jenkins:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'github-credentials',
        usernameVariable: 'USERNAME',
        passwordVariable: 'PASSWORD'
    )
])
```

The credentials are therefore not stored directly in the application `Jenkinsfile`.

---

## 6. Prevent the Jenkins Commit Loop

An important problem appears when Jenkins itself creates a commit.

Without protection, the pipeline could create this cycle:

```text
GitHub
   │
   ▼
Jenkins
   │
   ├── increment version
   ├── build
   ├── push Docker image
   └── commit pom.xml
          │
          ▼
       GitHub
          │
          ▼
       Jenkins
          │
          ├── increment version
          ├── build
          ├── commit
          └── ...
```

Branch 4 adds:

```text
[skip ci]
```

to the automatically generated commit message:

```text
chore: increment application version [skip ci]
```

The intended flow becomes:

```text
Developer commit
      │
      ▼
   Jenkins
      │
      ├── Increment version
      ├── Build JAR
      ├── Build Docker image
      ├── Push Docker image
      ├── Deploy
      │
      └── Commit version [skip ci]
                  │
                  ▼
                GitHub
                  │
                  X
             No CI trigger
```

This is the commit-loop prevention mechanism introduced in Branch 4.

---

## 7. Update the Application Jenkinsfile

The application repository continues to load the Shared Library:

```groovy
@Library('jenkins-shared-library') _
```

The Branch 4 `Jenkinsfile` is:

```groovy
@Library('jenkins-shared-library') _

pipeline {
    agent any

    tools {
        maven 'my-maven'
    }

    stages {

        stage("increment version") {
            steps {
                script {
                    incrementVersion()
                }
            }
        }

        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }

        stage("build and push docker image") {
            steps {
                script {
                    def version = getVersion()

                    env.IMAGE_NAME = "saljuboori/demo-app:${version}"

                    echo "Docker image: ${env.IMAGE_NAME}"

                    buildDockerImage(env.IMAGE_NAME)
                    dockerLogin()
                    dockerPush(env.IMAGE_NAME)
                }
            }
        }

        stage("deploy") {
            steps {
                script {
                    echo "deploying the application..."
                }
            }
        }

        stage("commit version") {
            steps {
                script {
                    commitVersion()
                }
            }
        }
    }
}
```

The application `Jenkinsfile` remains relatively small because the reusable implementation is maintained in the Shared Library.

---

## 8. Branch 4 Pipeline Stages

The Branch 4 Pipeline consists of five stages:

```text
increment version
        │
        ▼
build jar
        │
        ▼
build and push docker image
        │
        ▼
deploy
        │
        ▼
commit version
```

### Stage 1 — Increment Version

```groovy
incrementVersion()
```

Updates the Maven project version.

### Stage 2 — Build JAR

```groovy
buildJar()
```

Builds the Java/Maven application.

### Stage 3 — Build and Push Docker Image

```groovy
def version = getVersion()

env.IMAGE_NAME = "saljuboori/demo-app:${version}"

buildDockerImage(env.IMAGE_NAME)
dockerLogin()
dockerPush(env.IMAGE_NAME)
```

The Docker image is tagged using the Maven application version.

### Stage 4 — Deploy

```groovy
echo "deploying the application..."
```

Deployment remains a placeholder in this branch.

### Stage 5 — Commit Version

```groovy
commitVersion()
```

Commits the updated `pom.xml` and pushes it to GitHub with `[skip ci]`.

---

## 9. Run and Verify the Pipeline

After the Shared Library has been updated and the application `Jenkinsfile` is configured, run the Jenkins Pipeline.

Open the application Pipeline in Jenkins and select:

**Build Now**

The expected stages are:

```text
increment version
        │
        ▼
build jar
        │
        ▼
build and push docker image
        │
        ▼
deploy
        │
        ▼
commit version
```

---

## 10. Verify Version Increment

Before the Pipeline runs, check the current version in:

```text
pom.xml
```

For example:

```xml
<version>1.4.12</version>
```

After `incrementVersion()` executes, the version should become:

```xml
<version>1.4.13</version>
```

The Jenkins console should show:

```text
Incrementing app version...
```

---

## 11. Verify the Docker Image Tag

The `build and push docker image` stage should display the dynamically generated image name.

For example:

```text
Docker image: saljuboori/demo-app:1.4.13
```

The Docker image should therefore be built and pushed using the same version as the Maven application.

This confirms that the Docker tag is no longer hard-coded.

---

## 12. Verify the Git Commit

The final stage executes:

```groovy
commitVersion()
```

The resulting Git commit should have the message:

```text
chore: increment application version [skip ci]
```

The commit should contain the updated:

```text
pom.xml
```

The commit should then be pushed back to the `master` branch of the application repository.

---

## 13. Verify Commit-Loop Prevention

After Jenkins pushes the version commit to GitHub, verify that the `[skip ci]` marker prevents another CI execution from being triggered by that generated commit.

The expected behavior is:

```text
Developer commit
      │
      ▼
Jenkins Pipeline runs
      │
      ▼
Jenkins increments version
      │
      ▼
Jenkins commits [skip ci]
      │
      ▼
GitHub
      │
      X
No second CI Pipeline
```

This prevents an automatic version update from creating an endless Jenkins → GitHub → Jenkins cycle.

---

## 14. Branch 3 vs Branch 4

### Branch 3

Branch 3 used a static Docker image version:

```text
saljuboori/demo-app:1.0.0
```

The Docker image tag was manually specified in the `Jenkinsfile`.

### Branch 4

Branch 4 introduces automatic versioning:

```text
Current Maven version
        │
        ▼
incrementVersion()
        │
        ▼
New Maven version
        │
        ▼
getVersion()
        │
        ▼
Docker image tag
        │
        ▼
commitVersion()
        │
        ▼
GitHub [skip ci]
```

The Docker image version and Maven application version are now synchronized automatically.

---

## 15. Shared Library Structure After Branch 4

The Shared Library now contains:

```text
jenkins-shared-library/
├── src/
│   └── com/example/
│       └── Docker.groovy
│
└── vars/
    ├── buildDockerImage.groovy
    ├── buildJar.groovy
    ├── commitVersion.groovy
    ├── dockerLogin.groovy
    ├── dockerPush.groovy
    ├── getVersion.groovy
    └── incrementVersion.groovy
```

The responsibilities are:

| Function             | Responsibility                           |
| -------------------- | ---------------------------------------- |
| `buildJar()`         | Builds the Maven application             |
| `incrementVersion()` | Increments the Maven incremental version |
| `getVersion()`       | Reads the current Maven project version  |
| `buildDockerImage()` | Builds the Docker image                  |
| `dockerLogin()`      | Authenticates with Docker Hub            |
| `dockerPush()`       | Pushes the Docker image                  |
| `commitVersion()`    | Commits and pushes the updated `pom.xml` |

---

## 16. Important Design Note

The current `commitVersion()` implementation contains the application repository URL directly:

```text
https://github.com/saifaljuboori/java-maven-app.git
```

and pushes specifically to:

```text
HEAD:master
```

This is sufficient for the current learning project because the Shared Library is being used by this application repository.

However, this means `commitVersion()` is not completely application-agnostic.

A more reusable Shared Library implementation would accept the repository and branch as configuration rather than hard-coding them.

For this branch, the hard-coded repository is retained to match the implementation being demonstrated.

---

## 17. Branch 4 Complete

Branch 4 introduces dynamic application versioning into the Jenkins CI/CD Pipeline.

The Pipeline now:

* Automatically increments the Maven application version
* Reads the updated Maven version
* Uses the Maven version as the Docker image tag
* Builds the versioned Docker image
* Pushes the versioned Docker image to Docker Hub
* Commits the updated `pom.xml` to GitHub
* Uses the Jenkins-managed `github-credentials` credential for the Git push
* Adds `[skip ci]` to the automated version commit
* Prevents the Jenkins-generated commit from creating a CI loop
* Keeps the versioning logic reusable inside the Jenkins Shared Library

The resulting architecture is:

```text
                    GitHub
                       │
                       │ Developer commit
                       ▼
                    Jenkins
                       │
                       ▼
             Jenkins Shared Library
                       │
          ┌────────────┼─────────────┐
          │            │             │
          ▼            ▼             ▼
 incrementVersion   buildJar      getVersion
          │                          │
          │                          ▼
          │                    Docker image
          │                    version/tag
          │                          │
          │                          ▼
          │                     Docker Hub
          │
          ▼
       pom.xml
          │
          ▼
    commitVersion()
          │
          ▼
       GitHub
          │
          │ [skip ci]
          X
     No CI loop
```

Branch 4 establishes automated version management while keeping the implementation reusable through the Jenkins Shared Library.

The next branch will introduce **GitHub Webhooks**, allowing GitHub repository events to automatically notify Jenkins and trigger the CI Pipeline.

