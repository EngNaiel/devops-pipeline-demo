pipeline {
    agent any

    environment {
        // Configure this credential in Jenkins: Manage Jenkins > Credentials
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        IMAGE_NAME            = 'nalshareef0/devops-pipeline-demo'
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
        timeout(time: 5, unit: 'MINUTES') {
            withSonarQubeEnv('SonarQube') {
                sh 'mvn sonar:sonar'
            }
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
        withCredentials([usernamePassword(
            credentialsId: 'github-creds',
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_TOKEN'
        )]) {
            sh '''
                git config user.email "jenkins@local"
                git config user.name "Jenkins"
                sed -i "s|tag: .*|tag: \\"${IMAGE_TAG}\\"|" helm/values.yaml
                git add helm/values.yaml
                git commit -m "Update image tag to ${IMAGE_TAG}"
                git push https://${GIT_USER}:${GIT_TOKEN}@github.com/EngNaiel/devops-pipeline-demo.git HEAD:main
            '''
        }
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
