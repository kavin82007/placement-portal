pipeline {
    agent any

    environment {
        IMAGE = 'blackwolf82007/placement-portal'
        TAG   = "build-${env.BUILD_NUMBER}"
        APP   = 'placement-portal'
    }

    stages {

        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t %IMAGE%:%TAG% ."
            }
        }

        stage('Test Image') {
            steps {
                bat "docker image inspect %IMAGE%:%TAG%"
                bat "docker run --rm %IMAGE%:%TAG% ls /usr/share/nginx/html/index.html"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                    bat "docker push %IMAGE%:%TAG%"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    powershell "(Get-Content deployment.yaml) -replace '${env.IMAGE}:.*', '${env.IMAGE}:${env.TAG}' | Set-Content deploy-ci.yaml"
                    bat "kubectl apply -f deploy-ci.yaml"
                    bat "kubectl annotate deployment/%APP% kubernetes.io/change-cause=\"Jenkins build %BUILD_NUMBER% - %TAG%\" --overwrite"
                }
            }
        }

        stage('Verify Rollout / Auto Rollback') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    script {
                        try {
                            bat "kubectl rollout status deployment/%APP% --timeout=90s"
                        } catch (err) {
                            echo "ROLLOUT FAILED for ${env.TAG} - rolling back to previous stable release"
                            bat "kubectl rollout undo deployment/%APP%"
                            bat "kubectl rollout status deployment/%APP% --timeout=90s"
                            bat "kubectl rollout history deployment/%APP%"
                            error("Faulty release ${env.TAG} was rolled back automatically")
                        }
                    }
                }
            }
        }

        stage('Verify Application') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    bat "kubectl get deployment %APP%"
                    bat "kubectl get pods -l app=%APP%"
                    bat "kubectl get svc placement-portal-service"
                }
                bat "curl -sf http://localhost:30080 -o NUL && echo Application is reachable on NodePort 30080"
            }
        }
    }

    post {
        always {
            bat 'docker logout'
        }
    }
}