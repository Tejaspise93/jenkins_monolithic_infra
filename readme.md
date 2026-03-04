# CI/CD Pipeline — Jenkins + Maven + SonarQube + Nexus

## Overview

This project sets up a full CI/CD pipeline using Jenkins to automate building, testing, code quality analysis, artifact storage, and automated triggering via GitHub webhook for a Java web application.

---

## Infrastructure

| Server | Instance Type | What Runs On It |
|---|---|---|
| Jenkins Server | t3.small | Jenkins, Maven |
| Tools Server | m7i-flex.large | SonarQube (port 9000), Nexus (port 8081) |

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Jenkins | Latest LTS | CI/CD orchestration |
| Maven | 3.x | Build, test, and package Java project |
| SonarQube | 9.1.0 | Static code quality analysis |
| Nexus Repository Manager | 3.x | Artifact storage |
| Java | 17 | Runtime for Jenkins, SonarQube |
| Java | 8 | Runtime for Nexus |
| Git | - | Source code management |
| GitHub Webhook | - | Auto-trigger Jenkins on code push |

---

## Jenkins Plugins Installed

| Plugin | Purpose |
|---|---|
| **Git Plugin** | Enables Git checkout from GitHub in pipeline |
| **Maven Integration Plugin** | Allows Jenkins to run Maven commands |
| **SonarQube Scanner Plugin** | Integrates SonarQube analysis into pipeline |
| **Nexus Artifact Uploader** | Uploads build artifacts to Nexus repository |
| **Credentials Plugin** | Securely stores and injects credentials |
| **Pipeline Plugin** | Enables declarative pipeline (Jenkinsfile) support |
| **Workflow Aggregator** | Full pipeline suite (stages, steps, scripted blocks) |
| **GitHub Plugin** | Enables GitHub webhook trigger for automatic builds |

---

## Server Setup

### SonarQube (m7i-flex.large)
```bash
yum install -y java-17
cd /opt/
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.1.0.47736.zip
unzip sonarqube-9.1.0.47736.zip
useradd sonar
chown -R sonar:sonar sonarqube-9.1.0.47736
cd sonarqube-9.1.0.47736/bin/linux-x86-64

# Edit sonar.sh — uncomment RUN_AS_USER and set it to sonar
vi sonar.sh

sh sonar.sh start
sh sonar.sh status
# Access at: http://<m7i-IP>:9000
# Default login: admin / admin (change on first login)
```

---

### Nexus (m7i-flex.large)
```bash
sudo yum install java-1.8.0
cd /opt
sudo wget https://download.sonatype.com/nexus/3/nexus-unix-x86-64-3.79.0-09.tar.gz
tar -xvf nexus-unix-x86-64-3.79.0-09.tar.gz
mv nexus-3.79.0-09 nexus3
chown -R ec2-user:ec2-user nexus3 sonatype-work
cd nexus3/bin

# Edit nexus file — set run_as_user=ec2-user
vi nexus

ln -s /opt/nexus3/bin/nexus /etc/init.d/nexus
chkconfig --add nexus
chkconfig nexus on
sudo service nexus start
# Access at: http://<m7i-IP>:8081
# Get initial password:
cat /opt/sonatype-work/nexus3/admin.password
```

---

### Maven (t3.small Jenkins Server)
```bash
sudo apt update
sudo apt install -y maven
mvn --version
```

---

## Configurations Done

### SonarQube Configuration

**Step 1 — Login to SonarQube:**
```
Browser → http://<m7i-IP>:9000
Username: admin
Password: admin  (you will be forced to change this on first login)
```

**Step 2 — Generate Authentication Token:**
```
Top right corner → My Account
    ↓
Security tab
    ↓
Generate Tokens section
    Name: jenkins-token
    Type: User Token
    Click: Generate
    ↓
COPY THE TOKEN IMMEDIATELY — it is shown only once
```

**Step 3 — Add Token to Jenkins:**
```
Jenkins → Manage Jenkins → Credentials
    ↓
System → Global credentials → Add Credentials
    Kind:        Secret text
    Secret:      <paste the token copied from SonarQube>
    ID:          sonar-token
    Description: SonarQube Auth Token
    Click: Save
```

