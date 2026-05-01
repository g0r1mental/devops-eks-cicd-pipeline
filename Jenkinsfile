pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        DOCKER_CREDENTIAL_ID = 'docker-cred'
        GITHUB_CREDENTIAL_ID = 'git-cred'
        SONARQUBE_SERVER = 'sonar'
        KUBECONFIG_CREDENTIAL_ID = 'k8-cred'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: env.GITHUB_CREDENTIAL_ID, url: 'https://github.com/abrahimcse/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(env.SONARQUBE_SERVER) {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Docker Image Build') {
            steps {
                script {
                    dockerImage = docker.build("abrahimcse/blog-app:${env.BUILD_ID}")
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image abrahimcse/blog-app:${env.BUILD_ID}"
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('', env.DOCKER_CREDENTIAL_ID) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: env.KUBECONFIG_CREDENTIAL_ID, serverUrl: '']) {
                    sh 'kubectl apply -f kubernetes/deployment.yaml'
                }
            }
        }
    }
}
