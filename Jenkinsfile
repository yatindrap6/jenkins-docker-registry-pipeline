/*
  Jenkins Declarative Pipeline: Build & Push Docker image to self-hosted registry

  Requirements:
  - Jenkins installed natively (not a container)
  - Docker Desktop running with WSL integration (jenkins user in 'docker' group)
  - Self-hosted registry reachable (e.g., host.minikube.internal:5000)
  - Plugins: Pipeline, Git, Docker Pipeline (shell is used for docker commands)

  Notes:
  - If your registry is "insecure", Docker Desktop -> Settings -> Docker Engine:
      { "insecure-registries": ["localhost:5000","host.minikube.internal:5000"] }
    then restart Docker Desktop.
*/

pipeline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    durabilityHint('PERFORMANCE_OPTIMIZED')
  }

  parameters {
    string(name: 'REGISTRY_HOST', defaultValue: 'host.minikube.internal:5000',
           description: 'Self-hosted Docker registry host:port')
    string(name: 'IMAGE_NAME', defaultValue: 'flask-jenkins-demo',
           description: 'Image repository name (no registry prefix)')
    string(name: 'DOCKERFILE', defaultValue: 'Dockerfile',
           description: 'Path to Dockerfile relative to workspace root')
    string(name: 'BUILD_CONTEXT', defaultValue: '.',
           description: 'Docker build context path')
    booleanParam(name: 'REGISTRY_AUTH', defaultValue: false,
           description: 'Tick if your registry requires authentication')
    credentials(name: 'REGISTRY_CREDENTIALS_ID', defaultValue: '',
           description: 'Jenkins credentials ID for registry (user/pass or token). Used only if REGISTRY_AUTH=true')
  }

  environment {
    TAG_BUILD   = "${env.BUILD_NUMBER}"
    // These will be set in 'Compute Metadata' stage:
    TAG_COMMIT  = ''
    IMAGE_LATEST = ''
    IMAGE_BUILD  = ''
    IMAGE_COMMIT = ''
  }

  stages {

    stage('Checkout') {
      steps {
        // If the job is configured with "Pipeline script from SCM", Jenkins checks it out automatically.
        // Keep this for freestyle/inline pipeline usage:
        checkout scm
      }
    }

    stage('Preparation') {
      steps {
        sh '''
          echo "==> Docker version"
          docker version
          echo "==> Workspace: ${PWD}"
          ls -la
        '''
      }
    }

    stage('Compute Metadata') {
      steps {
        script {
          def shortSha = sh(script: 'git rev-parse --short HEAD || echo nogit', returnStdout: true).trim()
          env.TAG_COMMIT = shortSha
          env.IMAGE_LATEST = "${params.REGISTRY_HOST}/${params.IMAGE_NAME}:latest"
          env.IMAGE_BUILD  = "${params.REGISTRY_HOST}/${params.IMAGE_NAME}:${env.TAG_BUILD}"
          env.IMAGE_COMMIT = "${params.REGISTRY_HOST}/${params.IMAGE_NAME}:${env.TAG_COMMIT}"

          currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.TAG_COMMIT}"
          echo "Computed tags:"
          echo "  ${env.IMAGE_BUILD}"
          echo "  ${env.IMAGE_COMMIT}"
          echo "  ${env.IMAGE_LATEST}"
        }
      }
    }

    stage('Docker Login (if required)') {
      when { expression { return params.REGISTRY_AUTH } }
      steps {
        withCredentials([usernamePassword(credentialsId: params.REGISTRY_CREDENTIALS_ID,
                                          usernameVariable: 'REG_USER',
                                          passwordVariable: 'REG_PASS')]) {
          sh '''
            echo "Logging into ${REGISTRY_HOST} ..."
            echo "$REG_PASS" | docker login ${REGISTRY_HOST} -u "$REG_USER" --password-stdin
          '''
        }
      }
    }

    stage('Build Image') {
      steps {
        sh '''
          set -eux
          docker build -f "${DOCKERFILE}" \
            -t "${IMAGE_BUILD}" \
            -t "${IMAGE_COMMIT}" \
            -t "${IMAGE_LATEST}" \
            "${BUILD_CONTEXT}"
        '''
      }
    }

    stage('Push Image') {
      steps {
        // retry to handle transient registry/network hiccups
        retry(2) {
          sh '''
            set -eux
            docker push "${IMAGE_BUILD}"
            docker push "${IMAGE_COMMIT}"
            docker push "${IMAGE_LATEST}"
          '''
        }
      }
    }

    stage('Verify in Registry') {
      steps {
        sh '''
          echo "Listing image manifests from registry (if Registry HTTP v2 allows):"
          curl -sS "http://${REGISTRY_HOST}/v2/" || true
          docker image ls | grep "${IMAGE_NAME}" || true
        '''
      }
    }

    // Optional deploy stage to a K8s cluster (uncomment and adapt if you want automatic rollout)
    /*
    stage('Deploy to Kubernetes') {
      steps {
        sh '''
          # Example: update image in an existing deployment
          kubectl set image deployment/flask-app flask="${IMAGE_LATEST}" --record || true
          kubectl rollout status deployment/flask-app
        '''
      }
    }
    */
  }

  post {
    success {
      echo "✅ Build & push complete: ${env.IMAGE_BUILD}"
    }
    failure {
      echo "❌ Pipeline failed. Check logs above."
    }
    always {
      sh 'docker image ls | head -n 30 || true'
    }
  }
}

