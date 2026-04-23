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
      environment {
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        TF_VAR_env_prefix = 'test'
      }
      steps {
        script {
          dir('terraform') {
            sh "terraform init"
            sh "terraform apply --auto-approve"
            EC2_PUBLIC_IP = sh(
              script: "terraform output ec2_public_ip",
              returnStdout: true
            ).trim()
          }
        }
      }
    }
    stage("deploy") {
      environment {
        DOCKER_CREDS = credentials('docker')
      }
      steps {
        script {
          echo "waiting for EC2 server to initialize"
          sleep(time: 90, unit: "SECONDS")
          echo 'deploying docker image to EC2...'
          echo "${EC2_PUBLIC_IP}"
          
          def shellCmd = "bash ./server-cmds.sh ${params.IMAGE_NAME}:${params.IMAGE_TAG} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
          def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

          sshagent(['server-ssh-key']) {
            sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
            sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
            sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
          }
        }
      }
    }
  }
}
