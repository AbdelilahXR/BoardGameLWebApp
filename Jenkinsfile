pipeline {
    agent any
    
    tools{
        jdk "jdk17"
        maven "maven3"
    }
    environment{
        SCANNER_HOME = tool "sonar-scanner"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git credentialsId: 'git-cred', url: 'https://github.com/AbdelilahXR/BoardGameLWebApp.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('File System Scan - Trivy') {
            steps {
                    sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                      sh """$SCANNER_HOME/bin/sonar-scanner \
-Dsonar.projectKey=BoardGame \
-Dsonar.projectName=BoardGame \
-Dsonar.java.binaries=./target/classes"""
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script{
                    waitForQualityGate abortPipeline: false,
                    credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                script{
                    sh 'mvn package'
                }
            }
        }
        
         stage('Publish to Nexus') {
            steps {
                script{
                    withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                        sh 'mvn deploy'
                    }
                }
            }
        }
        
          stage('Build and Tag Docker Image') {
            steps {
                script{
                    // This step should not normally be used in your script. Consult the inline help for details.
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh 'docker build -t abdelilahxr/boardgame:latest . '
                    }
                }
            }
        }
        
        stage('Docker image Scan'){
            steps{
                 sh 'trivy image --format table -o trivy-image-repport.html abdelilahxr/boardgame:latest'
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker push abdelilahxr/boardgame:latest"
                    }
                }
            }
        }
        
        stage('Deploy To Kubernetes') {
            steps{
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.20.134:6443') {
                    sh 'kubectl apply -f deployment-service.yaml'
                }
            }
        }
        
        stage('Verify The Deployment') {
            steps{
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.20.134:6443') {
                    sh 'kubectl get pods -n webapps'
                    sh 'kubectl get svc -n webapps'
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
                to: 'ettarch.abdelilah@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
    
}
