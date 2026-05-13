pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node23'
    }

    environment {
        IMAGE = "subhashrokkala/zomato"
        REGION = "us-east-1"
        CLUSTER = "mycluster"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                credentialsId: 'git-creds',
                url: 'https://github.com/Subhash-Rokkala/zomato.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh '''
                npm install
                npm run build
                npm test || true
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    def scanner = tool 'sonar-scanner'

                    withSonarQubeEnv('sq') {
                        sh """
                        ${scanner}/bin/sonar-scanner \
                        -Dsonar.projectKey=zomato \
                        -Dsonar.projectName=zomato \
                        -Dsonar.sources=src
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package App') {
            steps {
                sh 'zip -r zomato-build.zip build/'
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {

                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file zomato-build.zip \
                    http://localhost:8081/repository/raw-hosted/zomato-build-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {

                    sh '''
                    docker build -t $IMAGE:$BUILD_NUMBER .
                    docker tag $IMAGE:$BUILD_NUMBER $IMAGE:latest

                    echo $PASS | docker login -u $USER --password-stdin

                    docker push $IMAGE:$BUILD_NUMBER
                    docker push $IMAGE:latest

                    docker logout
                    '''
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                --region $REGION \
                --name $CLUSTER

                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }

        stage('Get Service') {
            steps {
                sh 'kubectl get svc'
            }
        }
    }

    post {

        success {
            echo 'Pipeline Executed Successfully'
        }

        failure {
            echo 'Pipeline Failed'
        }
    }
}
