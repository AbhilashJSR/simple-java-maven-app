pipeline {
    agent any

    environment {
        JAVA_HOME = "/usr/lib/jvm/java-21-openjdk-amd64"
        PATH = "${JAVA_HOME}/bin:/opt/maven/bin:$PATH"
        GIT_REPO_URL = 'https://github.com/AbhilashJSR/simple-java-maven-app.git'
        SONAR_URL = 'http://13.232.131.185:30900/'
        SONAR_CRED_ID = 'sonar-token-id'
        NEXUS_URL = 'http://13.232.131.185:30801/#browse/browse:maven-releases'
        NEXUS_DOCKER_REPO = '13.232.131.185:30002/'
        NEXUS_CRED_ID = 'nexus-creds'
        NEXUS_DOCKER_CRED_ID = 'nexus-docker-creds'
        MAX_BUILDS_TO_KEEP = 5
        MVN_CMD = '/opt/maven/bin/mvn'
    }
        tools {
             maven 'Maven 3'
    }
    stages {
        stage('Checkout') {
            steps {
                git url: "${GIT_REPO_URL}", branch: 'master'
            }
        }

        stage('Create Sonar Project') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        sh """
                            curl -s -o /dev/null -w "%{http_code}" -X POST \\
                            -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                            "${SONAR_URL}/api/projects/create?project=${projectName}&name=${projectName}" || true
                        """
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def projectName = "${env.JOB_NAME}-${env.BUILD_NUMBER}".replace('/', '-')
                    withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('MySonar') {
                            sh """
                                mvn clean verify sonar:sonar \\
                                -Dsonar.projectKey=${projectName} \\
                                -Dsonar.host.url=${SONAR_URL} \\
                                -Dsonar.login=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }

        stage('Build and Tag Artifact') {
            steps {
                script {
                    def artifactName = "my-app-${BUILD_NUMBER}.jar"
                    sh "${MVN_CMD} clean package"
                    sh """
                        mkdir -p tagged-artifacts
                        cp target/*.jar tagged-artifacts/${artifactName}
                        echo "✅ Tagged JAR: ${artifactName}"
                    """
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                script {
                    def version = "1.0.${BUILD_NUMBER}"
                    def artifactId = "my-app"
                    def groupPath = "com/mycompany/app"
                    def finalArtifact = "${artifactId}-${version}.jar"
                    def uploadPath = "${groupPath}/${artifactId}/${version}"

                    sh """
                        mv tagged-artifacts/my-app-${BUILD_NUMBER}.jar tagged-artifacts/${finalArtifact}
                    """

                    withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            curl -u $USERNAME:$PASSWORD \\
                            --upload-file tagged-artifacts/${finalArtifact} \\
                            ${NEXUS_URL}${uploadPath}/${finalArtifact}
                        """
                    }
                }
            }
        }

        stage('Create Dockerfile') {
            steps {
                script {
                    writeFile file: 'Dockerfile', text: """
                        FROM openjdk:21-jdk-slim
                        WORKDIR /app
                        COPY tagged-artifacts/my-app-*.jar app.jar
                        EXPOSE 8080
                        ENTRYPOINT ["java", "-jar", "app.jar"]
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image to Nexus') {
            steps {
                script {
                    def imageTag = "${NEXUS_DOCKER_REPO}/my-app:${BUILD_NUMBER}"
                    withCredentials([usernamePassword(credentialsId: "${NEXUS_DOCKER_CRED_ID}", usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            echo $PASSWORD | docker login http://${NEXUS_DOCKER_REPO} -u $USERNAME --password-stdin
                            docker push ${imageTag}
                            docker logout http://${NEXUS_DOCKER_REPO}
                        """
                    }
                }
            }
        }

        stage('Delete Old Sonar Projects') {
            steps {
                script {
                    def currentBuild = env.BUILD_NUMBER.toInteger()
                    def minBuildToKeep = currentBuild - MAX_BUILDS_TO_KEEP.toInteger()

                    if (minBuildToKeep > 0) {
                        withCredentials([string(credentialsId: "${SONAR_CRED_ID}", variable: 'SONAR_TOKEN')]) {
                            for (int i = 1; i <= minBuildToKeep; i++) {
                                def oldProject = "${env.JOB_NAME}-${i}".replace('/', '-')
                                echo "Deleting old Sonar project: ${oldProject}"
                                sh """
                                    curl -s -o /dev/null -w "%{http_code}" -X POST \\
                                    -H "Authorization: Bearer ${SONAR_TOKEN}" \\
                                    "${SONAR_URL}/api/projects/delete?project=${oldProject}" || true
                                """
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
