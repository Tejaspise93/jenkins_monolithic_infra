# CI/CD Pipeline — Jenkins + Maven + SonarQube + Nexus + Tomcat

## Overview

This project sets up a full CI/CD pipeline using Jenkins to automate building, testing, code quality analysis, artifact storage, and deployment of a Java web application.

---

## Infrastructure

| Server | Purpose |
|---|---|
| Jenkins Server | Jenkins, Maven |
| Tools Server | SonarQube (port 9000), Nexus (port 8081), Tomcat (port 8080) |

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Jenkins | Latest LTS | CI/CD orchestration |
| Maven | 3.x | Build, test, and package Java project |
| SonarQube | 9.1.0 | Static code quality analysis |
| Nexus Repository Manager | 3.x | Artifact storage |
| Tomcat | 9.0.115 | Web application server for deployment |
| Java | 17 | Runtime for Jenkins, SonarQube, Tomcat |
| Java | 8 | Runtime for Nexus |
| Git | - | Source code management |
| GitHub Webhook | - | Auto-trigger Jenkins on code push |

---

## Jenkins Plugins Required

| Plugin | Purpose |
|---|---|
| SonarQube Scanner Plugin | Integrates SonarQube analysis into pipeline |
| Nexus Artifact Uploader | Uploads build artifacts to Nexus repository |
| Deploy to Container Plugin | Deploys WAR files to Tomcat |

**How to install a plugin:**

Jenkins -> Manage Jenkins -> Plugins -> Available plugins -> search plugin name -> check the box -> Install -> check "Restart Jenkins when installation is complete" -> verify under Installed plugins tab

---

## Server Setup

### SonarQube (Tools Server)

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
# Access at: http://<TOOLS-SERVER-IP>:9000
# Default login: admin / admin (you will be forced to change this on first login)
```

---

### Nexus (Tools Server)

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
# Access at: http://<TOOLS-SERVER-IP>:8081
# Get initial password:
cat /opt/sonatype-work/nexus3/admin.password
```

---

### Tomcat (Tools Server)

```bash
yum install -y java-17
cd /opt

wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.115/bin/apache-tomcat-9.0.115.tar.gz
tar -xvf apache-tomcat-9.0.115.tar.gz
mv apache-tomcat-9.0.115 tomcat
rm apache-tomcat-9.0.115.tar.gz

chmod +x /opt/tomcat/bin/*.sh
```

**Configure Tomcat Manager user:**

```bash
vi /opt/tomcat/conf/tomcat-users.xml
```

Add before `</tomcat-users>`:

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="deployer"
      password="deployer123"
      roles="manager-gui,manager-script"/>
```

**Allow remote access to Manager:**

```bash
vi /opt/tomcat/webapps/manager/META-INF/context.xml
```

Comment out the Valve block:

```xml
<!--
<Valve className="org.apache.catalina.valves.RemoteCIDRValve"
       allow="127.0.0.0/8,::1/128" />
-->
```

**Start Tomcat:**

```bash
/opt/tomcat/bin/startup.sh

# Verify
ps aux | grep tomcat
# Access at: http://<TOOLS-SERVER-IP>:8080
# Manager at: http://<TOOLS-SERVER-IP>:8080/manager -> login: deployer / deployer123
```

**Optional — Create shortcuts:**

```bash
echo "alias tomstart='/opt/tomcat/bin/startup.sh'" >> ~/.bashrc
echo "alias tomstop='/opt/tomcat/bin/shutdown.sh'" >> ~/.bashrc
source ~/.bashrc
```

> **Note:** These aliases are a convenience for manual use. If the server reboots, Tomcat will not come back up automatically — use the systemd service below for that.

**Configure Tomcat as a systemd service (auto-start on reboot):**

Step 1 — Verify your Java path:
```bash
readlink -f $(which java)
# Example output: /usr/lib/jvm/java-17-openjdk-amd64/bin/java
# Strip /bin/java from the end — that is your JAVA_HOME
# e.g. /usr/lib/jvm/java-17-openjdk-amd64
```

Step 2 — Create the service file:
```bash
vi /etc/systemd/system/tomcat.service
```

Paste this content:
```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking
User=root
Group=root

Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"  # replace with your actual path from the readlink command above
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Step 3 — Reload systemd and enable the service:
```bash
systemctl daemon-reload
systemctl enable tomcat
systemctl start tomcat
systemctl status tomcat
```

**Useful systemd commands:**
```bash
systemctl start tomcat
systemctl stop tomcat
systemctl restart tomcat
systemctl status tomcat
```

---

### Maven (Jenkins Server)

```bash
sudo apt update
sudo apt install -y maven
mvn --version
```

---

## Configurations

### SonarQube

**Step 1 — Generate an authentication token:**

