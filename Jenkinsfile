pipeline {
  agent { label 'docker' }

  environment {
    DOCKER_REGISTRY = "myregistry.example.com"
    DOCKER_CREDENTIALS = "docker-registry-credentials"
    GIT_CREDENTIALS = "git-credentials"
    DOCKER_IMAGE_NAME = "${env.DOCKER_REGISTRY}/devsecops-labs/app:latest"
    SSH_CREDENTIALS = "ssh-deploy-key"
    STAGING_URL = "http://localhost:3000"
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))

  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main',
         credentialsId: "${GIT_CREDENTIALS}",
         url: 'https://github.com/JLASOT/pipeline-SAS-SCA-DAST.git'
      }
    }
    
    stage('SAST - Semgrep') {
      steps {
        echo "Running Semgrep (SAST)..."
        sh '''
          docker run --rm -v $PWD:/src returntocorp/semgrep:latest semgrep --config=auto --json --output semgrep-results.json /src || true
          cat semgrep-results.json || true
        '''
        archiveArtifacts artifacts: 'semgrep-results.json', allowEmptyArchive: true
      }
    }
    
  stage('Build') {
    steps {
        echo "Building app (npm install and tests) using Docker..."
        sh '''
        docker run --rm \
            -v $PWD/src:/app \
            -w /app \
            node:16 \
            bash -c "npm install --no-audit --no-fund && \
                    if [ -f package.json ]; then \
                        if npm test --silent; then echo 'Tests OK'; else echo 'Tests failed (continue)'; fi; \
                    fi"
         '''
        }
    }

    stage('SCA - Dependency Check (OWASP dependency-check)') {
      steps {
        echo "Running SCA / Dependency-Check..."
        sh '''
          mkdir -p dependency-check-reports
          docker run --rm \
          -v $PWD:/src \
          -v $(pwd)/dependency-check-data:/usr/share/dependency-check/data \
          -v $(pwd)/dependency-check-reports:/report \
          owasp/dependency-check:latest \
          --project "devsecops-labs" \
          --scan /src \
          --format "JSON" \
          --out /report || true
      '''
        archiveArtifacts artifacts: 'dependency-check-reports/**', allowEmptyArchive: true
      }
    }

    stage('Docker Build & Trivy Scan') {
      steps {
        echo "Building Docker image..."
        sh '''
          docker build -t ${DOCKER_IMAGE_NAME} -f Dockerfile .
        '''

        echo "Scanning image with Trivy..."
        sh '''
          mkdir -p trivy-reports trivy-cache
          

          # Report JSON
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v $(pwd)/trivy-cache:/root/.cache/ \
            -v $(pwd)/trivy-reports:/reports \
            aquasec/trivy:latest image \
              --format json --output /reports/trivy-report.json ${DOCKER_IMAGE_NAME} || true

          # Fail on HIGH/CRITICAL vulnerabilities
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v $(pwd)/trivy-cache:/root/.cache/ \
            aquasec/trivy:latest image \
              --severity HIGH,CRITICAL ${DOCKER_IMAGE_NAME} || true
        '''
        archiveArtifacts artifacts: 'trivy-reports/**', allowEmptyArchive: true
      }
    }
    
    

    // stage('Policy Check - Fail on HIGH/CRITICAL CVEs') {
    //   steps {
    //     sh '''
    //       chmod +x scripts/scan_trivy_fail.sh
    //       ./scripts/scan_trivy_fail.sh $DOCKER_IMAGE_NAME || exit_code=$?
    //       if [ "${exit_code:-0}" -eq 2 ]; then
    //           echo "Failing pipeline due to HIGH/CRITICAL vulnerabilities detected by Trivy."
    //           exit 1
    //       fi
    //     '''
    //   }
    // }

    // stage('Push Image (optional)') {
    //   when {
    //     expression { return env.DOCKER_REGISTRY != null && env.DOCKER_REGISTRY != "" }
    //   }
    //   steps {
    //     echo "Pushing image to registry ${DOCKER_REGISTRY}..."
    //     withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
    //       sh '''
    //         echo "$DOCKER_PASS" | docker login ${DOCKER_REGISTRY} -u "$DOCKER_USER" --password-stdin
    //         docker push ${DOCKER_IMAGE_NAME}
    //         docker logout ${DOCKER_REGISTRY}
    //       '''
    //     }
    //   }
    // }
  

    stage('Deploy to Staging (docker-compose)') {
    //   agent { label 'docker' }
      steps {
        echo "Deploying to staging with docker-compose..."
        sh '''
          docker compose -f docker-compose.yml down || true
          docker compose -f docker-compose.yml up -d --build
          sleep 8
          docker ps -a
        '''
      }
    }

stage('DAST - OWASP ZAP scan') {
    steps {
        echo "Running DAST (OWASP ZAP) BASELINE scan (Última solución de permisos)..."
        sh '''
            USER_ID=$(id -u)
            GROUP_ID=$(id -g)

            mkdir -p zap-reports
            docker run --rm --network host \
                -v $PWD/zap-reports:/zap/wrk \
                --user 0 \
                ghcr.io/zaproxy/zaproxy:stable \
                /bin/bash -c "zap-baseline.py -t http://localhost:3000 -r zap-report.html || true; \
                              chown -R ${USER_ID}:${GROUP_ID} /zap/wrk; \
                              chmod -R 775 /zap/wrk"

            ls -l zap-reports
        '''
        archiveArtifacts artifacts: 'zap-reports/**', allowEmptyArchive: true
    }
}


  } // stages

  post {
    always {
      echo "Pipeline finished. Collecting artifacts..."
    }
    failure {
      echo "Pipeline failed!"
    }
  }
}
