#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
  [$class: 'GitSCMSource',
  remote: 'https://github.com/explicit-logic/jenkins-shared-library',
  credentialsId: 'github'
  ]
)

pipeline {   
  agent any
  tools {
    maven 'maven-3.9'
  }
  parameters {
    string(name: 'IMAGE_NAME', defaultValue: 'explicitlogic/app')
    string(name: 'IMAGE_TAG', defaultValue: 'java-maven-2.0')
  }
  stages {
    stage("build app") {
      steps {
        script {
          echo 'building application jar...'
          buildJar()
        }
      }
    }
    stage("build image") {
      steps {
        script {
          echo 'building docker image...'
          buildImage(params.IMAGE_NAME, params.IMAGE_TAG)
          dockerLogin()
          dockerPush(params.IMAGE_NAME, params.IMAGE_TAG)
        }
      }
    }
    stage("provision server") {
      sh "terraform init"
    }
    stage("deploy") {
      steps {
        script {
          echo 'deploying docker image to EC2...'
          
          def shellCmd = "bash ./server-cmds.sh ${params.IMAGE_NAME}:${params.IMAGE_TAG}"
          def ec2Instance = "ec2-user@$35.180.151.121"

          sshagent(['server-ssh-key']) {
            sh "scp -o server-cmds.sh ${ec2Instance}:/home/ec2-user"
            sh "scp -o docker-compose.yaml ${ec2Instance}:/home/ec2-user"
            sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
          }
        }
      }
    }
  }
}