SonarQube -> top right corner -> My Account -> Security tab -> Generate Tokens -> set name to `jenkins-token`, type: User Token -> Generate

Copy the token immediately — it is shown only once.

**Step 2 — Add token to Jenkins:**

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: Secret text
- Secret: paste the token
- ID: sonar-token
- Save

**Step 3 — Configure SonarQube server in Jenkins:**

Jenkins -> Manage Jenkins -> Configure System -> SonarQube servers -> Add SonarQube
- Name: SonarQube
- Server URL: http://<TOOLS-SERVER-IP>:9000
- Server auth token: sonar-token
- Check: Enable injection of SonarQube server configuration as build environment variables
- Save

**Step 4 — Configure SonarQube webhook:**

This is required for the Quality Gate stage to work. Without it, SonarQube never notifies Jenkins that analysis is complete and the pipeline will time out.

SonarQube -> Administration -> Configuration -> Webhooks -> Create
- Name: jenkins
- URL: http://<JENKINS-SERVER-PUBLIC-IP>:8080/sonarqube-webhook/
- Save

> **Note:** Use the public IP of the Jenkins server in the webhook URL. SonarQube calls back to Jenkins over the network.

**What SonarQube checks:** bugs, vulnerabilities, code smells, test coverage, and code duplications.

---

### Nexus

**Step 1 — Login to Nexus:**

Open `http://<TOOLS-SERVER-IP>:8081`. Username: `admin`. Get the initial password by running `cat /opt/sonatype-work/nexus3/admin.password`. After first login, set a new password and disable anonymous access when prompted.

**Step 2 — Create the maven-releases repository:**

Left sidebar -> Server Administration (gear icon) -> Repository -> Repositories -> Create repository -> select `maven2 (hosted)`
- Name: maven-releases
- Version policy: Release
- Layout policy: Strict
- Deployment policy: Disable redeploy
- Create repository

> **Note:** Disable redeploy keeps release artifacts immutable. Uploading the same version twice will fail with a 400 error — this is intentional and forces a version bump in `pom.xml` for every release.

**Step 3 — Add Nexus credentials to Jenkins:**

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: Username with password
- Username: admin
- Password: your Nexus admin password
- ID: nexus3
- Save

**Where artifacts are stored:**
```
http://<TOOLS-SERVER-IP>:8081 -> Browse -> maven-releases
-> in -> javahome -> myweb -> <version>
-> myweb-<version>.war
-> myweb-<version>.war.md5   (auto-generated)
-> myweb-<version>.war.sha1  (auto-generated)
```

---

### Tomcat Credentials in Jenkins

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: Username with password
- Username: deployer
- Password: deployer123
- ID: tomcat-deployer
- Save

---

### GitHub Webhook

**Step 1 — Enable webhook trigger in Jenkins:**

Jenkins -> your pipeline job -> Configure -> Build Triggers -> check "GitHub hook trigger for GITScm polling" -> Save

**Step 2 — Add webhook in GitHub:**

GitHub -> Repository -> Settings -> Webhooks -> Add webhook
- Payload URL: http://<JENKINS-SERVER-PUBLIC-IP>:8080/github-webhook/
- Content type: application/json
- Events: Just the push event
- Active: checked
- Add webhook

> **Note:** The trailing slash in `/github-webhook/` is required. Port 8080 must be open in the Jenkins Server security group.

**Step 3 — Verify webhook delivery:**

GitHub -> Repository -> Settings -> Webhooks -> click your webhook -> Recent Deliveries — should show a green tick on the ping event.

---

### SMTP Configuration (Email Notifications)

**Step 1 — Create a Gmail App Password:**

Google Account -> Security -> enable 2-Step Verification if not already on -> search "App passwords" -> App name: jenkins -> Create -> copy the 16-character password

**Step 2 — Configure SMTP in Jenkins:**

Jenkins -> Manage Jenkins -> Configure System -> E-mail Notification
- SMTP server: smtp.gmail.com
- Click Advanced
  - Use SMTP Authentication: checked
  - Username: your-email@gmail.com
  - Password: the 16-character app password
  - Use SSL: checked
  - SMTP Port: 465
- Test configuration by sending a test email to yourself
- Save

> **Note:** Gmail requires an App Password, not your regular Gmail password. App Passwords are only available when 2-Step Verification is enabled.

---

## Jenkins Credentials Summary

| Credential ID | Kind | Used For |
|---|---|---|
| sonar-token | Secret text | SonarQube authentication |
| nexus3 | Username with password | Nexus artifact upload |
| tomcat-deployer | Username with password | Tomcat WAR deployment |

---

## Jenkins Pipeline

