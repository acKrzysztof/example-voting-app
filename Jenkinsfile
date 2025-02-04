pipeline {
  agent none
  stages {
    stage('worker-build') {
      agent {
        docker {
          image 'maven:3.6.1-jdk-8-alpine'
          args '-v $HOME/.m2:/root/.m2'
        }

      }
      when {
        changeset '**/worker/**'
      }
      steps {
        echo 'Compiling worker app'
        dir(path: 'worker') {
          sh 'mvn compile'
        }

      }
    }

    stage('worker-test') {
      agent {
        docker {
          image 'maven:3.6.1-jdk-8-alpine'
          args '-v $HOME/.m2:/root/.m2'
        }

      }
      when {
        changeset '**/worker/**'
      }
      steps {
        echo 'Running tests'
        dir(path: 'worker') {
          sh 'mvn clean test'
        }

      }
    }

    stage('worker-package') {
      agent {
        docker {
          image 'maven:3.6.1-jdk-8-alpine'
          args '-v $HOME/.m2:/root/.m2'
        }

      }
      when {
        branch 'master'
        changeset '**/worker/**'
      }
      steps {
        echo 'Creating packages'
        dir(path: 'worker') {
          sh 'mvn package -DskipTests'
          archiveArtifacts(artifacts: '**/target/*.jar', fingerprint: true)
        }

      }
    }

    stage('worker-docker-package') {
      agent any
      when {
        branch 'master'
        changeset '**/worker/**'
      }
      steps {
        echo 'Creating docker image'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def workerImage = docker.build("ackrzysztof/worker:v${env.BUILD_ID}", "./worker")
            workerImage.push()
            workerImage.push("${env.BRANCH_NAME}")
          }
        }

      }
    }

    stage('result-build') {
      agent {
        docker {
          image 'node:8.9-alpine'
        }

      }
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Building result app'
        dir(path: 'result') {
          sh 'npm install'
        }

      }
    }

    stage('result-test') {
      agent {
        docker {
          image 'node:8.9-alpine'
        }

      }
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Running tests'
        dir(path: 'result') {
          sh 'npm install'
          sh 'npm test'
        }

      }
    }

    stage('result-docker imgage build and push') {
      agent any
      when {
        changeset '**/result/**'
      }
      steps {
        echo 'Create docker image and push'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def resultImage = docker.build("ackrzysztof/result:v${env.BUILD_ID}", './result')
            resultImage.push()
            resultImage.push("${env.BRANCH_NAME}")
          }
        }

      }
    }

    stage('vote-Build') {
      agent {
        docker {
          image 'python:2.7.16-alpine'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Running build vote app'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
        }

      }
    }

    stage('vote-Test') {
      agent {
        docker {
          image 'python:2.7.16-alpine'
          args '--user root'
        }

      }
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Running tests vote app'
        dir(path: 'vote') {
          sh 'pip install -r requirements.txt'
          sh 'nosetests -v'
        }

      }
    }

    stage('vote-integration-test'){
      agent any
      when{
        changeset '**/vote/**'
        branch 'master'
      }
      steps{
        echo 'Running integration tests on vote app'
        dir('vote'){
          sh 'integration_test.sh'
        }
      }
    }
    stage('vote-Docker image create and push') {
      agent any
      when {
        changeset '**/vote/**'
      }
      steps {
        echo 'Create docker image and push'
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerlogin') {
            def voteImage = docker.build("ackrzysztof/vote:v${env.BUILD_ID}", './vote')
            voteImage.push()
            voteImage.push("${env.BRANCH_NAME}")
          }
        }

      }
    }

    stage('SonarQube'){
      agent any
//      when{
//        branch 'master'
//      }
      environment{
        sonarpath = tool 'SonarQube'
      }

      steps {
        echo 'Running SonarQube Analysis...'
        withSonarQubeEnv('sonar-instavote') {
          sh "${sonarpath}/bin/sonar-scanner \
              -Dproject.settings=sonar-project.properties \
              -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
        }
      }
    }
    stage('Quality Gate'){
      steps{
        timeout(time: 1, unit: 'HOURS') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    stage('deploy to dev') {
      agent any
      when {
        branch 'master'
      }
      steps {
        echo 'Deploy instavote app with docker compose'
        sh 'docker-compose up -d'
      }
    }

  }
  post {
    always {
      echo 'example app pipeline completed'
    }

    success {
      slackSend(message: "Build Succedeed - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
    }

    failure {
      slackSend(message: "Build Failed - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
    }

  }
}
