pipeline {
    agent any

    environment {
        // You must set the following environment variables
        SCANNER_HOME = tool 'sonar-scanner'
        AWS_ACCOUNT_ID = credentials('ACCOUNT_ID')
        AWS_ECR_REPO_NAME = credentials('ECR_REPO_WEBAPP')
        AWS_DEFAULT_REGION = 'us-east-1'
        ORGANIZATION_NAME = "darshantech5"
        SERVICE_NAME = "fleetman-webapp"
            
        REPOSITORY_TAG = "${ORGANIZATION_NAME}-${SERVICE_NAME}:${BUILD_ID}"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/"
    }

    stages {
        stage('Preparation') {
            steps {
                cleanWs()
                git credentialsId: 'GitHub', url: "https://github.com/${ORGANIZATION_NAME}/${SERVICE_NAME}"
            }
        }

        stage('Checkout from Git') {
            steps {
                git credentialsId: 'GitHub', url: "https://github.com/${ORGANIZATION_NAME}/${SERVICE_NAME}"
            }
        }

        stage('Sonarqube Analysis') {
            steps {
				withSonarQubeEnv('sonar-server') {
					sh ''' $SCANNER_HOME/bin/sonar-scanner \
					-Dsonar.projectName=fleetman-webapp \
					-Dsonar.projectKey=fleetman-webapp '''
				}
                
            }
        }

        stage('Quality Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' 
                }
            }
        }

        stage('Trivy File Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage("Docker Image Build") {
            steps {
                script {
                    sh 'docker system prune -f'
                    sh 'docker container prune -f'
                    sh 'docker build -t ${AWS_ECR_REPO_NAME} .'
                }
            }
        }

        stage("ECR Image Pushing") {
            steps {
                script {
                        sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}'
                        sh 'docker tag ${AWS_ECR_REPO_NAME} ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                        sh 'docker push ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}'
                }
            }
        }

        stage("TRIVY Image Scan") {
            steps {
                sh 'trivy image ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${BUILD_NUMBER} > trivyimage.txt'
            }
        }

        stage('Checkout Code') {
            steps {
                git credentialsId: 'GitHub', url: "https://github.com/${ORGANIZATION_NAME}/${SERVICE_NAME}"
            }
        }

        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "fleetman-webapp"
                GIT_ORG_NAME = "fleetman-k8s-ci"
            }
            steps {
              dir('manifests') {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "deepaktyagi048@gmail.com"
                        git config user.name "deeepak-tyagii"
                        BUILD_NUMBER=${BUILD_NUMBER}
                        echo $BUILD_NUMBER
                        imageTag=$(grep -oP '(?<=fleetman-webapp:)[^ ]+' deploy.yaml)
                        echo $imageTag
                        sed -i "s/${AWS_ECR_REPO_NAME}:${imageTag}/${AWS_ECR_REPO_NAME}:${BUILD_NUMBER}/" deploy.yaml
                        git add deploy.yaml
                        git commit -m "Update deployment Image to version \${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_ORG_NAME}/${GIT_REPO_NAME} HEAD:master
                    '''
                  }
                }
            }
        }
    }
}