```groovy
pipeline {
    agent any

    environment {
        TOOLS_SERVER_IP    = "x.x.x.x"           // tool server IP
        GIT_REPO           = "https://github.com/your-repo.git"

        // Built automatically from IP above — do not change below
        SONAR_HOST_URL     = "http://${TOOLS_SERVER_IP}:9000"
        SONAR_TOKEN        = credentials('sonar-token')
        NEXUS_URL          = "http://${TOOLS_SERVER_IP}:8081"
        NEXUS_REPO         = "maven-releases"
        TOMCAT_URL         = "http://${TOOLS_SERVER_IP}:8080"
        email_recipient    = "your-email@gmail.com"
    }

    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to build')
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: "${params.BRANCH}",
                    url: "${GIT_REPO}"
                script {
                    // Used env.POM_VERSION so the value persists across all stages
                    env.POM_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "--- Detected Version: ${env.POM_VERSION} ---"
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
            post {
                always {
                    // Publish JUnit results so test trends appear in Jenkins UI
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=myweb \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=\$SONAR_TOKEN
                    """
                }
            }
        }

        // Enforce Quality Gate — pipeline fails if Sonar gate does not pass
        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
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
                    nexusUrl: "${TOOLS_SERVER_IP}:8081",
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    repository: "${NEXUS_REPO}",
                    version: "${env.POM_VERSION}"
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                script {
                    def nexusWarUrl = "${NEXUS_URL}/repository/${NEXUS_REPO}/in/javahome/myweb/${env.POM_VERSION}/myweb-${env.POM_VERSION}.war"
                    echo "--- Pulling WAR from Nexus: ${nexusWarUrl} ---"

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'nexus3',
                            usernameVariable: 'NEXUS_USER',
                            passwordVariable: 'NEXUS_PASS'
                        ),
                        usernamePassword(
                            credentialsId: 'tomcat-deployer',
                            usernameVariable: 'TOMCAT_USER',
                            passwordVariable: 'TOMCAT_PASS'
                        )
                    ]) {
                        // Step 1: Download WAR from Nexus
                        sh """
                            curl --fail \
                                -u \$NEXUS_USER:\$NEXUS_PASS \
                                -o myweb.war \
                                ${nexusWarUrl}
                        """

                        // Step 2: Undeploy old version
                        sh """
                            curl -s -o /dev/null \
                                -u \$TOMCAT_USER:\$TOMCAT_PASS \
                                "${TOMCAT_URL}/manager/text/undeploy?path=/myweb" || true
                        """

                        // Step 3: Deploy new WAR to Tomcat
                        sh """
                            curl -v --fail \
                                -u \$TOMCAT_USER:\$TOMCAT_PASS \
                                -T myweb.war \
                                "${TOMCAT_URL}/manager/text/deploy?path=/myweb&update=true"
                        """
                    }
                    echo "--- App deployed at: ${TOMCAT_URL}/myweb ---"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded — ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            echo "App live at: ${TOMCAT_URL}/myweb"
            mail to: "${email_recipient}",
                 subject: "Build Succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """
                     Build succeeded.

                     Job:     ${env.JOB_NAME}
                     Build:   #${env.BUILD_NUMBER}
                     Branch:  ${params.BRANCH}
                     App URL: ${TOMCAT_URL}/myweb

                     Console: ${env.BUILD_URL}
                 """
        }
        failure {
            echo "Pipeline failed — ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            echo "Check console output: ${env.BUILD_URL}"
            mail to: "${email_recipient}",
                 subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: """
                     Build failed.

                     Job:    ${env.JOB_NAME}
                     Build:  #${env.BUILD_NUMBER}
                     Branch: ${params.BRANCH}

                     Check the console output for details:
                     ${env.BUILD_URL}
                 """
        }
        always {
            cleanWs()
        }
    }
}
```

---

## Pipeline Stage Breakdown

| Stage | What It Does |
|---|---|
| Git Checkout | Pulls code from GitHub using the branch parameter. Reads `pom.xml` to extract the project version and stores it as `env.POM_VERSION`. |
| Build | Runs `mvn clean package -DskipTests` to compile and produce a `.war` file in `target/`. |
| Test | Runs `mvn test`. Publishes surefire XML reports to Jenkins. Pipeline stops if any test fails. |
| SonarQube Analysis | Scans code quality and sends the report to SonarQube. Must use the `withSonarQubeEnv` wrapper — without it the Quality Gate stage cannot find the analysis. |
| SonarQube Quality Gate | Waits up to 5 minutes for SonarQube to evaluate the gate. Aborts the pipeline if the gate fails. Requires the SonarQube webhook to be configured. |
| Nexus Upload | Dynamically finds the `.war` and uploads it to the `maven-releases` repository on Nexus. |
| Deploy to Tomcat | Downloads the WAR from Nexus, undeploys the old version, and deploys the new version to Tomcat. |

