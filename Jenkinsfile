pipeline {
    agent any

    environment {
        // Configure this credential in Jenkins: Manage Jenkins > Credentials
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        IMAGE_NAME            = "yourdockerhubuser/devops-pipeline-demo"
        IMAGE_TAG             = "${env.BUILD_NUMBER}"
    }

    tools {
        maven 'Maven3' // name configured in Manage Jenkins > Tools
        jdk 'JDK17'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            steps {
                // 'SonarQube' must match the server name set in
                // Manage Jenkins > System > SonarQube servers
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
            }
        }

        stage('Docker Push') {
            steps {
                sh '''
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Update Helm values (GitOps)') {
            steps {
                echo "TODO (Phase 4): bump image.tag to ${IMAGE_TAG} in helm/values.yaml, commit and push so ArgoCD picks it up."
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed - check the stage logs above.'
        }
    }
}
