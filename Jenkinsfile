pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        SCANNER_HOME = tool 'SonarQube-Scanner'
        APP_NAME = "java-registration-app"
        RELEASE = "1.0.0"
        DOCKER_USER = "nehakyatham"
        DOCKER_PASS = "dockerhub-token"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/NehaKyatham/registration-app.git'
            }
        }

        stage('Build Package') {
            steps {
                dir('webapp') {
                    sh "mvn package"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Servers') {
                    dir('webapp') {
                        sh 'mvn -U clean install sonar:sonar'
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarQube-Token'
                }
            }
        }

        stage('Artifactory configuration') {
            steps {
                rtServer(
                    id: "jfrog-server",
                    url: "http://18.171.171.155:8082/artifactory",
                    credentialsId: "jfrog"
                )

                rtMavenDeployer(
                    id: "MAVEN_DEPLOYER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release-local",
                    snapshotRepo: "libs-snapshot-local"
                )

                rtMavenResolver(
                    id: "MAVEN_RESOLVER",
                    serverId: "jfrog-server",
                    releaseRepo: "libs-release",
                    snapshotRepo: "libs-snapshot"
                )
            }
        }

        stage('Deploy Artifacts') {
            steps {
                rtMavenRun(
                    tool: "maven",
                    pom: 'webapp/pom.xml',
                    goals: 'clean install',
                    deployerId: "MAVEN_DEPLOYER",
                    resolverId: "MAVEN_RESOLVER"
                )
            }
        }

        stage('Publish build info') {
            steps {
                rtPublishBuildInfo(
                    serverId: "jfrog-server"
                )
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build("${IMAGE_NAME}")
                    }
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Image Scan") {
            steps {
                script {
                    sh """
                        docker run -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image nehakyatham/java-registration-app:latest \
                        --no-progress --scanners vuln --exit-code 0 \
                        --severity HIGH,CRITICAL --format table > trivyimage.txt
                    """
                }
            }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }
    }
}


