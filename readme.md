# CI/CD Pipeline — Jenkins + Maven + SonarQube + Nexus + Tomcat




## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Jenkins | Latest LTS | CI/CD orchestration |
| Maven | 3.x | Build, test, and package the Java project |
| SonarQube | 9.1.0 | Static code quality analysis |
| Nexus Repository Manager | 3.x | Artifact storage |
| Tomcat | 9.0.115 | Web application server for deployment |
| Java | 17 | Runtime for Jenkins, SonarQube, Tomcat |
| Java | 8 | Runtime for Nexus |
| Git / GitHub | - | Source code management |
| GitHub Webhook | - | Auto-triggers Jenkins on push |

---

## Jenkins Plugins Required

| Plugin | Purpose |
|---|---|
| SonarQube Scanner Plugin | Integrates SonarQube analysis into the pipeline |
| Nexus Artifact Uploader | Uploads build artifacts to the Nexus repository |
| Deploy to Container Plugin | Deploys WAR files to Tomcat |

**How to install a plugin:**
Jenkins → Manage Jenkins → Plugins → Available plugins → search name → check box → Install → check "Restart Jenkins when installation is complete" → verify under Installed plugins tab

---

## Configurations

### SonarQube

**Step 1 — Generate a Jenkins Authentication Token**

SonarQube → top-right corner → My Account → Security tab → Generate Tokens
- Name: `jenkins-token`
- Type: User Token
- Click Generate — copy the token immediately, it is shown only once

**Step 2 — Add the Token to Jenkins**

Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
- Kind: Secret text
- Secret: paste the token
- ID: `sonar-token`
- Save

**Step 3 — Configure the SonarQube Server in Jenkins**

Jenkins → Manage Jenkins → Configure System → SonarQube servers → Add SonarQube
- Name: `SonarQube`
- Server URL: `http://<TOOLS-SERVER-IP>:9000`
- Server auth token: `sonar-token`
- Check: Enable injection of SonarQube server configuration as build environment variables
- Save

**Step 4 — Configure the SonarQube Webhook**

> **Note:** Without this webhook, SonarQube never notifies Jenkins that analysis is complete. The Quality Gate stage will time out.

SonarQube → Administration → Configuration → Webhooks → Create
- Name: `jenkins`
- URL: `http://<JENKINS-SERVER-PUBLIC-IP>:8080/sonarqube-webhook/`
- Save

> **Note:** Use the public IP of the Jenkins server. The trailing slash in `/sonarqube-webhook/` is required.

**What SonarQube checks:** bugs, vulnerabilities, code smells, test coverage, and code duplications.

---

### Nexus

**Step 1 — Login to Nexus**

Open `http://<TOOLS-SERVER-IP>:8081`. Username: `admin`. Get the initial password from the Tools Server:

```bash
cat /opt/sonatype-work/nexus3/admin.password
```

After first login, set a new password and disable anonymous access when prompted.

**Step 2 — Create the maven-releases Repository**

Left sidebar → Server Administration (gear icon) → Repository → Repositories → Create repository → `maven2 (hosted)`
- Name: `maven-releases`
- Version policy: Release
- Layout policy: Strict
- Deployment policy: Disable redeploy
- Create repository

> **Note:** Disable redeploy keeps release artifacts immutable. Uploading the same version twice fails with a 400 error — this is intentional and forces a version bump in `pom.xml` for every release.

**Step 3 — Add Nexus Credentials to Jenkins**

Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
- Kind: Username with password
- Username: `admin`
- Password: your Nexus admin password
- ID: `nexus3`
- Save

**Where artifacts are stored:**
```
http://<TOOLS-SERVER-IP>:8081 → Browse → maven-releases
→ in → javahome → myweb → <version>
→ myweb-<version>.war
→ myweb-<version>.war.md5   (auto-generated)
→ myweb-<version>.war.sha1  (auto-generated)
```

---

### Tomcat Credentials in Jenkins

Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
- Kind: Username with password
- Username: `deployer`
- Password: `deployer123`
- ID: `tomcat-deployer`
- Save

---

### GitHub Webhook

**Step 1 — Enable Webhook Trigger in Jenkins**

Jenkins → your pipeline job → Configure → Build Triggers → check "GitHub hook trigger for GITScm polling" → Save

**Step 2 — Add the Webhook in GitHub**

GitHub → Repository → Settings → Webhooks → Add webhook
- Payload URL: `http://<JENKINS-SERVER-PUBLIC-IP>:8080/github-webhook/`
- Content type: `application/json`
- Events: Just the push event
- Active: checked
- Add webhook

> **Note:** The trailing slash in `/github-webhook/` is required. Port 8080 must be open in the Jenkins Server security group.

**Step 3 — Verify Webhook Delivery**

