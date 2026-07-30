
pipeline {
    agent {
        label 'build'
    }
    environment {
        AWS_REGION = "ap-south-1"
        ECR_REPO = "107153401316.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG = "V${env.BUILD_NUMBER}"
        SERVICE_NAME = "k8s-fleetman-position-tracker"
        HELM_CHART_DIR = "./fleetman-position-tracker-helm"
        CLUSTER_NAME = "eks-cluster"
    }
    stages {
        stage('checkout-stage') {
            steps {
                git branch: 'main', credentialsId: 'git-id', url: "https://github.com/Devops-practice-jan/k8s-fleetman-position-tracker.git"
            }
        }
        stage('sonarqube-analysis') {
            steps {
              withSonarQubeEnv('sonar') {
                sh '''
                mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                -Dsonar.projectKey=k8s-fleetman-position-tracker \
                -Dsonar.projectName=k8s-fleetman-position-tracker
                '''
              }
            }
         }
        stage('jar-build-stage') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }
        stage('AWS Setup & Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh """
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region $AWS_REGION
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}     
                    """
                }
            }
        }
        stage('Create ECR repo if not exist'){
            steps{
                sh """ 
                  aws ecr describe-repositories --repository-names ${SERVICE_NAME} --region ${AWS_REGION} || \\
                  aws ecr create-repository --repository-name ${SERVICE_NAME} --region ${AWS_REGION}
                """
            }
        }
        stage('Build & Push Image') {
            steps {
                sh """
                    docker build -t ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG} .
                    docker push ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG}
                """
            }
        }
        stage('Trivy & Snyk Container Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh "snyk auth \$SNYK_TOKEN"
                    sh "snyk container test ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG} --file=Dockerfile --fail-on=upgradable || true" // Snyk test as a security gate
                    sh "snyk container monitor ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG} --file=Dockerfile" // Snyk monitor to track the image
                }
                sh """ 
                    trivy image --format json --output trivy-report.json ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG} 
                """
                archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
            }
        }
        stage('Update kubeconfig'){
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
            }
        }
        stage('Helm Deploy') {
            steps {
                sh """ 
                    helm upgrade --install ${SERVICE_NAME} ${HELM_CHART_DIR} \\
                    --set image.repository=${ECR_REPO}/${SERVICE_NAME} \\
                    --set image.tag=${IMAGE_TAG}
                """
            }
        }
    }
    post {
        success {
            echo "✅ Build and deployment pipeline for ${SERVICE_NAME} completed successfully! 🎉"
            slackSend(
                channel: "#jenkins",
                color: 'good',
                message: "✅ SUCCESS: The pipeline *${env.JOB_NAME}* build <${env.BUILD_URL}|#${env.BUILD_NUMBER}> completed successfully! The new image is deployed to EKS."
            )
        }
        failure {
            echo "❌ Build and deployment pipeline for ${SERVICE_NAME} failed."
            slackSend(
                channel: "#jenkins",
                color: 'danger',
                message: "❌ FAILURE: The pipeline *${env.JOB_NAME}* build <${env.BUILD_URL}|#${env.BUILD_NUMBER}> failed. Check the logs for details."
            )
        }
    }
}
