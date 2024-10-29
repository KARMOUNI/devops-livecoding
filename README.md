DevOps TP-2 Report
 Overview
This report documents the steps taken and concepts learned while configuring CI/CD pipelines with GitHub Actions, Docker, and SonarCloud.

 Goals and Good Practices
Goal: The main objective was to set up a CI/CD pipeline that builds, tests, and deploys a Docker image for the application, ensuring code quality and best practices.
Good Practices: Documenting every step, using secure environment variables, and separating different jobs in the CI/CD pipeline.
 CI/CD Pipeline Details
CI Pipeline - build-and-test
Checkout Code: Clones the repository.
Set up JDK: Uses Amazon Corretto JDK 17.
Build and Test: Runs mvn clean verify with integration tests using Testcontainers.
CD Pipeline - build-and-push-docker-image
Login to DockerHub: Uses GitHub Secrets for DockerHub credentials.
Build and Push Image: Builds Docker images for the backend, database, and HTTP server, then pushes them to DockerHub.
Quality Gate - SonarCloud Analysis
Run SonarCloud Analysis: Configured in main.yml to analyze code quality on each commit. Uses a specific project and organization key.
Questions and Answers

#What are testcontainers?
 Testcontainers are Java libraries that allow integration tests to run in lightweight, throwaway Docker containers. For this TP, Testcontainers were used to spin up a PostgreSQL container to verify database integration. here's an example of testcontainers in the pom.xml:


# documenting the First CI with backend test :
 here's my snippet of code for the main.yaml :

name: CI Workflow

on:
push:
branches:
- main
pull_request:
branches:
- main

jobs:
build-and-test:
runs-on: ubuntu-22.04
steps:
- name: Checkout Code
uses: actions/checkout@v2

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto'  
      - name: Build and Test with Maven
        run: mvn clean verify
 PS: it's validated

 
# Why did we use secured variables?
  Secured variables (GitHub Secrets) are used to protect sensitive information, such as DockerHub credentials, from being exposed in the public repository. They ensure that credentials are only accessible within the GitHub Actions environment.


# Why did we use the needs: build-and-test on the job?
  The needs: build-and-test directive ensures that the Docker image is only built and pushed if the build-and-test job completes successfully. This dependency prevents deploying an untested or failed build.

# For what purpose do we need to push Docker images?
 Pushing Docker images to a repository (like DockerHub) allows for easy deployment and version control of the application. Once an image is built, it can be reused across environments, ensuring consistency.

image
 here's my snippet of code for the main.yaml :

name: CI Workflow

on:
push:
branches:
- main
pull_request:
branches:
- main

jobs:
build-and-test:
runs-on: ubuntu-22.04
steps:
- name: Checkout Code
uses: actions/checkout@v2

      - name: Set up Amazon Corretto JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto'

      - name: Build and Test with Maven
        run: mvn -f simple-api/pom.xml clean verify

build-and-push-docker-image:
needs: build-and-test
runs-on: ubuntu-22.04
steps:
- name: Checkout code
uses: actions/checkout@v2

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin <<< ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push backend image
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build and push database image
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build and push http-server image
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-http-server:latest
          push: ${{ github.ref == 'refs/heads/main' }}
️ PS: It's validated


# What is a Quality Gate?
  A Quality Gate is a set of conditions that code must meet to be considered production-ready. Using SonarCloud, the Quality Gate checks code metrics such as coverage, bugs, and code smells to ensure maintainability and security.

# Explain the Quality Gate configuration used
  The Quality Gate in SonarCloud was configured to check metrics such as code coverage, maintainability, reliability, and security. If any thresholds are not met, the build fails, enforcing a standard for code quality.

#Documenting the quality gate configuration.
  here's my snippet of code for the main.yaml :

name: CI Workflow

on:
push:
branches:
- main
pull_request:
branches:
- main

jobs:
build-and-test:
runs-on: ubuntu-22.04
steps:
- name: Checkout Code
uses: actions/checkout@v2

      - name: Set up Amazon Corretto JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto'

      - name: Build and Test with Maven
        run: mvn -f simple-api/pom.xml clean verify

      - name: Build and Test with Sonar
        run: mvn -B verify sonar:sonar 
          -Dsonar.projectKey=maghwa_devops-livecoding
          -Dsonar.organization=maghwa
          -Dsonar.host.url=https://sonarcloud.io 
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./simple-api/pom.xml


build-and-push-docker-:
needs: build-and-test
runs-on: ubuntu-22.04
steps:
- name: Checkout code

        uses: actions/checkout@v2

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin <<< ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push backend image
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build and push database image
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build and push http-server image
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-http-server:latest
          push: ${{ github.ref == 'refs/heads/main' }}
  PS: It's Validated


in SonarCloud:


Split Pipelines 
The pipeline is split into two jobs:

test-backend: Runs on develop and main branches for tests only.

build-and-push-docker-image: Runs on main branch only after test-backend passes, ensuring the Docker image is pushed only for tested code. image

here's my snippet code for test-backend:

name: Test Backend
on:
push:
branches: [main, develop]
pull_request:

jobs:
test-backend:
runs-on: ubuntu-22.04
steps:
- uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses : actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'corretto'

      - name: Build and test with Maven
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=maghwa_devops-livecoding -Dsonar.organization=maghwa -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./simple-api/pom.xml
here's my snippet code for build-and-push-docker-image:
name: Build and Push Docker Image
on:
workflow_run:
branches: [main]
workflows: ["Test Backend"]
types:
- completed

jobs:
build-and-push-docker-image:
if: ${{github.event.workflow_run.conclusion == 'success'}}
runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}


      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}


      - name: Build image and push front
        uses: docker/build-push-action@v3
        with:
          context: ./http-server
          tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-http-server:latest
          push: ${{ github.ref == 'refs/heads/main' }}
