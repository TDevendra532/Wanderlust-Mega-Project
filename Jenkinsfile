@Library('SharedLib') _

pipeline {
    agent any

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Frontend Docker tag of the image built by the CI job')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Backend Docker tag of the image built by the CI job')
    }

    stages {
        stage("Workspace cleanup") {
            steps {
                script {
                    cleanWs()
                }
            }
        }

        stage('Git: Code Checkout') {
            steps {
                script {
                    code_checkout("https://github.com/TDevendra532/Wanderlust-Mega-Project.git", "main")
                }
            }
        }

        stage('Verify: Docker Image Tags') {
            steps {
                script {
                    echo "FRONTEND_DOCKER_TAG: ${params.FRONTEND_DOCKER_TAG}"
                    echo "BACKEND_DOCKER_TAG: ${params.BACKEND_DOCKER_TAG}"
                }
            }
        }

        stage('Security Scan: OWASP Dependency Check') {
            steps {
                script {
                    sh 'dependency-check --scan . --format HTML --out dependency-check-report.html'
                }
            }
        }

        stage("Update: Kubernetes manifests") {
            steps {
                script {
                    dir('kubernetes') {
                        sh """
                            sed -i -e 's|image: .*trainwithshubham/wanderlust-backend-beta:.*|image: ${params.BACKEND_DOCKER_TAG}|' backend.yaml
                        """
                        sh """
                            sed -i -e 's|image: .*trainwithshubham/wanderlust-frontend-beta:.*|image: ${params.FRONTEND_DOCKER_TAG}|' frontend.yaml
                        """
                    }
                }
            }
        }

        stage("Git: Code update and push to GitHub") {
            steps {
                script {
                    withCredentials([gitUsernamePassword(credentialsId: 'github-cred', gitToolName: 'Default')]) {
                        sh '''
                        git config --global user.email "devendratalhande726@gmail.com"
                        git config --global user.name "TDevendra532"

                        echo "Checking repository status:"
                        git status

                        echo "Adding changes to git:"
                        git add .

                        echo "Committing changes:"
                        git commit -m "Updated Kubernetes manifests with latest image tags"

                        echo "Pushing changes to GitHub:"
                        git push https://github.com/TDevendra532/Wanderlust-Mega-Project.git main
                        '''
                    }
                }
            }
        }

        stage('Docker: Push to My DockerHub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh '''
                        echo "Logging into DockerHub"
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                        echo "Tagging images for my repository"
                        docker tag trainwithshubham/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG} devendra532/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG}
                        docker tag trainwithshubham/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG} devendra532/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG}

                        echo "Pushing images to my DockerHub"
                        docker push devendra532/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG}
                        docker push devendra532/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG}
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                emailext attachLog: true,
                from: 'devendratalhande726@gmail.com',
                subject: "Wanderlust Application Updated & Deployed - '${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><strong>URL:</strong> ${env.BUILD_URL}</p>
                    </body>
                    </html>
                """,
                to: 'devendratalhande726@gmail.com',
                mimeType: 'text/html'
            }
        }
        failure {
            script {
                emailext attachLog: true,
                from: 'devendratalhande726@gmail.com',
                subject: "Wanderlust Application Build Failed - '${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                    </body>
                    </html>
                """,
                to: 'devendratalhande726@gmail.com',
                mimeType: 'text/html'
            }
        }
    }
}