**Step 4 — Configure SonarQube Server in Jenkins:**
```
Install SonarQube Scanner plugin → restart Jenkins
    ↓
Jenkins → Manage Jenkins → Configure System
    ↓
Scroll to: SonarQube servers
    Click: Add SonarQube
    Name:             SonarQube
    Server URL:       http://<m7i-IP>:9000
    Server auth token: select sonar-token
    Click: Save
```

**What SonarQube checks:**
- Bugs — code that will likely cause runtime errors
- Vulnerabilities — security issues in your code
- Code Smells — maintainability issues
- Test Coverage — how much code is covered by tests
- Duplications — copy-pasted code blocks

---

### Nexus Configuration

**Step 1 — Login to Nexus:**
```
Browser → http://<m7i-IP>:8081
Username: admin
Password: cat /opt/sonatype-work/nexus3/admin.password
```
After first login, Nexus will ask you to set a new password and configure anonymous access.

**Step 2 — Create maven-releases Repository:**
```
Left sidebar → Server Administration (gear icon)
    ↓
Repository → Repositories → Create repository
    ↓
Select recipe: maven2 (hosted)
    ↓
    Name:              maven-releases
    Version policy:    Release
    Layout policy:     Strict
    Deployment policy: Allow redeploy    ← important
    ↓
Click: Create repository
```

**Step 3 — Add Nexus Credentials to Jenkins:**
```
Jenkins → Manage Jenkins → Credentials
    ↓
System → Global credentials → Add Credentials
    Kind:        Username with password
    Username:    admin
    Password:    <your nexus admin password>
    ID:          nexus3
    Description: Nexus Admin Credentials
    Click: Save
```

**Where artifacts are stored:**
```
http://<m7i-IP>:8081 → Browse → maven-releases
→ in → javahome → myweb → <version>
→ myweb-<version>.war        ← artifact
→ myweb-<version>.war.md5    ← checksum (auto-generated)
→ myweb-<version>.war.sha1   ← checksum (auto-generated)
```

---

### GitHub Webhook Configuration

Webhook ensures Jenkins automatically triggers a build on every code push — no manual intervention needed.

**Step 1 — Enable webhook trigger in Jenkins:**
```
Jenkins → Your Pipeline Job → Configure
    ↓
Build Triggers section
    ↓
Check: ✅ GitHub hook trigger for GITScm polling
    ↓
Save
```

**Step 2 — Add webhook in GitHub:**
```
GitHub → Repository → Settings → Webhooks → Add webhook
    ↓
    Payload URL:  http://<t3-small-PUBLIC-IP>:8080/github-webhook/
    Content type: application/json
    Secret:       (leave empty)
    Events:       ✅ Just the push event
    Active:       ✅ checked
    ↓
Click: Add webhook
```

> **Note:** Use the PUBLIC IP of your t3.small — GitHub needs to reach Jenkins from the internet. The trailing slash in `/github-webhook/` is required.

**Step 3 — Verify webhook delivery:**
```
GitHub → Repository → Settings → Webhooks
    ↓
Click your webhook → Recent Deliveries
    ↓
Should show ✅ green tick on the ping event
```

**Step 4 — Allow port 8080 in AWS Security Group:**
```
AWS Console → EC2 → t3.small instance
    ↓
Security tab → Security groups → Inbound rules
    ↓
Confirm port 8080 is open to 0.0.0.0/0
```

**Test it:**
```bash
git add .
git commit -m "testing webhook trigger"
git push origin master
# Jenkins pipeline should trigger automatically within seconds
```

---

