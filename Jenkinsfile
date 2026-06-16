pipeline {
    agent any
    
    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }
    
    environment {
        SONAR_HOST_URL = 'http://13.233.95.166:9000'
        DOCKER_IMAGE = 'sidk27/bankapp:latest'
        EKS_CLUSTER = 'bankapp-cluster'
        AWS_REGION = 'us-east-1'
        KUBE_NAMESPACE = 'webapps'
    }
    
    stages {

        stage('Code Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/17101A0031/Github-Action-Project.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn -B package --file pom.xml -DskipTests'
            }
        }
        
        stage('Security Check') {
            steps {
                sh '''
                    if ! command -v trivy &> /dev/null; then
                        sudo apt-get install -y wget gnupg
                        wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
                        echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
                        sudo apt-get update && sudo apt-get install -y trivy
                    fi
                    trivy fs --format table -o trivy-report.json .
                '''
                sh '''
                    if ! command -v gitleaks &> /dev/null; then
                        sudo apt install -y gitleaks
                    fi
                    gitleaks detect --source . -r gitleaks-report.json -f json || true
                '''
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=bankapp \
                        -Dsonar.host.url=$SONAR_HOST_URL
                    '''
                }
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USERNAME',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_TOKEN
                        docker build -t $DOCKER_IMAGE .
                        docker push $DOCKER_IMAGE
                        docker logout
                    '''
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=$AWS_REGION
                        aws eks update-kubeconfig \
                            --name $EKS_CLUSTER \
                            --region $AWS_REGION
                        kubectl create namespace $KUBE_NAMESPACE \
                            --dry-run=client -o yaml | kubectl apply -f -
                        kubectl apply -f ds.yml
                        kubectl rollout status deployment/bankapp -n $KUBE_NAMESPACE
                        kubectl get pods -n $KUBE_NAMESPACE
                        kubectl get svc -n $KUBE_NAMESPACE
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline Successful!'
        }
        failure {
            echo '❌ Pipeline Failed!'
        }
        always {
            cleanWs()
        }
    }
}