GitHub → Repository → Settings → Webhooks → click your webhook → Recent Deliveries — should show a green tick on the ping event.

---

### SMTP Configuration (Email Notifications)

**Step 1 — Create a Gmail App Password**

Google Account → Security → enable 2-Step Verification if not already on → search "App passwords" → App name: `jenkins` → Create → copy the 16-character password

> **Note:** Gmail requires an App Password, not your regular Gmail password. App Passwords are only available when 2-Step Verification is enabled.

**Step 2 — Configure SMTP in Jenkins**

Jenkins → Manage Jenkins → Configure System → E-mail Notification
- SMTP server: `smtp.gmail.com`
- Click Advanced
  - Use SMTP Authentication: checked
  - Username: `your-email@gmail.com`
  - Password: the 16-character app password
  - Use SSL: checked
  - SMTP Port: `465`
- Test configuration by sending a test email to yourself
- Save

---

## Jenkins Credentials Summary

| Credential ID | Kind | Used For |
|---|---|---|
| `sonar-token` | Secret text | SonarQube authentication |
| `nexus3` | Username with password | Nexus artifact upload |
| `tomcat-deployer` | Username with password | Tomcat WAR deployment |

---

## Jenkins Pipeline

Update `TOOLS_SERVER_IP`, `GIT_REPO`, and `email_recipient` before running.

```groovy
pipeline {
    agent any

    environment {
        TOOLS_SERVER_IP    = "x.x.x.x"
        GIT_REPO           = "https://github.com/your-repo.git"

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
                        // Download WAR from Nexus
                        sh """
                            curl --fail \
                                -u \$NEXUS_USER:\$NEXUS_PASS \
                                -o myweb.war \
                                ${nexusWarUrl}
                        """

                        // Undeploy old version
                        sh """
                            curl -s -o /dev/null \
                                -u \$TOMCAT_USER:\$TOMCAT_PASS \
                                "${TOMCAT_URL}/manager/text/undeploy?path=/myweb" || true
                        """

                        // Deploy new WAR to Tomcat
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

**Branch as parameter** — The pipeline accepts a branch name at runtime so the same Jenkinsfile can build `master`, `dev`, or any feature branch without editing it.

**Dynamic artifact extraction** — `find target/ -name '*.war'` is used instead of hardcoding the WAR filename so the pipeline works even when the version in `pom.xml` changes.

**Separate Build and Test stages** — Build compiles with `-DskipTests` and tests run in their own stage for clear visibility on which stage failed.

**Credentials never hardcoded** — SonarQube token, Nexus password, and Tomcat credentials are all stored in Jenkins Credential Manager and injected at runtime.

**Pull from Nexus to deploy** — Tomcat pulls the WAR from Nexus instead of using the local build artifact. This validates the full chain — the artifact stored in Nexus is exactly what gets deployed.

**Immutable release artifacts** — The `maven-releases` repository has redeploy disabled. The same version cannot be overwritten once uploaded, ensuring traceability between what is in Nexus and what is deployed.

**`withSonarQubeEnv` wrapper** — Required for `waitForQualityGate` to work. Without it, Jenkins has no record of a SonarQube analysis happening and the Quality Gate stage throws an `IllegalStateException`.

**SonarQube webhook** — Required for the Quality Gate to receive a callback from SonarQube. Without it the pipeline times out waiting for the gate result.

---

## Suggested Improvements

**Use a dedicated SonarQube service account** — The current setup generates the Jenkins token from the `admin` account. If the admin password is rotated or the account is disabled, the integration breaks. Create a dedicated service account (e.g. `jenkins-bot`), grant it only the permissions it needs, and generate the token from that account.

**Disable anonymous access on Nexus** — Anonymous access should be explicitly disabled after the initial setup. It allows unauthenticated users to browse and download artifacts, which is a security risk in any externally accessible environment.

**Run Nexus under a dedicated service account** — Nexus currently runs as `ec2-user`, which has broad system access. Create a dedicated `nexus` OS user with no login shell and no sudo rights, scoped only to the Nexus directories — the same pattern already used for SonarQube with the `sonar` user.

**Configure SonarQube with a production database** — SonarQube 9.x ships with an embedded H2 database that is not supported for production use. For anything beyond a local demo, configure SonarQube to use PostgreSQL via `sonarqube-9.1.0.47736/conf/sonar.properties`.

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
login: deployer / deployer123
/myweb should be listed as running
```

**Live Application:**
```
http://<TOOLS-SERVER-IP>:8080/myweb
```

**Webhook Deliveries:**
```
GitHub → Repository → Settings → Webhooks → Recent Deliveries
```

---

## Project Info

- **Application:** myweb
- **Group ID:** in.javahome
- **Packaging:** WAR
- **Repository:** https://github.com/Tejaspise93/my-app-java