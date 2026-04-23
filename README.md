# Module 12 - Infrastructure as Code with Terraform

This repository contains a demo project created as part of my **DevOps studies** in the [TechWorld with Nana – DevOps Bootcamp](https://www.techworld-with-nana.com/devops-bootcamp).

**Demo Project:** Complete CI/CD with Terraform

**Technologies used:** Terraform, Jenkins, Docker, AWS, Git, Java, Maven, Linux, Docker Hub

**Project Description:**

Integrate provisioning stage into complete CI/CD Pipeline to automate provisioning server instead of deploying to an existing server.

- Create SSH Key Pair
- Install Terraform inside Jenkins container
- Add Terraform configuration to application’s git repository
- Adjust Jenkinsfile to add “provision” step to the CI/CD pipeline that provisions EC2 instance
- So the complete CI/CD project we build has the following configuration:
  - a.CI step: Build artifact for Java Maven application
  - b.CI step: Build and push Docker image to Docker Hub
  - c.CD step: Automatically provision EC2 instance using TF
  - d.CD step: Deploy new application version on the provisioned EC2 instance with Docker Compose

---

### Prerequisites

Before starting, complete the following setup modules:

- **Jenkins on DigitalOcean:** [jenkins-module-8.1](https://github.com/explicit-logic/jenkins-module-8.1)
- **Build Tools (Maven, Node):** [jenkins-module-8.2](https://github.com/explicit-logic/jenkins-module-8.2?tab=readme-ov-file#install-build-tools-maven-node-in-jenkins)

- Install `SSH Agent` plugin

---

### Configure a Multibranch Pipeline in Jenkins

1. Go to **Dashboard** → **New Item**
2. Name it `java-maven-app`, select **Multibranch Pipeline**, click **OK**

**Branch Sources:**

Click **Add source** → **GitHub** and fill in:

| Field | Value |
|---|---|
| Credentials | `github` |
| Repository HTTPS URL | `https://github.com/explicit-logic/terraform-module-12.5` |

Click **Validate** to confirm access.

**Behaviors** — click **Add** and enable:
- `Discover branches`

**Build Configuration:**
- Script Path: `Jenkinsfile`

3. Click **Save** — Jenkins will scan the repository and create a job for each branch.

---



### Create SSH Key Pair

Go to AWS -> EC2 -> Key pairs

- Create key pair

Name: `myapp-key-pair`

![](./images/create-key-pair.png)

#### Store SSH Key Pair in Jenkins

Go to `java-maven-app` → **Credentials** → **Global** → **Add Credentials**

Kind: `SSH Username with private key`

ID: `server-ssh-key`

Username: `ec2-user`

Private Key -> Enter directly

Paste the content of pem file `cat ~/Downloads/myapp-key-pair.pem`

![](./images/add-ssh-key-to-jenkins.png)

Restrict access to pem file

```sh
chmod 400 ~/Downloads/myapp-key-pair.pem
```


### Add AWS Credentials to Jenkins

1. Go to `java-maven-app` → **Credentials** → **Global** → **Add Credentials**
2. Add two **Secret text** credentials for admin user:

| ID                       | Secret               |
| ------------------------ | -------------------- |
| `AWS_ACCESS_KEY_ID`      | `<Access key>`       |
| `AWS_SECRET_ACCESS_KEY`  | `<Secret access key>` |

![Jenkins AWS credentials](./images/jenkins-aws-creds.png)


### Install Terraform inside Jenkins container

Connect to Jenkins droplet

```sh
ssh root@<DROPLET-IP>
```

Enter the Jenkins container as `root`:

```sh
docker ps
docker exec -it -u 0 <container_id> bash
```

Check your OS version
```sh
cat /etc/os-release
```

install `wget`,`gpg`, `lsb-release`

```sh
apt update
apt install wget
apt install gnupg
apt install lsb-release

# Verify installation
wget --version
gpg --version
```

![](./images/os-release.png)

See Terraform installation instructions: https://developer.hashicorp.com/terraform/install

Install terraform
```sh
wget -O - https://apt.releases.hashicorp.com/gpg | gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com bookworm main" | tee /etc/apt/sources.list.d/hashicorp.list
apt update && apt install terraform

# Verify installation
terrafrom -v
```

![](./images/install-terraform.png)


### Add Terraform configuration to application’s git repository

- Create `terraform` directory
- Create `main.tf`
- Create `variables.tf`
- Create `entry-script.sh` with installation of the docker compose

The installation documentation for Docker Compose standalone can be found here: https://docs.docker.com/compose/install/standalone/

- define default values for variables in `variables.tf`

get your public IP
`curl https://ipinfo.io/ip`


### Adjust Jenkinsfile to add “provision” step to the CI/CD pipeline that provisions EC2 instance


Add the following stage to the `Jenkinsfile`:

```groovy
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
        }
      }
    }
  }
```

See: [](./Jenkinsfile)

### Deploy stage in Jenkinsfile

Add this line to get ec2 instance ip address in "provision server" stage

```groovy
EC2_PUBLIC_IP = sh(
  script: "terraform output ec2_public_ip",
  returnStdout: true
).trim()
```

Add the deploy stage:

```groovy
stage("deploy") {
  steps {
    script {
      echo "waiting for EC2 server to initialize"
      sleep(time: 90, unit: "SECONDS")
      echo 'deploying docker image to EC2...'
      echo "${EC2_PUBLIC_IP}"
      
      def shellCmd = "bash ./server-cmds.sh ${params.IMAGE_NAME}:${params.IMAGE_TAG}"
      def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

      sshagent(['server-ssh-key']) {
        sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
        sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
        sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
      }
    }
  }
}
```

### Login to docker to pull the image

Add the lines below to the deploy stage, it will automatically creates `DOCKER_CREDS_USR`, `DOCKER_CREDS_PSW` env variables
```groovy
environment {
  DOCKER_CREDS = credentials('docker')
}
```

then run the shell script like this

```groovy
def shellCmd = "bash ./server-cmds.sh ${params.IMAGE_NAME}:${params.IMAGE_TAG} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
```

### Run CI/CD pipeline

Connect to the newly created ec2 instance

```sh
ssh -i ~/Downloads/myapp-key-pair.pem ec2-user@<ec2_public_ip>
```

Run
```sh
docker ps
```
