pipeline {
  agent any

  environment {
    DOCKER_IMAGE = "naveenneth/sit223-demo"
    GIT_COMMIT = ""
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Build') {
      steps {
        sh '''
          echo "==> Building Docker image"
          docker build -t $DOCKER_IMAGE:$GIT_COMMIT .
        '''
      }
      post { success { sh 'docker images | head -n 5' } }
    }

    stage('Test') {
      steps {
        sh 'npm ci'
        sh 'npm test --silent || true'
        junit testResults: '**/junit*.xml', allowEmptyResults: true
      }
    }

    stage('Code Quality') {
      steps {
        withCredentials([string(credentialsId: 'sonar_token', variable: 'SONAR_TOKEN')]) {
          sh '''
            ./node_modules/.bin/jest --coverage --silent || true
            if command -v sonar-scanner >/dev/null 2>&1; then
              sonar-scanner \
                -Dsonar.login=$SONAR_TOKEN \
                -Dsonar.projectKey=sit223-pipeline-demo \
                -Dsonar.host.url=https://sonarcloud.io
            else
              echo "sonar-scanner not found; configure Global Tool or PATH"
              exit 1
            fi
          '''
        }
      }
    }

    stage('Security') {
      steps {
        sh '''
          echo "==> Dependency scan (npm audit)"
          npm audit --audit-level=low || true

          echo "==> Image scan (Trivy)"
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image --exit-code 0 --severity MEDIUM,HIGH,CRITICAL \
            $DOCKER_IMAGE:$GIT_COMMIT

          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image --exit-code 1 --severity CRITICAL \
            $DOCKER_IMAGE:$GIT_COMMIT || (echo "Critical vulns found"; exit 1)
        '''
      }
    }

    stage('Deploy to Staging') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub_creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "==> Push image to DockerHub"
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push $DOCKER_IMAGE:$GIT_COMMIT

            echo "==> Deploy staging (Compose)"
            export DOCKER_IMAGE=$DOCKER_IMAGE
            export GIT_COMMIT=$GIT_COMMIT
            docker compose -f docker-compose.staging.yml down || true
            docker compose -f docker-compose.staging.yml up -d --build
            ./scripts/health_check.sh http://localhost:8081/health
          '''
        }
      }
    }

    stage('Release to Production') {
      when { branch 'main' }
      steps {
        script { env.RELEASE_TAG = "v1.${env.BUILD_NUMBER}" }
        sh '''
          echo "==> Tag + push prod image"
          docker tag $DOCKER_IMAGE:$GIT_COMMIT $DOCKER_IMAGE:$RELEASE_TAG
          docker push $DOCKER_IMAGE:$RELEASE_TAG

          echo "==> Deploy production (Compose)"
          export DOCKER_IMAGE=$DOCKER_IMAGE
          export RELEASE_TAG=$RELEASE_TAG
          docker compose -f docker-compose.prod.yml down || true
          docker compose -f docker-compose.prod.yml up -d
          ./scripts/health_check.sh http://localhost:8080/health
        '''
      }
    }

    stage('Monitoring & Alerting') {
      steps {
        sh '''
          echo "==> Prod health probe"
          code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health || true)
          if [ "$code" != "200" ]; then
            echo "ALERT: Prod not healthy (HTTP $code)"
            exit 1
          fi
          echo "Monitoring OK"
        '''
      }
    }
  }

  post {
    success { echo "✅ Pipeline SUCCESS" }
    failure { echo "❌ Pipeline FAILURE" }
  }
}
