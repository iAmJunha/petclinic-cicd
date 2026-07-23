pipeline {
    agent any

    environment {
        REGION = 'ap-southeast-1'
        ACCOUNT_ID = '450444046629'
        ECR_REPO = "${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/user15-petclinic"
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                sh 'chmod +x mvnw'
                sh './mvnw clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh './mvnw sonar:sonar -Dsonar.host.url=http://localhost:9000 -Dsonar.token=$SONAR_TOKEN -Dsonar.projectKey=petclinic'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Scan') {
            steps {
                sh "trivy image --exit-code 0 --severity HIGH,CRITICAL ${ECR_REPO}:${IMAGE_TAG} || true"
            }
        }

        stage('ECR Push') {
            steps {
                sh """
                aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                docker push ${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Update Manifest & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-cred', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                    sed -i "s|${ECR_REPO}:.*|${ECR_REPO}:${IMAGE_TAG}|" k8s/deployment.yaml
                    git config user.email "jenkins@ci.local"
                    git config user.name "Jenkins CI"
                    git add k8s/deployment.yaml
                    git commit -m "Update image tag to ${IMAGE_TAG} [ci skip]" || echo "No changes to commit"
                    git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/iAmJunha/petclinic-cicd.git main
                    """
                }
            }
        }
    }
}
