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


