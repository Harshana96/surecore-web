pipeline {
  agent any

  environment {
    REGISTRY = "localhost:5000"
    IMAGE    = "surecore-web"
  }

  stages {

    stage('Detect environment') {
      steps {
        script {
          if (env.BRANCH_NAME == 'develop') {
            env.ENV_NAME  = 'dev'
            env.ENV_LABEL = 'DEVELOP'
            env.ENV_HOST  = 'dev.89.167.27.46.nip.io'
            env.REPLICAS  = '1'
          } else if (env.BRANCH_NAME == 'release/qa') {
            env.ENV_NAME  = 'qa'
            env.ENV_LABEL = 'QA'
            env.ENV_HOST  = 'qa.89.167.27.46.nip.io'
            env.REPLICAS  = '1'
          } else if (env.BRANCH_NAME == 'main') {
            env.ENV_NAME  = 'prod'
            env.ENV_LABEL = 'PRODUCTION'
            env.ENV_HOST  = 'prod.89.167.27.46.nip.io'
            env.REPLICAS  = '2'
          } else {
            error("Branch ${env.BRANCH_NAME} not mapped to any environment")
          }
          echo "Target: ${env.ENV_NAME} → ${env.ENV_HOST}"
        }
      }
    }

    stage('Build Docker image') {
      steps {
        sh """
          docker build -t ${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-${env.BUILD_NUMBER} .
          docker tag ${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-${env.BUILD_NUMBER} \
                     ${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-latest
        """
      }
    }

    stage('Push to registry') {
      steps {
        sh """
          docker push ${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-${env.BUILD_NUMBER}
          docker push ${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-latest
        """
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        sh """
          sed 's|ENV_NAME|${env.ENV_NAME}|g' k8s/deployment.yaml | \
          kubectl apply -f - -n ${env.ENV_NAME}

          kubectl apply -f k8s/service.yaml -n ${env.ENV_NAME}

          sed 's|ENV_HOST|${env.ENV_HOST}|g' k8s/ingress.yaml | \
          kubectl apply -f - -n ${env.ENV_NAME}

          kubectl set image deployment/surecore-web \
            surecore-web=${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-latest \
            -n ${env.ENV_NAME}

          kubectl scale deployment/surecore-web \
            --replicas=${env.REPLICAS} \
            -n ${env.ENV_NAME}

          kubectl rollout status deployment/surecore-web \
            -n ${env.ENV_NAME} --timeout=60s
        """
      }
    }

    stage('Smoke test') {
      steps {
        sh """
          sleep 5
          kubectl get pods -n ${env.ENV_NAME}
          echo "SUCCESS - ${env.ENV_NAME} live at http://${env.ENV_HOST}"
        """
      }
    }

  }

  post {
    success {
      echo "Pipeline passed - ${env.ENV_NAME} live at http://${env.ENV_HOST}"
    }
    failure {
      echo "Pipeline failed - check logs"
    }
  }
}