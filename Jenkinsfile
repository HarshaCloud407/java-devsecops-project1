pipeline {
    agent any

    tools {
        jdk 'jdk21'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME   = 'hariprasad123456/springboot1'
        IMAGE_TAG    = "${env.BUILD_NUMBER}"
        AWS_REGION   = 'ap-southeast-2'
        EKS_CLUSTER  = 'my-eks-cluster'
        NEXUS_URL    = 'http://54.206.109.221:8081/repository/rakesh-repo/'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/HarshaCloud407/java-devsecops-project1.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs-${BUILD_NUMBER}.txt ."
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Nexus Deploy') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-cred',
                                                  usernameVariable: 'NEXUS_USER',
                                                  passwordVariable: 'NEXUS_PASS')]) {
                    // Single quotes on purpose: Maven (not Groovy) resolves ${env.*} in settings.xml
                    writeFile file: 'nexus-settings.xml', text: '''<settings>
  <servers>
    <server>
      <id>rakesh-repo</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASS}</password>
    </server>
  </servers>
</settings>'''
                    sh '''mvn deploy -DskipTests -s nexus-settings.xml \
                          -DaltDeploymentRepository=rakesh-repo::$NEXUS_URL'''
                }
            }
            post {
                always {
                    sh "rm -f nexus-settings.xml"
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectName=springboot \
                                -Dsonar.projectKey=springboot \
                                -Dsonar.java.binaries=target'''
                    }
                }
            }
        }
        stage('Docker build and tag') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }
        stage('Trivy image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${IMAGE_TAG} --format table -o image-${BUILD_NUMBER}.txt"
            }
        }
        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh """
                            docker push ${IMAGE_NAME}:${IMAGE_TAG}
                            docker push ${IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }
        stage('Deploy to kubernetes') {
            steps {
                // FIX: 'aws-cred' is of type "AWS Credentials", so use the AWS binding
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-cred',
                                  accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                    sh """
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER} --kubeconfig kubeconfig-${BUILD_NUMBER}
                        export KUBECONFIG=\$PWD/kubeconfig-${BUILD_NUMBER}

                        # point the deployment at this build's image
                        sed -i "s|image: ${IMAGE_NAME}.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" deployment.yaml

                        kubectl apply -f deployment.yaml -n default
                        kubectl apply -f service.yaml -n default
                        kubectl rollout status deployment/springboot-app -n default --timeout=180s
                    """
                }
            }
            post {
                always {
                    sh "rm -f kubeconfig-${BUILD_NUMBER}"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'fs-*.txt, image-*.txt, target/*.jar', allowEmptyArchive: true
        }
        success {
            echo "Build ${BUILD_NUMBER} succeeded: ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo "Build ${BUILD_NUMBER} failed"
        }
    }
}
