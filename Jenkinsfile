
pipeline {
    agent any

    parameters {
        string(
            name: 'REGISTRY_HOST',
            defaultValue: 'docker.io',
            description: 'Registry hostname'
        )
        string(
            name: 'IMAGE_REPOSITORY',
            defaultValue: 'your-dockerhub-user/varshinimk',
            description: 'Docker Hub username/repository'
        )
    }

    environment {
        IMAGE_NAME     = 'todo-app'
        CONTAINER_NAME = "todo-app-e2e-${BUILD_NUMBER}"
        E2E_NAMESPACE  = "todo-e2e-${BUILD_NUMBER}"
        E2E_PORT       = '3100'
        K8S_NAMESPACE  = 'default'
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

            cleanup() {
                docker rm -f "$TEST_CONTAINER" >/dev/null 2>&1 || true
            }
            trap cleanup EXIT

            docker create \
              --name "$TEST_CONTAINER" \
              -w /work \
              node:20-alpine \
              sh -c "sleep 600"

            docker start "$TEST_CONTAINER"

            docker cp "$WORKSPACE/." "$TEST_CONTAINER:/work/"

            docker exec "$TEST_CONTAINER" \
              sh -lc "npm install --no-package-lock --no-audit --no-fund && npm test"
        '''
    }
}

        
stage('Build Image') {
    steps {
        sh '''
            set -eu

            echo "Building Docker image..."

            docker build \
              -t "$IMAGE_NAME:$BUILD_NUMBER" \
              -t "$IMAGE_NAME:latest" .

            echo "Built images:"
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

        withCredentials([
            usernamePassword(
                credentialsId: 'docker-registry',
                usernameVariable: 'REG_USER',
                passwordVariable: 'REG_PASS'
            )
        ]) {
            sh '''
                set -eu

                IMAGE_REF="${REGISTRY_HOST}/${IMAGE_REPOSITORY}:${BUILD_NUMBER}"

                echo "Logging in to Docker registry..."

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
                bat '''
                    @echo off
                    set "IMAGE_REF=%REGISTRY_HOST%/%IMAGE_REPOSITORY%:%BUILD_NUMBER%"

                    kubectl create namespace "%E2E_NAMESPACE%"
                    if errorlevel 1 exit /b 1

                    kubectl create deployment todo-app ^
                      --image="%IMAGE_REF%" ^
                      --namespace="%E2E_NAMESPACE%"

                    if errorlevel 1 exit /b 1

                    kubectl expose deployment todo-app ^
                      --name=todo-app-service ^
                      --port=3000 ^
                      --target-port=3000 ^
                      --namespace="%E2E_NAMESPACE%"

                    if errorlevel 1 exit /b 1

                    kubectl rollout status deployment/todo-app ^
                      -n "%E2E_NAMESPACE%" --timeout=180s

                    if errorlevel 1 exit /b 1

                    powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $p=Start-Process -FilePath 'kubectl' -ArgumentList @('port-forward','-n',$env:E2E_NAMESPACE,'service/todo-app-service','3100:3000') -PassThru -RedirectStandardOutput ($env:WORKSPACE + '\\pf-out.log') -RedirectStandardError ($env:WORKSPACE + '\\pf-err.log'); Set-Content -Path ($env:WORKSPACE + '\\pf.pid') -Value $p.Id"

                    if errorlevel 1 exit /b 1

                    powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $ready=$false; for ($i=0; $i -lt 40; $i++) { try { Invoke-WebRequest -UseBasicParsing 'http://localhost:3100/health' -TimeoutSec 2 | Out-Null; $ready=$true; break } catch { Start-Sleep -Seconds 1 } }; if (-not $ready) { Get-Content ($env:WORKSPACE + '\\pf-err.log') -ErrorAction SilentlyContinue; kubectl get pods -n $env:E2E_NAMESPACE; kubectl logs deployment/todo-app -n $env:E2E_NAMESPACE; exit 1 }"

                    if errorlevel 1 exit /b 1

                    docker run --rm ^
                      --mount "type=bind,source=%WORKSPACE%,target=/work" ^
                      -w /work ^
                      -e BASE_URL=http://host.docker.internal:3100 ^
                      node:20-alpine ^
                      sh -lc "npm install --no-package-lock --no-audit --no-fund && npm run test:e2e"

                    if errorlevel 1 exit /b 1
                '''
            }
            post {
                always {
                    bat '''
                        @echo off
                        powershell -NoProfile -ExecutionPolicy Bypass -Command "$f=$env:WORKSPACE + '\\pf.pid'; if (Test-Path $f) { $id=Get-Content $f; Stop-Process -Id ([int]$id) -Force -ErrorAction SilentlyContinue; Remove-Item $f -Force -ErrorAction SilentlyContinue }"
                        kubectl logs deployment/todo-app -n "%E2E_NAMESPACE%" 2>NUL
                        kubectl delete namespace "%E2E_NAMESPACE%" --ignore-not-found=true --wait=false
                        exit /b 0
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression { return isMainBranch() }
            }
            steps {
                bat '''
                    @echo off
                    set "IMAGE_REF=%REGISTRY_HOST%/%IMAGE_REPOSITORY%:%BUILD_NUMBER%"

                    kubectl create deployment "%IMAGE_NAME%" ^
                      --image="%IMAGE_REF%" ^
                      --namespace="%K8S_NAMESPACE%" ^
                      --dry-run=client -o yaml > todo-deployment.yaml

                    if errorlevel 1 exit /b 1

                    kubectl apply -f todo-deployment.yaml
                    if errorlevel 1 exit /b 1

                    kubectl set image deployment/%IMAGE_NAME% ^
                      %IMAGE_NAME%="%IMAGE_REF%" ^
                      -n "%K8S_NAMESPACE%"

                    if errorlevel 1 exit /b 1

                    kubectl expose deployment "%IMAGE_NAME%" ^
                      --name=todo-app-service ^
                      --port=3000 ^
                      --target-port=3000 ^
                      --namespace="%K8S_NAMESPACE%" ^
                      --dry-run=client -o yaml | kubectl apply -f -

                    if errorlevel 1 exit /b 1

                    kubectl rollout status deployment/%IMAGE_NAME% ^
                      -n "%K8S_NAMESPACE%" --timeout=180s

                    if errorlevel 1 exit /b 1

                    kubectl get deployments,services,pods ^
                      -n "%K8S_NAMESPACE%"
                '''
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed. Check the failing stage logs.'
        }
    }
}

def isMainBranch() {
    return env.BRANCH_NAME == 'main' ||
           env.GIT_BRANCH == 'main' ||
           env.GIT_BRANCH == 'origin/main'
}