---

## Pipeline Flow

```
Developer pushes code to GitHub
    |
    v
GitHub sends webhook to Jenkins
    |
    v
[Stage 1] Git Checkout
    pulls branch, extracts version from pom.xml -> stored as env.POM_VERSION
    |
    v
[Stage 2] Build
    mvn clean package -> target/myweb-x.x.x.war
    |
    v
[Stage 3] Test
    mvn test -> JUnit results published to Jenkins
    |
    v
[Stage 4] SonarQube Analysis
    scans code quality -> report sent to Tools Server:9000
    |
    v
[Stage 5] SonarQube Quality Gate
    waits for webhook callback (max 5 min) -> aborts if gate fails
    |
    v
[Stage 6] Nexus Upload
    finds WAR dynamically -> uploads to Tools Server:8081/maven-releases
    |
    v
[Stage 7] Deploy to Tomcat
    downloads WAR from Nexus -> undeploys old version -> deploys new version
    |
    v
App live at Tools Server:8080/myweb
    |
    v
Email notification sent, workspace cleaned
```

---

## Design Decisions

**Branch as parameter** — The pipeline accepts a branch name at runtime so the same pipeline can build `master`, `dev`, or any feature branch without editing the Jenkinsfile.

**Dynamic artifact extraction** — `find target/ -name '*.war'` is used instead of hardcoding the WAR filename so the pipeline works even when the version in `pom.xml` changes.

**Separate Build and Test stages** — Build compiles with `-DskipTests` and tests run in their own stage for clear visibility on which stage failed.

**Credentials never hardcoded** — SonarQube token, Nexus password, and Tomcat credentials are all stored in Jenkins Credential Manager and injected at runtime.

**Pull from Nexus to deploy** — Tomcat pulls the WAR from Nexus instead of using the local build artifact. This validates the full chain — the artifact stored in Nexus is exactly what gets deployed.

**Immutable release artifacts** — The `maven-releases` repository has redeploy disabled. The same version cannot be overwritten once uploaded, ensuring traceability between what is in Nexus and what is deployed.

**`withSonarQubeEnv` wrapper** — Required for `waitForQualityGate` to work. Without it, Jenkins has no record of a SonarQube analysis happening and the Quality Gate stage throws an `IllegalStateException`.

**SonarQube webhook** — Required for the Quality Gate to receive a callback from SonarQube. Without it the pipeline times out waiting for the gate result.

---

## Suggested Improvements

**Use a dedicated SonarQube service account** — The current setup generates the Jenkins token from the `admin` account. If the admin password is rotated or the account is disabled, the integration breaks. Create a dedicated service account (e.g., `jenkins-bot`), grant it only the permissions it needs, and generate the token from that account.

**Disable anonymous access on Nexus** — Anonymous access should be explicitly disabled after the initial setup. It allows unauthenticated users to browse and download artifacts, which is a security risk in any externally accessible environment.

**Run Nexus under a dedicated service account** — Nexus currently runs as `ec2-user`, which has broad system access. Create a dedicated `nexus` OS user with no login shell and no sudo rights, scoped only to the Nexus directories — the same pattern already used for SonarQube with the `sonar` user.

**Configure SonarQube with a production database** — SonarQube 9.x ships with an embedded H2 database that is not supported for production use. For anything beyond a local demo, configure SonarQube to use PostgreSQL via `sonarqube-9.1.0.47736/conf/sonar.properties`.

---

## Verifying Results

**SonarQube Dashboard:**
```
http://<TOOLS-SERVER-IP>:9000 -> Projects -> myweb
```

**Nexus Artifact:**
```
http://<TOOLS-SERVER-IP>:8081 -> Browse -> maven-releases
-> in -> javahome -> myweb -> <version> -> myweb-<version>.war
```

**Tomcat Manager:**
```
http://<TOOLS-SERVER-IP>:8080/manager
login: deployer / deployer123
/myweb should be listed as running
```

**Live Application:**
```
http://<TOOLS-SERVER-IP>:8080/myweb
```

**Webhook Deliveries:**
```
GitHub -> Repository -> Settings -> Webhooks -> Recent Deliveries
```

---

## AWS Security Group — Required Open Ports

| Server | Port | Purpose |
|---|---|---|
| Jenkins Server | 8080 | Jenkins UI and GitHub webhook receiver |
| Tools Server | 9000 | SonarQube |
| Tools Server | 8081 | Nexus |
| Tools Server | 8080 | Tomcat / Live application |

---

## Project Info

- **Application:** myweb
- **Group ID:** in.javahome
- **Packaging:** WAR
- **Repository:** https://github.com/Tejaspise93/my-app-java