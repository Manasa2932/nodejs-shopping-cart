pipeline {
    agent any
    tools {
        nodejs 'nodejs'
    }
    environment {
        ACR_NAME = 'myacr316'
        ACR_LOGIN_SERVER = 'myacr316.azurecr.io'
        IMAGE_NAME = 'manasashoppingcartimage'
        RESOURCE_GROUP = 'myResourceGroup'
        AKS_CLUSTER = 'myAKSCluster'
        IMAGE_TAG = "v2"
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/Manasa2932/nodejs-shopping-cart.git'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                node -v
                npm -v
                npm install
                '''
            }
        }
        stage('SonarQube SAST Scan') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=nodejs-shopping-cart \
                        -Dsonar.projectName=nodejs-shopping-cart \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://4.239.241.154:9000 \
                        -Dsonar.token=squ_3fd22018647fc2cf30175b859f6f303ae59713d1
                        """
                    }
                }
            }
        }
        stage('Snyk SCA Scan') {
            steps {
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh '''
                    snyk --version
                    snyk auth $SNYK_TOKEN
                    snyk test --severity-threshold=high || true
                    snyk monitor || true
                    '''
                }
            }
        }
        stage('Docker Build') {
            steps {
                sh '''
                docker build -t manasashoppingcartimage:v2 .
                docker tag manasashoppingcartimage:v2 myacr316.azurecr.io/manasashoppingcartimage:v2
                docker tag manasashoppingcartimage:v2 myacr316.azurecr.io/manasashoppingcartimage:latest
                '''
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 0 $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }
        stage('Login to Azure & ACR') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_APP_ID', passwordVariable: 'AZURE_PASSWORD'),
                    string(credentialsId: 'azure-tenant', variable: 'AZURE_TENANT')
                ]) {
                    sh '''
                    az login --service-principal -u $AZURE_APP_ID -p $AZURE_PASSWORD --tenant $AZURE_TENANT
                    az acr login --name $ACR_NAME
                    '''
                }
            }
        }
        stage('Push Image to ACR') {
            steps {
                sh '''
                docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:latest
                '''
            }
        }
        stage('Connect to AKS') {
            steps {
                sh '''
                az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
                '''
            }
        }
    }
}
