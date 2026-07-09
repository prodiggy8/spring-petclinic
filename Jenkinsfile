pipeline {
  agent any

  // poll scm every minute
  triggers { pollSCM('* * * * *') }

  environment {
    IMAGE_REPO    = 'petclinic'
    PUSH_REGISTRY = 'localhost:5000'
    TAG           = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Build & Test') {
      steps {
        sh './mvnw -B clean package'
      }
      post {
        always { junit 'target/surefire-reports/*.xml' }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh './mvnw -B sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -Dsonar.projectKey=petclinic -Dsonar.token=$SONAR_TOKEN'
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        sh 'docker build -t $PUSH_REGISTRY/$IMAGE_REPO:$TAG .'
        sh 'docker push $PUSH_REGISTRY/$IMAGE_REPO:$TAG'
      }
    }

    stage('Deploy (Ansible)') {
      steps {
        sshagent(['petclinic-vm-ssh']) {
          sh 'ansible-playbook -i ansible/inventory.ini ansible/deploy.yml -e tag=$TAG'
        }
      }
    }

    stage('ZAP Scan') {
      steps {
        sh 'docker run --rm --user 0 --network devsecops-net -v zap_wrk:/zap/wrk zaproxy/zap-stable zap-baseline.py -t http://192.168.56.10:8080 -r zap-report.html || true'
        sh 'docker run --rm -v zap_wrk:/zap/wrk busybox cat /zap/wrk/zap-report.html > zap-report.html || true'
      }
    }
  }

  post {
    always {
      publishHTML(target: [reportDir: '.', reportFiles: 'zap-report.html', reportName: 'ZAP Baseline Report', keepAll: true, allowMissing: true])
      archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
    }
  }
}
