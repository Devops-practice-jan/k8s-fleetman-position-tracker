pipeline {
    agent { label 'build' }

    environment {
        AWS_REGION    = "ap-south-1"
        ACCOUNT_ID    = "107153401316"
        ECR_REPO      = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        SERVICE_NAME  = "k8s-fleetman-position-tracker"
        IMAGE_TAG     = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = "default"
        CLUSTER_NAME  = "eks-cluster"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-id', url: "https://github.com/Devops-practice-jan/k8s-fleetman-position-tracker.git"
            }
        }
        stage('SonarQube Analysis Stage') {
            steps{
               withSonarQubeEnv('sonar') {
                sh """
                  mvn clean verify sonar:sonar \
                  -Dsonar.projectKey=k8s-fleetman-position-tracker \
                  -Dsonar.projectName=k8s-fleetman-position-tracker

                """
              }
            }
        }

         stage('Build Jar') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Create ECR Repo if Not Exists') {
            steps {
                sh """
                    aws ecr describe-repositories \
                      --repository-names ${SERVICE_NAME} \
                      --region ${AWS_REGION} || \
                    aws ecr create-repository \
                      --repository-name ${SERVICE_NAME} \
                      --region ${AWS_REGION}
                """
            }
        }

        stage('Build & Push Image') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} \
                      | docker login --username AWS --password-stdin ${ECR_REPO}
                    docker build -t ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG} .
                    docker push ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                    trivy image --format json --output trivy-report.json ${ECR_REPO}/${SERVICE_NAME}:${IMAGE_TAG}
                """
                archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
            }
        }

        stage('AWS Auth & Update kubeconfig') {
            steps {
                sh """
                    aws eks --region ${AWS_REGION} update-kubeconfig --name ${CLUSTER_NAME}
                    kubectl get ns
                """
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                sh """
                    helm upgrade --install ${SERVICE_NAME} ./fleetman-position-tracker-helm \
                      --namespace ${K8S_NAMESPACE} \
                      --set image.repository=${ECR_REPO}/${SERVICE_NAME} \
                      --set image.tag=${IMAGE_TAG} \
                      --wait
                """
            }
        }
    }
    post {
        success {
            echo "✅ Build and deployment pipeline for ${SERVICE_NAME} completed successfully! 🎉"
        }
        failure {
            echo "❌ Build and deployment pipeline for ${SERVICE_NAME} failed."
            // You can add more actions here, like sending notifications
        }
    }
}
