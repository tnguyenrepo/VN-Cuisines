pipeline {
    agent { label 'foody&argocd' }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME        = "foody"
        IMAGE_REPO      = "raynguyen86/foody"
        GIT_REPO_URL    = "https://gitlab.com/node_repo/VN-cuisine.git"
        ARGOCD_REPO_URL = "https://gitlab.com/node_repo/tokp03.git"

        DOCKER_CREDS    = "raynguyen86-dockerhub"
        GIT_CREDS       = "gitlab-credentials"
    }

    stages {

        stage('Checkout Source') {
            steps {
                deleteDir()
                checkout scmGit(
                    branches: [[name: 'main']],
                    userRemoteConfigs: [[
                        credentialsId: GIT_CREDS,
                        url: GIT_REPO_URL
                    ]]
                )
                script {
                    GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    IMAGE_TAG = "v${BUILD_NUMBER}-${GIT_SHA}"
                }
            }
        }

        stage('SAST - Static Code Analysis') {
            steps {
                sh '''
                semgrep --config=auto . || true
                '''
            }
        }

        stage('Dependency Scan') {
            steps {
                sh '''
                npm install
                npm audit --audit-level=high
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build \
                  --label git_sha=${GIT_SHA} \
                  --label build_number=${BUILD_NUMBER} \
                  -t ${IMAGE_REPO}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Container Image Scan') {
            steps {
                sh '''
                trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Generate SBOM') {
            steps {
                sh '''
                syft ${IMAGE_REPO}:${IMAGE_TAG} -o spdx-json > sbom.json
                '''
                archiveArtifacts artifacts: 'sbom.json'
            }
        }

        stage('Push & Sign Image') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: DOCKER_CREDS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${IMAGE_REPO}:${IMAGE_TAG}

                    cosign sign --yes ${IMAGE_REPO}:${IMAGE_TAG}
                    docker logout
                    '''
                }
            }
        }

        stage('Update ArgoCD GitOps Repo') {
            steps {
                deleteDir()
                withCredentials([
                    usernamePassword(credentialsId: GIT_CREDS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')
                ]) {
                    sh '''
                    git clone https://${GIT_USER}:${GIT_PASS}@gitlab.com/node_repo/tokp03.git
                    cd tokp03/foody-argocd-main/deploy-helm-argocd

                    helm lint .
                    sed -i "s/tag:.*/tag: ${IMAGE_TAG}/" values-test.yaml

                    git config user.email "ci@jenkins.local"
                    git config user.name "jenkins-bot"
                    git add values-test.yaml
                    git commit -m "chore: deploy ${IMAGE_TAG}"
                    git push
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        failure {
            echo "Pipeline failed – deployment blocked"
        }
        success {
            echo "Secure image deployed via ArgoCD"
        }
    }
}
