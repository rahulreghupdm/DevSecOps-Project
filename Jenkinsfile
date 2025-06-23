pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        // stage('Install Dependencies') {
        //     steps {
        //         sh "npm install"
        //     }
        // }

       stage('OWASP FS Scan') {
    steps {
        dependencyCheck  odcInstallation: 'DP-Check',
                         additionalArguments: """
                           --scan .
                           --disableYarnAudit
                           --disableNodeAudit
                           --format HTML,XML
                           --out  odc-report
                           --data \$JENKINS_HOME/odc-data   // persistent cache
                           --purge                         // safety-net: drop bad DBs
                         """,
                         nvdCredentialsId: 'nvd-api-key'   // speeds up updates, avoids 403s
        dependencyCheckPublisher pattern: 'odc-report/dependency-check-report.xml'
    }
}


        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build --build-arg TMDB_V3_API_KEY=49836a14e0518018c5313292 -t netflix ."
                        sh "docker tag netflix rahulreghupdm/netflix:latest"
                        sh "docker push rahulreghupdm/netflix:latest"
                    }
                }
            }
        }

        stage("Trivy Image Scan") {
            steps {
                sh "trivy image rahulreghupdm/netflix:latest > trivyimage.txt"
            }
        }

        stage('Deploy to Container') {
            steps {
                sh 'docker run -d -p 8081:80 rahulreghupdm/netflix:latest'
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>" +
                      "Build Number: ${env.BUILD_NUMBER}<br/>" +
                      "URL: ${env.BUILD_URL}<br/>",
                to: 'rahulreghupdm@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
