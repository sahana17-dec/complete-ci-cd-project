pipeline {
    agent any

    environment {
        DOCKER_TAG = "20250127"
        IMAGE_NAME = "sahana1712/fullstack"
        AWS_REGION = "ap-south-1"
        CLUSTER_NAME = "microdegree-cluster"
    }

    tools {
        jdk 'java-17'
        maven 'maven'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/sahana17-dec/complete-ci-cd-project.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${DOCKER_TAG} ."
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                script {
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image --format table \
                            -o trivy-image-report.html \
                            sahana1712/fullstack:20250127
                    """
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "docker push ${IMAGE_NAME}:${DOCKER_TAG}"
                }
            }
        }
        
        stage('Updating the Cluster') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}"
                }
            }
        }
        
        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'microdegree-cluster', contextName: '', credentialsId: 'kube', namespace: 'microdegree', restrictKubeConfigAccess: false, serverUrl: 'https://22AA1392787ED20A11F5546AA15185AE.gr7.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n microdegree"
                    sh "kubectl apply -f deployment.yml -n microdegree"
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'microdegree-cluster', contextName: '', credentialsId: 'kube', namespace: 'microdegree', restrictKubeConfigAccess: false, serverUrl: 'https://22AA1392787ED20A11F5546AA15185AE.gr7.ap-south-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n microdegree"
                    sh "kubectl get svc -n microdegree"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'aa5218321@gmail.com',
                    from: 'manojdevopstest@gmail.com',
                    replyTo: 'manojdevopstest@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
