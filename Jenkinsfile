pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node19'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('Checkout from Git'){
            steps{
                git branch: 'master', url: 'https://github.com/Rajesh-210/chatbot-ui.git'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Chatbot \
                    -Dsonar.projectKey=Chatbot '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('OWASP FS SCAN') {
            steps {
               withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
          }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.json"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"  
                        sh "docker build -t chatbot ."
                        sh "docker tag chatbot ${DOCKER_USERNAME}/reddit:${BUILD_NUMBER}"
                        sh "docker tag chatbot ${DOCKER_USERNAME}/reddit:latest"
                        sh "docker push ${DOCKER_USERNAME}/chatbot:${BUILD_NUMBER}"
                        sh "docker push ${DOCKER_USERNAME}/chatbot:latest"
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
               withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                   sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                   sh "trivy image ${DOCKER_USERNAME}/chatbot:${BUILD_NUMBER} > trivy.txt"
               }
            }
        }
        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "chatbot-ui"
                GIT_USER_NAME = "Rajesh-210"
            }
            steps {
                dir('K8s'){
                    withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git config user.email "rajeshchilukuri210@gmail.com"
                            git config user.name "Rajesh-210"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            echo $BUILD_NUMBER
                            imageTag=$(grep -oP '(?<=chatbot:)[^ ]+' deployment.yml)
                            echo $imageTag
                            sed -i "s/chatbot:${imageTag}/chatbot:${BUILD_NUMBER}/" deployment.yml
                            git add deployment.yml
                            git commit -m "Update deployment Image to version \${BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        '''
                    }
                }
            }
        }
        stage('Deploy to Kubernetes'){
            steps{
                script{
                    withCredentials([
                        aws(credentialsId: 'aws-key',
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')
                    ]){
                        dir('K8s'){
                            sh '''
                                aws eks update-kubeconfig --name Chatbot-EKS-Cluster --region ap-south-1
                                kubectl apply -f deployment.yml
                                kubectl apply -f service.yml
                                kubectl rollout status deployment/chatbot-clone-deployment --timeout=120s
                                echo "====== Deployed Pods ======"
                                kubectl get pods
                                echo "====== Service Details ======"
                                kubectl get svc
                            '''
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                // Get job name, build number, and pipeline status
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                pipelineStatus = pipelineStatus.toUpperCase()

                // Set the banner color based on the status
                def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                // HTML body for the email
                def body = """
                <body>
                    <div style="border: 2px solid ${bannerColor}; padding: 10px;">
                        <h3 style="color: ${bannerColor};">
                            Pipeline Status: ${pipelineStatus}
                        </h3>
                        <p>Job: ${jobName}</p>
                        <p>Build Number: ${buildNumber}</p>
                        <p>Status: ${pipelineStatus}</p>
                    </div>
                </body>
                """

                // Send email notification
                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'chilukurirajesh@priaccinnovations.ai',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
        
