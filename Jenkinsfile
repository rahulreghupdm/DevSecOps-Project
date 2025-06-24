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
        stage('Docker version check'){
            steps{
                
                sh "docker -v"

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

        stage('Trivy FS Scan') {
            steps {
                sh 'docker run --rm -v $(pwd):/project aquasec/trivy fs /project > trivyfs.txt'
            }
        }

        // stage('OWASP FS SCAN') {
        //     steps {
        //         dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        //         dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //     }
        // }

       stage("Checkout Code") {
            steps {
                checkout scm
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
                sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image rahulreghupdm/netflix:latest > trivyimage.txt'
            }
        }       


        stage('Deploy to Container') {
            steps {
                sh "docker rm -f netflix_container || true"
                sh 'docker run -d --name netflix_container -p 8081:80 rahulreghupdm/netflix:latest'
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
