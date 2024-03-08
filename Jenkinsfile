pipeline {
  environment {
    userName = "hexlo"
    imageName = "terraria-server-docker"
    // Set buildVersion to manually change the server version. Leaving empty will default to 'latest'
    buildVersion = 'latest'
    tag = "${buildVersion ? buildVersion : 'latest'}"
    gitRepo = "https://github.com/${userName}/${imageName}.git"
    dockerhubRegistry = "${userName}/${imageName}"
    githubRegistry = "ghcr.io/${userName}/${imageName}"
    arch='amd64'
    
    dockerhubCredentials = 'DOCKERHUB_TOKEN'
    githubCredentials = 'GITHUB_TOKEN'
    
    dockerhubImage = ''
    dockerhubImageLatest = ''
    githubImage = ''
    
    serverVersion = ''
    versionTag = ''
  }
  agent any
  triggers {
        cron('H H/2 * * *')
  }
  stages {
    stage('Cloning Git') {
      steps {
        environment { HOME = "${env.WORKSPACE}" }
        git branch: 'main', credentialsId: 'GITHUB_TOKEN', url: gitRepo
      }
    }
    stage('Getting Latest Version') {
      steps {
        environment { HOME = "${env.WORKSPACE}" }
        script {
          echo "tag=${tag}"
          if (tag == 'latest') {
            latestVersion = sh(script: "${WORKSPACE}/.scripts/get-terraria-version.sh", returnStdout: true).trim()
            versionTag = sh(script: "echo $latestVersion | sed 's/[0-9]/&./g;s/\\.\$//'", returnStdout:true).trim()
          }
          else {
            versionTag = sh(script: "echo $buildVersion | sed 's/[0-9]/&./g;s/\\.\$//'", returnStdout:true).trim()
          }
          echo "versionTag=${versionTag}"
          echo "buildVersion=${buildVersion}"
        }
      }
    }
    stage('Building amd64 image') {
      steps{
        environment { HOME = "${env.WORKSPACE}" }
        script {
          arch='amd64'
          "Building ${dockerhubRegistry}-${arch}:${tag}"
          // Docker Hub
          dockerhubImage = docker.build( "${dockerhubRegistry}-${arch}:${tag}", "--target build-amd64 --platform linux/amd64 --no-cache --build-arg VERSION=${buildVersion} ." )
          
          // Github
          githubImage = docker.build( "${githubRegistry}-${arch}:${tag}", "--target build-amd64 --platform linux/amd64 --no-cache --build-arg VERSION=${buildVersion} ." )
        }
      }
    }
    stage('Building arm64 image') {
      steps{
        environment { HOME = "${env.WORKSPACE}" }
        script {
          arch='arm64'
          echo "Building ${dockerhubRegistry}-${arch}:${tag}"
          // Docker Hub
          dockerhubImage = docker.build( "${dockerhubRegistry}-${arch}:${tag}", "--target build-arm64 --platform linux/arm64 --no-cache --build-arg VERSION=${buildVersion} ." )
          
          // Github
          githubImage = docker.build( "${githubRegistry}-${arch}:${tag}", "--target build-arm64 --platform linux/arm64 --no-cache --build-arg VERSION=${buildVersion} ." )
        }
      }
    }
    stage('Deploy Image') {
      steps{
        environment { HOME = "${env.WORKSPACE}" }
        script {
          // Docker Hub
          echo "create manifest"
          sh "docker manifest create --amend ${dockerhubRegistry}:${tag} ${dockerhubRegistry}-amd64:${tag} ${dockerhubRegistry}-arm64:${tag}"
          docker.withRegistry( '', "${dockerhubCredentials}" ) {
            dockerhubImage.push("${tag}")
            dockerhubImage.push("${versionTag}")
          }
          // Github
          docker.withRegistry("https://${githubRegistry}", "${githubCredentials}" ) {
            githubImage.push("${tag}")
            githubImage.push("${versionTag}")
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        environment { HOME = "${env.WORKSPACE}" }
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
          // Docker Hub
          sh "docker rmi ${dockerhubRegistry}:${tag}"
          sh "docker rmi ${dockerhubRegistry}:${versionTag}"
          sh "docker rmi ${dockerhubRegistry}-amd64:${tag}"
          sh "docker rmi ${dockerhubRegistry}-arm64:${tag}"

          // Github
          sh "docker rmi ${githubRegistry}:${tag}"
          sh "docker rmi ${githubRegistry}:${versionTag}"
        }

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
