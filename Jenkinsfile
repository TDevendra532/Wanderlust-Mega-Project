@Library('SharedLib')

pipeline {
    agent any

    environment {
        SONAR_HOME = tool "Sonar"
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: 'latest', description: 'Frontend Docker Image Tag')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: 'latest', description: 'Backend Docker Image Tag')
    }

    stages {
        stage("Load Credentials") {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS'),
                        usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_PASS'),
                        string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')
                    ]) {
                        env.DOCKERHUB_USER = "devendra532"  // Hardcoded to your Docker Hub username
                        env.GITHUB_USER = GITHUB_USER
                        env.SONAR_TOKEN = SONAR_TOKEN
                    }
                }
            }
        }

        stage("Validate Parameters") {
            steps {
                script {
                    if (!params.FRONTEND_DOCKER_TAG || !params.BACKEND_DOCKER_TAG) {
                        error("Both FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        stage("Workspace Cleanup") {
            steps {
                cleanWs()
            }
        }

        stage("Git: Code Checkout") {
            steps {
                script {
                    code_checkout("https://github.com/${env.GITHUB_USER}/Wanderlust-Mega-Project.git", "main")
                }
            }
        }

        stage("Security Scans") {
            parallel {
                stage("Trivy: Filesystem Scan") {
                    steps {
                        script { trivy_scan() }
                    }
                }

                stage("OWASP: Dependency Check") {
                    steps {
                        script { owasp_dependency() }
                    }
                }
            }
        }

        stage("SonarQube: Code Analysis") {
            steps {
                script { sonarqube_analysis("Sonar", "wanderlust", "wanderlust") }
            }
        }

        stage("SonarQube: Code Quality Gates") {
            steps {
                script { sonarqube_code_quality() }
            }
        }

        stage("Exporting Environment Variables") {
            parallel {
                stage("Backend Env Setup") {
                    steps {
                        script {
                            dir("Automations") {
                                sh "bash updatebackendnew.sh"
                            }
                        }
                    }
                }

                stage("Frontend Env Setup") {
                    steps {
                        script {
                            dir("Automations") {
                                sh "bash updatefrontendnew.sh"
                            }
                        }
                    }
                }
            }
        }

        stage("Docker: Build and Push Images") {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials') {
                        parallel(
                            "Build & Push Backend": {
                                dir('backend') {
                                    docker_build("${env.DOCKERHUB_USER}/wanderlust-backend-beta", params.BACKEND_DOCKER_TAG)
                                    docker_push("${env.DOCKERHUB_USER}/wanderlust-backend-beta", params.BACKEND_DOCKER_TAG)
                                }
                            },
                            "Build & Push Frontend": {
                                dir('frontend') {
                                    docker_build("${env.DOCKERHUB_USER}/wanderlust-frontend-beta", params.FRONTEND_DOCKER_TAG)
                                    docker_push("${env.DOCKERHUB_USER}/wanderlust-frontend-beta", params.FRONTEND_DOCKER_TAG)
                                }
                            }
                        )
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                archiveArtifacts artifacts: '*.xml', followSymlinks: false
                build job: "Wanderlust-CD", parameters: [
                    string(name: 'FRONTEND_DOCKER_TAG', value: params.FRONTEND_DOCKER_TAG),
                    string(name: 'BACKEND_DOCKER_TAG', value: params.BACKEND_DOCKER_TAG)
                ]
            }
        }
    }
}

