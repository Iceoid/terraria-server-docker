pipeline {
  environment {
    userName = "hexlo"
    imageName = "terraria-server-docker"
    buildVersion = '1434'
    tag = buildVersion ?: 'latest'
    gitRepo = "https://github.com/${userName}/${imageName}.git"
    dockerhubRegistry = "${userName}/${imageName}"
    githubRegistry = "ghcr.io/${userName}/${imageName}"
    
    dockerhubCredentials = 'DOCKERHUB_TOKEN'
    githubCredentials = 'GITHUB_TOKEN'
    
    dockerhubImage = ''
    dockerhubImageLatest = ''
    githubImage = ''
    
    serverVersion = ''
    versionTag = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git branch: 'main', credentialsId: 'GITHUB_TOKEN', url: gitRepo
      }
    }
    stage('Getting Latest Version') {
      steps {
        script {
          
          if (tag == 'latest') {
            serverVersion = sh(script: "${WORKSPACE}/get-latest-version.sh", , returnStdout: true).trim()
          }
          else {
            serverVersion = buildVersion
          }
          
          versionTag = sh(script: "echo $serverVersion | sed 's/./&./g;s/\\.\$//'", , returnStdout:true).trim()
          echo "serverVersion=${serverVersion}"
          echo "versionTag=${versionTag}"
          
        }
      }
    }
    stage('Building image') {
      steps{
        script {
          // Docker Hub
          dockerhubImageLatest = docker.build( "${dockerhubRegistry}:${tag}", "--build-arg VERSION=${serverVersion} ." )
          dockerhubImageBuildNum = docker.build( "${dockerhubRegistry}:${BUILD_NUMBER}", "--build-arg VERSION=${serverVersion} ." )
          if (serverVersion) {
            dockerhubImageVerNum = docker.build( "${dockerhubRegistry}:${versionTag}", "--build-arg VERSION=${serverVersion} ." )
          }
          
          // Github
          githubImage = docker.build( "${githubRegistry}:${tag}", "--build-arg VERSION=${serverVersion} ." )
          githubImageBuildNum = docker.build( "${githubRegistry}:${BUILD_NUMBER}", "--build-arg VERSION=${serverVersion} ." )
          if (serverVersion) {
            githubImageVerNum = docker.build( "${githubRegistry}:${versionTag}", "--build-arg VERSION=${serverVersion} ." )
          }
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          // Docker Hub
          docker.withRegistry( '', "${dockerhubCredentials}" ) {
            dockerhubImageLatest.push()
            dockerhubImageBuildNum.push()
            if (dockerhubImageVerNum) {
              dockerhubImageVerNum.push()
            }
          }
          // Github
          docker.withRegistry("https://${githubRegistry}", "${githubCredentials}" ) {
            githubImage.push()
            githubImageBuildNum.push()
            if (githubImageVerNum) {
              githubImageVerNum.push()
            }
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        // Docker Hub
        sh "docker rmi ${dockerhubRegistry}:${tag}"
        sh "docker rmi ${dockerhubRegistry}:${BUILD_NUMBER}"
        sh "docker rmi ${dockerhubRegistry}:${versionTag}"
        
        // Github
        sh "docker rmi ${githubRegistry}:${tag}"
        sh "docker rmi ${githubRegistry}:${BUILD_NUMBER}"
        sh "docker rmi ${githubRegistry}:${versionTag}"
      }
    }
  }
  post {
    failure {
        mail bcc: '', body: "<b>Jenkins Build Report</b><br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} \
        <br>Build URL: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', \
        subject: "Jenkins Build Failed: ${env.JOB_NAME}", to: "jenkins@mindlab.dev";  

    }
  }
}
