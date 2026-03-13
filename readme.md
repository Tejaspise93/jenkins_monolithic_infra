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

## Jenkins Plugins Installed

| Plugin | Purpose |
|---|---|
| **SonarQube Scanner Plugin** | Integrates SonarQube analysis into pipeline |
| **Nexus Artifact Uploader** | Uploads build artifacts to Nexus repository |
| **Deploy to Container Plugin** | Deploys WAR files to Tomcat |


**How to install any plugin:**
```
Jenkins → Manage Jenkins → Plugins
    ↓
Available plugins tab
    ↓
Search plugin name → Check the box
    ↓
Click: Install
    ↓
Check: Restart Jenkins when installation is complete
    ↓
Verify: Installed plugins tab → search plugin name ✅
```

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
# Default login: admin / admin (change on first login)
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

# Download and extract Tomcat
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.115/bin/apache-tomcat-9.0.115.tar.gz
tar -xvf apache-tomcat-9.0.115.tar.gz
mv apache-tomcat-9.0.115 tomcat
rm apache-tomcat-9.0.115.tar.gz

# Give execute permissions
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
# Manager at: http://<TOOLS-SERVER-IP>:8080/manager → login: deployer / deployer123
```

**Optional — Create shortcuts:**
```bash
echo "alias tomstart='/opt/tomcat/bin/startup.sh'" >> ~/.bashrc
echo "alias tomstop='/opt/tomcat/bin/shutdown.sh'" >> ~/.bashrc
source ~/.bashrc

# Usage:
tomstart    # starts tomcat
tomstop     # stops tomcat
```

> **Note:** The aliases are a convenience for manual use. They call the same `startup.sh` and `shutdown.sh` scripts. However, if the server reboots, Tomcat will not come back up automatically — for that you need the systemd service below.

**Configure Tomcat as a systemd service (auto-start on reboot):**

Step 1 — Verify your Java path first:
```bash
readlink -f $(which java)
# Example output: /usr/lib/jvm/java-17-openjdk-amd64/bin/java
# Use the path up to (not including) /bin/java as JAVA_HOME below
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

Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
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
systemctl enable tomcat      # auto-start on boot
systemctl start tomcat       # start it now
```

Step 4 — Verify:
```bash
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

## Configurations Done

### SonarQube Configuration

**Step 1 — Login to SonarQube:**
```
Browser → http://<TOOLS-SERVER-IP>:9000
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
    ↓
Check: ✅ Enable injection of SonarQube server configuration as build environment variables
    ↓
Click: Add SonarQube
    Name:              SonarQube
    Server URL:        http://<TOOLS-SERVER-IP>:9000
    Server auth token: select sonar-token
    Click: Save
```

**Step 5 — Configure SonarQube Webhook:**

This is required for the Quality Gate stage to work. Without the webhook, SonarQube never notifies Jenkins that analysis is complete and the pipeline will time out waiting.

```
SonarQube → Administration → Configuration → Webhooks
    ↓
Create
    Name:   jenkins
    URL:    http://<JENKINS-SERVER-PUBLIC-IP>:8080/sonarqube-webhook/
    ↓
Save
```

> **Note:** Use the PUBLIC IP of your Jenkins server in the webhook URL. SonarQube calls back to Jenkins over the network so it needs the public IP.

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
Browser → http://<TOOLS-SERVER-IP>:8081
Username: admin
Password: cat /opt/sonatype-work/nexus3/admin.password
```
After first login, Nexus will ask you to set a new password and configure anonymous access. **Disable anonymous access.**

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
    Deployment policy: Disable redeploy    ← keeps release artifacts immutable
    ↓
Click: Create repository
```

> **Note:** Release repositories should be immutable. With redeploy disabled, uploading the same version twice will fail with a 400 error, which is intentional — it forces a version bump in `pom.xml` for every release and prevents accidental overwrites of deployed artifacts.

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
http://<TOOLS-SERVER-IP>:8081 → Browse → maven-releases
→ in → javahome → myweb → <version>
→ myweb-<version>.war        ← artifact
→ myweb-<version>.war.md5    ← checksum (auto-generated)
→ myweb-<version>.war.sha1   ← checksum (auto-generated)
```

---

### Tomcat Credentials in Jenkins
```
Jenkins → Manage Jenkins → Credentials
    ↓
System → Global credentials → Add Credentials
    Kind:        Username with password
    Username:    deployer
    Password:    deployer123
    ID:          tomcat-deployer
    Description: Tomcat Deploy User
    Click: Save
