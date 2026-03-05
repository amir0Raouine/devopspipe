pipeline {
  agent any

  environment {
    // === À ADAPTER ===
    APP_NAME      = "spring-app"
    K8S_NAMESPACE = "devops"
    DOCKER_REPO   = "student-management"   // nom du repo image sur DockerHub
    // ==================
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/amir0Raouine/devopspipe.git'
      }
    }

    stage('Build (Maven)') {
      steps {
        sh '''
          mvn -v
          mvn clean package -DskipTests
        '''
      }
    }

    stage('SonarQube') {
      steps {
        withSonarQubeEnv('sonar') {
          sh '''
            mvn sonar:sonar
          '''
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            set -e

            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

            IMAGE="$DOCKER_USER/$DOCKER_REPO"
            TAG="${BUILD_NUMBER}"

            docker build -t "$IMAGE:$TAG" .
            docker tag "$IMAGE:$TAG" "$IMAGE:latest"

            docker push "$IMAGE:$TAG"
            docker push "$IMAGE:latest"

            echo "Pushed: $IMAGE:$TAG"
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'dockerhub-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            set -e

            IMAGE="$DOCKER_USER/$DOCKER_REPO:${BUILD_NUMBER}"

            kubectl get ns $K8S_NAMESPACE >/dev/null 2>&1 || kubectl create ns $K8S_NAMESPACE

            # (Optionnel) Secret DockerHub pour tirer l'image si ton cluster en a besoin
            kubectl -n $K8S_NAMESPACE delete secret regcred --ignore-not-found=true
            kubectl -n $K8S_NAMESPACE create secret docker-registry regcred \
              --docker-server=https://index.docker.io/v1/ \
              --docker-username="$DOCKER_USER" \
              --docker-password="$DOCKER_PASS"

            # Apply manifest si tu en as (recommandé)
            if [ -d "k8s" ]; then
              echo "Applying ./k8s manifests..."
              kubectl apply -n $K8S_NAMESPACE -f k8s/
            fi

            # Assure que le deployment existe puis met à jour l'image
            kubectl -n $K8S_NAMESPACE get deploy/$APP_NAME >/dev/null 2>&1

            kubectl -n $K8S_NAMESPACE set image deploy/$APP_NAME $APP_NAME="$IMAGE" --record=true
            kubectl -n $K8S_NAMESPACE rollout status deploy/$APP_NAME --timeout=180s
          '''
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          set -e

          # 1) Essaie d'abord via port-forward sur le service (marche même sans NodePort/Ingress)
          # => adapte le port service/actuator selon ton app
          SVC="${APP_NAME}-svc"
          LOCAL_PORT=8089
          TARGET_PORT=8080

          echo "Trying port-forward on service/$SVC ..."
          kubectl -n $K8S_NAMESPACE port-forward svc/$SVC $LOCAL_PORT:$TARGET_PORT >/tmp/pf.log 2>&1 &
          PF_PID=$!

          # petite attente
          sleep 4

          # Exemple: endpoint health (adapte si besoin)
          # /actuator/health (Spring Boot)
          curl -sf "http://127.0.0.1:$LOCAL_PORT/actuator/health" | head -c 500
          echo ""

          kill $PF_PID || true
        '''
      }
    }
  }

  post {
    always {
      sh '''
        echo "=== Pods ==="
        kubectl -n devops get pods -o wide || true
      '''
    }
    failure {
      sh '''
        echo "=== Describe deploy ==="
        kubectl -n devops describe deploy spring-app || true
        echo "=== Recent logs ==="
        kubectl -n devops logs deploy/spring-app --tail=120 || true
      '''
    }
  }
}
