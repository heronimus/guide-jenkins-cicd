pipeline {

  agent {
    // Run command on Docker-VM, instead inside Jenkins container.
    label 'docker-vm'
  }

  stages {
    stage ('Check Docker') {
      steps {
        echo "Show Docker"
        sh '''
          docker --version
        '''
      }
    }

    stage ('Deploy: Execute docker-compose') {
      steps {
        sh '''
          docker-compose -f docker/docker-compose.yml up $(cat docker/compose-arguments)
        '''
      }
    }

    stage ('Deploy: Show Docker Info') {
      steps {
        sh '''
          docker-compose -f docker/docker-compose.yml ps
          docker container ls
        '''
      }
    }
  }
}