```

---

### GitHub Webhook Configuration

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
    Payload URL:  http://<JENKINS-SERVER-PUBLIC-IP>:8080/github-webhook/
    Content type: application/json
    Secret:       (leave empty)
    Events:       ✅ Just the push event
    Active:       ✅ checked
    ↓
Click: Add webhook
```

> **Note:** Use the PUBLIC IP of your Jenkins server. The trailing slash in `/github-webhook/` is required. Port 8080 must be open in the firewall/security group for the Jenkins server.

**Step 3 — Verify webhook delivery:**
```
GitHub → Repository → Settings → Webhooks
    ↓
Click your webhook → Recent Deliveries
    ↓
Should show ✅ green tick on the ping event
```

**Test it:**
```bash
git add .
git commit -m "testing webhook trigger"
git push origin master
# Jenkins pipeline should trigger automatically within seconds
```

---

### SMTP Configuration (Email Notifications)

**Step 1 — Create Gmail App Password:**
```
Google Account → Security
    ↓
Enable 2-Step Verification if not already on
    ↓
Search: App passwords
    ↓
App name: jenkins → Click: Create
    ↓
Copy the 16 character password — this is your SMTP password
```

**Step 2 — Configure SMTP in Jenkins:**
```
Jenkins → Manage Jenkins → Configure System
    ↓
Scroll to: E-mail Notification
    ↓
SMTP server:        smtp.gmail.com
    ↓
Click: Advanced
    Use SMTP Authentication: ✅ checked
    Username:   your-email@gmail.com
    Password:   <16 character app password>
    Use SSL:    ✅ checked
    SMTP Port:  465
    ↓
Test configuration by sending test e-mail
    Test e-mail recipient: your-email@gmail.com
    Click: Test configuration
    ↓
Should say: Email was successfully sent ✅
    ↓
Save
```

> **Note:** Gmail requires an App Password — not your regular Gmail login password. App Passwords are only available when 2-Step Verification is enabled on your Google account.

---

## Jenkins Credentials Summary

| Credential ID | Kind | Used For |
|---|---|---|
| `sonar-token` | Secret text | SonarQube authentication |
| `nexus3` | Username with password | Nexus artifact upload |
| `tomcat-deployer` | Username with password | Tomcat WAR deployment |

---

## Jenkins Pipeline
```groovy
pipeline {
    agent any

    // ============================================================
    // CONFIGURE YOUR SERVER IPs HERE — change these values only
    // ============================================================
    environment {
        TOOLS_SERVER_IP    = "x.x.x.x"           // SonarQube, Nexus, Tomcat server IP
        GIT_REPO           = "https://github.com/your-repo.git"

        // Built automatically from IP above — do not change below
        SONAR_HOST_URL     = "http://${TOOLS_SERVER_IP}:9000"
        SONAR_TOKEN        = credentials('sonar-token')
        NEXUS_URL          = "http://${TOOLS_SERVER_IP}:8081"
        NEXUS_REPO         = "maven-releases"
        TOMCAT_URL         = "http://${TOOLS_SERVER_IP}:8080"
        email_recipient    = "your-email@gmail.com"
    }
    // ============================================================

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

    // Post block for notifications and workspace cleanup
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
| **Git Checkout** | Pulls code from GitHub using the branch parameter. Reads `pom.xml` to extract project version dynamically. |
| **Build** | Runs `mvn clean package -DskipTests` to compile and produce a `.war` file in `target/`. |
| **Test** | Runs `mvn test` to execute all JUnit tests. Publishes surefire XML reports to Jenkins. Pipeline stops if any test fails. |
| **SonarQube Analysis** | Scans code quality and sends report to SonarQube server. Wrapped in `withSonarQubeEnv` — required for the Quality Gate stage to work. |
| **SonarQube Quality Gate** | Waits for SonarQube to evaluate the gate (max 5 minutes). Aborts the pipeline if the gate fails. Requires the SonarQube webhook to be configured. |
| **Nexus Upload** | Dynamically finds the `.war` and uploads it to the `maven-releases` repository on Nexus. |
| **Deploy to Tomcat** | Downloads WAR from Nexus, undeploys old version, deploys new version to Tomcat. |

---

## Pipeline Flow
```
Developer pushes code to GitHub
            ↓
