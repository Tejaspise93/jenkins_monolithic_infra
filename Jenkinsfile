pipeline {
    agent any

    // ============================================================
    // CONFIGURE YOUR SERVER IPs HERE — change these values only
    // ============================================================
    environment {
        TOOLS_SERVER_IP    = "x.x.x.x"       // SonarQube, Nexus, Tomcat server IP

        // Built automatically from IP above — do not change below
        SONAR_HOST_URL     = "http://${TOOLS_SERVER_IP}:9000"
        SONAR_TOKEN        = credentials('sonar-token')
        NEXUS_URL          = "http://${TOOLS_SERVER_IP}:8081"
        NEXUS_REPO         = "maven-releases"
        TOMCAT_URL         = "http://${TOOLS_SERVER_IP}:8080"
        email_recipient     = "you_email@gmail.com"
    }
    // ============================================================

    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to build')
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: "${params.BRANCH}",
                    url: 'https://github.com/Tejaspise93/my-app-java.git'
                script {
                    // Use env.POM_VERSION so the value persists across all stages
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
                sh """
                    mvn sonar:sonar \
                        -Dsonar.projectKey=myweb \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.login=\$SONAR_TOKEN
                """
            }
        }

        // Enforce Quality Gate — pipeline fails if Sonar gate does not pass
        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
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
            cleanWs()   // clean workspace after every run to avoid stale artifacts
        }
    }
}