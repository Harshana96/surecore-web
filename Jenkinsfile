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
            env.PORT      = '3001'
            env.ENV_LABEL = 'DEVELOP'
          } else if (env.BRANCH_NAME == 'release/qa') {
            env.ENV_NAME  = 'qa'
            env.PORT      = '3002'
            env.ENV_LABEL = 'QA'
          } else if (env.BRANCH_NAME == 'main') {
            env.ENV_NAME  = 'prod'
            env.PORT      = '3003'
            env.ENV_LABEL = 'PRODUCTION'
          } else {
            error("Branch ${env.BRANCH_NAME} not mapped to any environment")
          }
          echo "Target: ${env.ENV_NAME} on port ${env.PORT}"
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

    stage('Deploy') {
      steps {
        sh """
          docker stop surecore-${env.ENV_NAME} || true
          docker rm   surecore-${env.ENV_NAME} || true
          docker run -d \
            --name    surecore-${env.ENV_NAME} \
            --restart always \
            -p ${env.PORT}:80 \
            ${env.REGISTRY}/${env.IMAGE}:${env.ENV_NAME}-latest
        """
      }
    }

    stage('Smoke test') {
      steps {
        sh """
          sleep 3
          curl -f http://localhost:${env.PORT}
          echo "SUCCESS - ${env.ENV_NAME} is live on port ${env.PORT}"
        """
      }
    }

  }

  post {
    success {
      echo "Pipeline passed - ${env.ENV_NAME} deployed successfully"
    }
    failure {
      echo "Pipeline failed - check logs"
    }
  }
}