GitHub sends webhook to Jenkins (automatic)
            ↓
    [Stage 1] Git Checkout
        → pulls branch: params.BRANCH
        → extracts version from pom.xml → stored as env.POM_VERSION
            ↓
    [Stage 2] Build
        → mvn clean package
        → generates target/myweb-x.x.x.war
            ↓
    [Stage 3] Test
        → mvn test
        → JUnit tests must pass
        → surefire reports published to Jenkins
            ↓
    [Stage 4] SonarQube Analysis
        → withSonarQubeEnv wrapper registers the analysis
        → scans code quality
        → report sent to Tools Server:9000
            ↓
    [Stage 5] SonarQube Quality Gate
        → waits for SonarQube webhook callback (max 5 min)
        → pipeline aborts if gate fails
            ↓
    [Stage 6] Nexus Upload
        → finds WAR dynamically
        → uploads to Tools Server:8081/maven-releases
            ↓
    [Stage 7] Deploy to Tomcat
        → downloads WAR from Nexus
        → undeploys old version
        → deploys new version to Tools Server:8080
        → app live at Tools Server:8080/myweb
            ↓
    [Post] Email notification sent, workspace cleaned
```

---

## Verifying Results

**SonarQube Dashboard:**
```
http://<TOOLS-SERVER-IP>:9000 → Projects → myweb
```

**Nexus Artifact:**
```
http://<TOOLS-SERVER-IP>:8081 → Browse → maven-releases
→ in → javahome → myweb → <version> → myweb-<version>.war
```

**Tomcat Manager:**
```
http://<TOOLS-SERVER-IP>:8080/manager
→ Login: deployer / deployer123
→ /myweb should be listed as running ✅
```

**Live Application:**
```
http://<TOOLS-SERVER-IP>:8080/myweb
```

**Webhook Deliveries:**
```
GitHub → Repo → Settings → Webhooks → Recent Deliveries
```

---

## Key Design Decisions

**Branch as parameter** — The pipeline accepts a branch name at runtime so the same pipeline can build `master`, `dev`, or any feature branch without editing the Jenkinsfile.

**Dynamic artifact extraction** — `find target/ -name '*.war'` is used instead of hardcoding the WAR filename so the pipeline works even when the version in `pom.xml` changes.

**Separate Build and Test stages** — Build compiles with `-DskipTests` and tests run in their own stage for clear visibility on which stage failed.

**Credentials never hardcoded** — SonarQube token, Nexus password, and Tomcat credentials are all stored in Jenkins Credential Manager and injected at runtime.

**Pull from Nexus to deploy** — Tomcat pulls the WAR from Nexus instead of using the local build artifact. This validates the full chain — the artifact stored in Nexus is exactly what gets deployed.

**Webhook automation** — GitHub webhook eliminates manual triggering. Every push automatically starts the full pipeline.

**Immutable release artifacts** — The `maven-releases` repository has redeploy disabled. This enforces version discipline — the same version cannot be overwritten once uploaded, ensuring traceability between what is in Nexus and what is deployed.

**`withSonarQubeEnv` wrapper** — Required for `waitForQualityGate` to work. The wrapper registers the analysis with the Jenkins pipeline context. Without it, Jenkins has no record of a SonarQube analysis happening and the Quality Gate stage throws an `IllegalStateException`.

**SonarQube webhook** — Required for the Quality Gate to receive a callback from SonarQube. Without it the pipeline times out waiting for the gate result.

---

## Suggested Improvements

The following improvements were identified but not implemented. They are documented here for future reference.

### Use a dedicated SonarQube service account
The current setup generates the Jenkins token from the `admin` user account. If the admin password is rotated or the account is disabled, the integration breaks. The better approach is to create a dedicated service account in SonarQube (e.g., `jenkins-bot`), grant it only the permissions it needs, and generate the token from that account.

### Disable anonymous access on Nexus
After the initial Nexus setup wizard, anonymous access should be explicitly disabled. Anonymous access allows unauthenticated users to browse and download artifacts from the repository. This is a security risk in any environment accessible beyond localhost.

### Run Nexus under a dedicated service account
Nexus is currently configured to run as `ec2-user`, which is a privileged default cloud user with broad system access. The better practice is to create a dedicated `nexus` OS user with no login shell and no sudo rights, scoped only to the Nexus directories — the same pattern already used for SonarQube with the `sonar` user.

### Configure SonarQube with a production database
SonarQube 9.x ships with an embedded H2 database which is not supported for production use. For anything beyond a local demo, SonarQube should be configured to use PostgreSQL. This is set in `sonarqube-9.1.0.47736/conf/sonar.properties`.

---

## Project Info

- **Application:** myweb
- **Group ID:** in.javahome
- **Packaging:** WAR
- **Repository:** https://github.com/Tejaspise93/my-app-java