## Jenkins Pipeline
```groovy
pipeline {
    agent any
    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to build')
    }

    environment {
        POM_VERSION    = ""
        SONAR_HOST_URL = "http://<m7i-IP>:9000"
        SONAR_TOKEN    = credentials('sonar-token')
        NEXUS_URL      = "http://<m7i-IP>:8081"
        NEXUS_REPO     = "maven-releases"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: "${params.BRANCH}",
                    url: 'https://github.com/Tejaspise93/my-app-java.git'
                script {
                    POM_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "--- Detected Version: ${POM_VERSION} ---"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh """
                    mvn sonar:sonar \
                        -Dsonar.projectKey=myweb \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=\$SONAR_TOKEN
                """
            }
        }

        stage('Nexus Upload') {
            steps {
                script {
                    def warFile = sh(
                        script: "find target/ -name '*.war' | head -1",
                        returnStdout: true
                    ).trim()
                    echo "--- Uploading artifact: ${warFile} ---"

                    nexusArtifactUploader artifacts: [[
                        artifactId: 'myweb',
                        classifier: '',
                        file: warFile,
                        type: 'war'
                    ]],
                    credentialsId: 'nexus3',
                    groupId: 'in.javahome',
                    nexusUrl: '<m7i-IP>:8081',
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    repository: "${NEXUS_REPO}",
                    version: "${POM_VERSION}"
                }
            }
        }
    }
}
```

---

## Pipeline Stage Breakdown

| Stage | What It Does |
|---|---|
| **Git Checkout** | Pulls code from GitHub using the branch specified in the parameter. Also reads `pom.xml` to extract the project version dynamically. |
| **Build** | Runs `mvn clean package -DskipTests` to compile the code and produce a `.war` file in the `target/` folder. |
| **Test** | Runs `mvn test` to execute all JUnit tests. Pipeline stops here if any test fails. |
| **SonarQube Analysis** | Sends code to SonarQube for static analysis — checks for bugs, vulnerabilities, and code smells. Results viewable at `http://<m7i-IP>:9000`. |
| **Nexus Upload** | Dynamically finds the `.war` file in `target/` and uploads it to the `maven-releases` repository on Nexus. |

---

## Pipeline Flow
```
Developer pushes code to GitHub
            ↓
GitHub sends webhook to Jenkins (automatic)
            ↓
    [Stage 1] Git Checkout
        → pulls branch: params.BRANCH
        → extracts version from pom.xml
            ↓
    [Stage 2] Build
        → mvn clean package
        → generates target/myweb-x.x.x.war
            ↓
    [Stage 3] Test
        → mvn test
        → JUnit tests must pass
            ↓
    [Stage 4] SonarQube Analysis
        → scans code quality
        → report sent to m7i:9000
            ↓
    [Stage 5] Nexus Upload
        → finds WAR dynamically
        → uploads to m7i:8081/maven-releases
```

---

## Verifying Results

**SonarQube Dashboard:**
```
http://<m7i-IP>:9000 → Projects → myweb
```
Check: Bugs, Vulnerabilities, Code Smells, Quality Gate status

**Nexus Artifact:**
```
http://<m7i-IP>:8081 → Browse → maven-releases
→ in → javahome → myweb → <version> → myweb-<version>.war
```

**Webhook Deliveries:**
```
GitHub → Repo → Settings → Webhooks → Recent Deliveries
```

---

## Key Design Decisions

**Branch as parameter** — The pipeline accepts a branch name at runtime so the same pipeline can build `master`, `dev`, or any feature branch without editing the Jenkinsfile.

**Dynamic artifact extraction** — Instead of hardcoding the WAR filename, `find target/ -name '*.war'` is used so the pipeline continues to work even when the version number in `pom.xml` changes.

**Separate Build and Test stages** — Build compiles with `-DskipTests`, and tests run in their own stage. This gives clear visibility in the Jenkins UI on exactly which stage failed.

**Credentials never hardcoded** — Both the SonarQube token and Nexus password are stored in Jenkins Credential Manager and injected at runtime using `credentials()`.

**Webhook automation** — GitHub webhook eliminates manual triggering. Every push to the repository automatically starts the full pipeline.

---

## Project Info

- **Application:** myweb
- **Group ID:** in.javahome
- **Packaging:** WAR
- **Repository:** https://github.com/Tejaspise93/my-app-java