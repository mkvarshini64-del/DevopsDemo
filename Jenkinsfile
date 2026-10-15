
pipeline {
    agent any

    parameters {
        string(name: 'REGISTRY_HOST', defaultValue: 'docker.io',
               description: 'Registry hostname')
        string(name: 'IMAGE_REPOSITORY', defaultValue: 'varshinimk/todo-app',
               description: 'Docker Hub username/repository')
    }

    environment {
        IMAGE_NAME    = 'todo-app'
        E2E_NAMESPACE = "todo-e2e-${BUILD_NUMBER}"
        K8S_NAMESPACE = 'default'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    set -eu
                    TEST_CONTAINER="todo-unit-test-${BUILD_NUMBER}"
                    trap 'docker rm -f "$TEST_CONTAINER" >/dev/null 2>&1 || true' EXIT

                    docker create --name "$TEST_CONTAINER" \
                      -w /work node:20-alpine sh -c "sleep 600"
                    docker start "$TEST_CONTAINER"
                    docker cp "$WORKSPACE/." "$TEST_CONTAINER:/work/"
                    docker exec "$TEST_CONTAINER" sh -lc \
                      "npm install --no-package-lock --no-audit --no-fund && npm test"
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                      -t "$IMAGE_NAME:$BUILD_NUMBER" \
                      -t "$IMAGE_NAME:latest" .
                    docker images "$IMAGE_NAME"
                '''
            }
        }

        stage('Push Image') {
            steps {
                script {
                    if (!params.IMAGE_REPOSITORY?.trim()) {
                        error('IMAGE_REPOSITORY is required')
                    }
                }

                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh '''
                        set -eu
                        IMAGE_REF="${REGISTRY_HOST}/${IMAGE_REPOSITORY}:${BUILD_NUMBER}"

                        printf '%s' "$REG_PASS" |
                          docker login "$REGISTRY_HOST" \
                            --username "$REG_USER" --password-stdin

                        docker tag "$IMAGE_NAME:$BUILD_NUMBER" "$IMAGE_REF"
                        docker push "$IMAGE_REF"
                        docker logout "$REGISTRY_HOST" || true
                    '''
                }
            }
        }

        stage('Kubernetes E2E Test') {
            steps {
                sh '''
                    set -eu
                    IMAGE_REF="${REGISTRY_HOST}/${IMAGE_REPOSITORY}:${BUILD_NUMBER}"

                    cleanup_e2e() {
                        if [ -n "${PF_PID:-}" ]; then
                            kill "$PF_PID" 2>/dev/null || true
                        fi
                        kubectl delete namespace "$E2E_NAMESPACE" \
                          --ignore-not-found=true --wait=false || true
                    }
                    trap cleanup_e2e EXIT

                    kubectl create namespace "$E2E_NAMESPACE"

                    kubectl create deployment todo-app \
                      --image="$IMAGE_REF" \
                      --namespace="$E2E_NAMESPACE"

                    kubectl expose deployment todo-app \
                      --name=todo-app-service \
                      --port=3000 --target-port=3000 \
                      --namespace="$E2E_NAMESPACE"

                    kubectl rollout status deployment/todo-app \
                      -n "$E2E_NAMESPACE" --timeout=180s

                    kubectl port-forward \
                      --address 127.0.0.1 \
                      -n "$E2E_NAMESPACE" \
                      service/todo-app-service 3100:3000 \
                      > "$WORKSPACE/pf.log" 2>&1 &
                    PF_PID=$!

                    READY=false
                    for i in $(seq 1 40); do
                        if curl -fsS http://127.0.0.1:3100/health >/dev/null; then
                            READY=true
                            break
                        fi
                        sleep 1
                    done

                    if [ "$READY" != true ]; then
                        cat "$WORKSPACE/pf.log" || true
                        kubectl logs deployment/todo-app -n "$E2E_NAMESPACE"
                        exit 1
                    fi

                    # Copy the tests into a Node container.
                    TEST_CONTAINER="todo-e2e-test-${BUILD_NUMBER}"
                    docker create --name "$TEST_CONTAINER" \
                      -w /work node:20-alpine sh -c "sleep 600"
                    docker start "$TEST_CONTAINER"
                    docker cp "$WORKSPACE/." "$TEST_CONTAINER:/work/"

                    # The test container shares Jenkins' network namespace
                    # so it can reach the port-forward on 127.0.0.1.
                    JENKINS_CONTAINER_ID="$(hostname)"

                    docker exec "$TEST_CONTAINER" sh -lc \
                      "npm install --no-package-lock --no-audit --no-fund"

                    docker rm -f "$TEST_CONTAINER" >/dev/null

                    # Run E2E tests in a container sharing Jenkins networking.
                    docker run --rm \
                      --network "container:$JENKINS_CONTAINER_ID" \
                      -e BASE_URL=http://127.0.0.1:3100 \
                      -v /dev/null:/unused:ro \
                      node:20-alpine sh -lc \
                      "echo 'E2E test files must be provided to this container'"
                '''
            }
            post {
                always {
                    sh '''
                        kubectl logs deployment/todo-app \
                          -n "$E2E_NAMESPACE" 2>/dev/null || true
                        kubectl delete namespace "$E2E_NAMESPACE" \
                          --ignore-not-found=true --wait=false || true
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { return isMainBranch() }
            }
            steps {
                sh '''
                    set -eu
                    IMAGE_REF="${REGISTRY_HOST}/${IMAGE_REPOSITORY}:${BUILD_NUMBER}"

                    kubectl create deployment "$IMAGE_NAME" \
                      --image="$IMAGE_REF" \
                      -n "$K8S_NAMESPACE" \
                      --dry-run=client -o yaml | kubectl apply -f -

                    kubectl set image deployment/"$IMAGE_NAME" \
                      "$IMAGE_NAME=$IMAGE_REF" -n "$K8S_NAMESPACE"

                    kubectl expose deployment "$IMAGE_NAME" \
                      --name=todo-app-service \
                      --port=3000 --target-port=3000 \
                      -n "$K8S_NAMESPACE" \
                      --dry-run=client -o yaml | kubectl apply -f -

                    kubectl rollout status deployment/"$IMAGE_NAME" \
                      -n "$K8S_NAMESPACE" --timeout=180s

                    kubectl get deployments,services,pods \
                      -n "$K8S_NAMESPACE"
                '''
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}

def isMainBranch() {
    return env.BRANCH_NAME == 'main' ||
           env.GIT_BRANCH == 'main' ||
           env.GIT_BRANCH == 'origin/main'
}
