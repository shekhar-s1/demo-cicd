pipeline {
  agent any

  environment {
    // Replace this with your DockerHub repo (e.g. myuser/demo-app)
    DOCKER_REPO = "REPLACE_DOCKERUSER/demo-app"
    // Sonar token and DockerHub creds should be stored in Jenkins credentials
    // credential IDs used below:
    // - sonar-token  (Secret text)
    // - dockerhub-creds (Username with password)
    // - kubeconfig (Secret file containing kubeconfig)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Test') {
      steps {
        sh 'mvn -B -DskipTests=false clean package'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh 'mvn -B sonar:sonar -Dsonar.login=${SONAR_TOKEN} -Dsonar.projectKey=demo-cicd -Dsonar.host.url=http://SONARQUBE_URL:9000'
        }
      }
    }

    stage('Build Docker image') {
      steps {
        script {
          IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0,7)}"
          IMAGE = "${env.DOCKER_REPO}:${IMAGE_TAG}"
        }
        sh 'docker build -t ${IMAGE} .'
      }
    }

    stage('Scan image with Trivy') {
      steps {
        echo 'Scanning image with Trivy...'
        // run Trivy as a container (no local install needed on agent)
        sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE} || true'
      }
    }

    stage('Push image to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin'
          sh 'docker push ${IMAGE}'
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            mkdir -p $HOME/.kube
            cp ${KUBECONFIG_FILE} $HOME/.kube/config
            # replace placeholder and apply
            sed "s|REPLACE_IMAGE|${IMAGE}|g" k8s/deployment.yaml | kubectl apply -f -
            kubectl apply -f k8s/service.yaml
          '''
        }
      }
    }
  }

  post {
    success {
      echo "Pipeline succeeded, deployed ${IMAGE}"
    }
    failure {
      echo "Pipeline failed"
    }
  }
}
