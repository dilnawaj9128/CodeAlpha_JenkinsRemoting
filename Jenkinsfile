pipeline {
  agent { label 'docker-agent' }
  stages {
    stage('Node info') {
      steps { sh 'hostname && uname -m && whoami' }
    }
    stage('Clone') {
      steps {
        git branch: 'main', url: 'https://github.com/dilnawaj9128/CodeAlpha_DockerWebServer.git'
      }
    }
    stage('Docker build') {
      steps { sh 'docker build -t codealpha_web:jenkins .' }
    }
    stage('Run and test') {
      steps {
        sh '''
          docker rm -f jenkins-web || true
          docker run -d --name jenkins-web -p 8090:8080 codealpha_web:jenkins
          sleep 5
          curl -s localhost:8090 | grep Hello
        '''
      }
    }
  }
  post {
    always { sh 'docker rm -f jenkins-web || true' }
  }
}
