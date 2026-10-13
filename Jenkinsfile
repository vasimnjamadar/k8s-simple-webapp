pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKER_USER = 'vasimnjamadar'
        IMAGE_NAME = 'k8s-simple-webapp'
        GITOPS_REPO = 'https://github.com/vasimnjamadar/k8s-simple-webapp-gitops.git'
        K8S_NAMESPACE = 'simple-webapp'
        DEPLOYMENT_NAME = 'k8s-simple-webapp'

        PATH = "/opt/homebrew/bin:/usr/local/bin:/Applications/Docker.app/Contents/Resources/bin:${env.PATH}"
    }

    stages {
        stage('Checkout Application') {
            steps {
                checkout scm
                sh '''
                    set -eu
                    test -f Dockerfile
                    test -f index.html
                    test -f Jenkinsfile
                    git log -1 --oneline
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -eu
                    grep -q "Hello from the new Kubernetes application" index.html
                    echo "Application tests passed"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                      -t "$DOCKER_USER/$IMAGE_NAME:$BUILD_NUMBER" \
                      .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu
                        echo "$DH_PASSWORD" | docker login \
                          --username "$DH_USER" \
                          --password-stdin

                        docker push "$DOCKER_USER/$IMAGE_NAME:$BUILD_NUMBER"
                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Repository') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-gitops-creds',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        rm -rf gitops-repo

                        ASKPASS_SCRIPT="$(mktemp)"
                        trap 'rm -f "$ASKPASS_SCRIPT"' EXIT

                        cat > "$ASKPASS_SCRIPT" <<'ASKPASS'
#!/bin/sh
case "$1" in
  *Username*) printf '%s\\n' "$GIT_USERNAME" ;;
  *Password*) printf '%s\\n' "$GIT_TOKEN" ;;
  *) exit 1 ;;
esac
ASKPASS

                        chmod 700 "$ASKPASS_SCRIPT"
                        export GIT_ASKPASS="$ASKPASS_SCRIPT"
                        export GIT_TERMINAL_PROMPT=0

                        git clone "$GITOPS_REPO" gitops-repo
                        cd gitops-repo

                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@example.com"

                        sed -i '' \
                          "s|image: vasimnjamadar/k8s-simple-webapp:.*|image: vasimnjamadar/k8s-simple-webapp:${BUILD_NUMBER}|" \
                          k8s/deployment.yaml

                        grep -F \
                          "image: vasimnjamadar/k8s-simple-webapp:${BUILD_NUMBER}" \
                          k8s/deployment.yaml

                        git add k8s/deployment.yaml
                        git commit -m "Deploy simple-webapp build ${BUILD_NUMBER}"
                        git -c credential.helper= push origin main
                    '''
                }
            }
        }

        stage('Wait for Kubernetes Deployment') {
            steps {
                sh '''
                    set -eu

                    kubectl rollout status \
                      deployment/"$DEPLOYMENT_NAME" \
                      -n "$K8S_NAMESPACE" \
                      --timeout=180s

                    kubectl get pods -n "$K8S_NAMESPACE"
                    kubectl get service k8s-simple-webapp -n "$K8S_NAMESPACE"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
            echo 'Application URL: http://localhost:8090'
        }
        failure {
            echo 'Pipeline failed. Check the first failed stage in the console log.'
        }
        always {
            echo "Finished build ${env.BUILD_NUMBER}"
        }
    }
}
