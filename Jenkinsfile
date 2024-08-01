pipeline {
    agent { label 'slave-1' }

    tools {
        jdk 'jdk-17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('SCM') {
            steps {
                git changelog: false, poll: false, url: 'https://gitlab.com/thecloudsavvy/Boardgame.git'
            }
        }

        stage('Code compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('maven test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('file system scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('Code Quality Check') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Boardgame -Dsonar.projectKey=Boardgame \
                            -Dsonar.java.binaries=. '''
                    }
                }
            }
        }

        stage('Code Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') { // Corrected the typo here from 'Buid' to 'Build'
            steps {
                sh 'mvn clean install'
            }
        }

        stage('deploy to nexus3') {
            steps {
                script {
                    withMaven(globalMavenSettingsConfig: 'global-setting', jdk: 'jdk-17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                        sh 'mvn deploy'
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-token', toolName: 'docker') {
                        sh "docker build -t abatan/boardshack:latest ."
                    }
                }
            }
        }

        stage('Docker image scan') {
            steps {
                sh 'trivy image --format table -o image-trivy.html abatan/boardshack:latest'
            }
        }

        stage('Docker push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-token', toolName: 'docker') {
                        sh "docker push abatan/boardshack:latest"
                    }
                }
            }
        }

        stage('Deploy to k8s') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8s', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.20.147:6443') {
                        sh "kubectl apply -f deployment-service.yaml"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
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
                    to: 'taiwoabatan.co@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
