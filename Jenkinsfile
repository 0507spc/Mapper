pipeline {
  agent { label 'miniserver' }

  environment {
    // Nexus Docker registry
    REGISTRY_URL  = 'nexus.server.cranie.com'
    REGISTRY_REPO = 'maptoposter'          // Docker repo / namespace
    IMAGE_NAME    = 'maptoposter'          // Final image name

    // GitHub repo to clone
    MAPTOP_REPO   = 'https://github.com/originalankur/maptoposter.git'

    // Nexus Docker creds (Jenkins credentials ID)
    REGISTRY_CREDS = 'nexus-docker-creds'
  }

  options {
    timestamps()
  }

  triggers {
    // Example: daily at 02:00; or use pollSCM / webhooks as you prefer
    cron('H 2 * * *')
  }

  stages {

    stage('Checkout pipeline repo') {
      steps {
        // This clones the repo that contains this Jenkinsfile
        checkout scm
      }
    }

    stage('Clone maptoposter') {
      steps {
        sh '''
          rm -rf maptoposter
          git clone "${MAPTOP_REPO}" maptoposter
        '''
      }
    }

    stage('Compute image tags') {
      steps {
        script {
          dir('maptoposter') {
            def gitHash = sh(
              script: 'git rev-parse --short HEAD',
              returnStdout: true
            ).trim()

            // Dynamic tag: <short-hash>-<build-number>
            env.FULL_TAG    = "${gitHash}-${env.BUILD_NUMBER}"
            env.DOCKER_IMAGE = "${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:${FULL_TAG}"
            env.LATEST_IMAGE = "${REGISTRY_URL}/${REGISTRY_REPO}/${IMAGE_NAME}:latest"
          }
          echo "Full tag: ${env.FULL_TAG}"
          echo "Image: ${env.DOCKER_IMAGE}"
          echo "Latest: ${env.LATEST_IMAGE}"
        }
      }
    }

    stage('Docker Build') {
      steps {
        dir('maptoposter') {
          sh '''
            docker build \
              -t "${IMAGE_NAME}:${FULL_TAG}" \
              .
          '''
        }
      }
    }

    stage('Tag for Nexus') {
      steps {
        sh '''
          # Tag dynamic and latest pointing to same local image
          docker tag "${IMAGE_NAME}:${FULL_TAG}" "${DOCKER_IMAGE}"
          docker tag "${IMAGE_NAME}:${FULL_TAG}" "${LATEST_IMAGE}"
        '''
      }
    }

    stage('Login to Nexus') {
      steps {
        withCredentials([usernamePassword(credentialsId: REGISTRY_CREDS,
                                          usernameVariable: 'NEXUS_USER',
                                          passwordVariable: 'NEXUS_PASS')]) {
          sh '''
            echo "${NEXUS_PASS}" | docker login "${REGISTRY_URL}" \
              --username "${NEXUS_USER}" --password-stdin
          '''
        }
      }
    }

    stage('Push to Nexus') {
      steps {
        sh '''
          docker push "${DOCKER_IMAGE}"
          docker push "${LATEST_IMAGE}"
        '''
      }
    }
  }

  post {
    always {
      sh 'docker logout "${REGISTRY_URL}" || true'
      sh 'docker image prune -f || true'
    }
  }
}
