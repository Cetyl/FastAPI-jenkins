# FastAPI Jenkins Pipeline with ArgoCD Deployment
 This repository contains a Jenkins pipeline script for building, testing, and deploying a FastAPI application, integrated with SonarQube for code analysis and Docker for containerization.

## Prerequisites
Before running the application, ensure you have the following installed:

- Python 3.x

- pip (Python package installer)
  

 Before running the Jenkins pipeline, ensure the following prerequisites are met:

- Jenkins with necessary plugins (Docker, SonarQube Scanner)
  
- Docker installed on the Jenkins agent
  
- SonarQube server accessible and configured
- ArgoCD installed and configured for application deployment

## Installation

Clone the repository and install the required Python packages:
```
git clone https://github.com/Cetyl/fastAPI.git

cd fastAPI

sudo apt update

sudo apt install python3-pip -y

sudo pip install fastapi uvicorn
```


## Usage

Run the FastAPI application locally:

```
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

```

The API server will start at http://localhost:8000.


## Integration with Jenkins CI
This repository is integrated with Jenkins CI to automate the build, test, and deployment process.



### Jenkins Pipeline Configuration
To set up Jenkins CI for this project, create a pipeline job with the following configuration:

```
pipeline {
    agent any
    
    environment {
        APP_PORT = 8000  // Port on which FastAPI will run
        VM_IP = '34.134.188.249'  // Replace with your VM's IP address
        SONARQUBE_URL = "http://${VM_IP}:9000"  // Replace with your SonarQube server URL
        SONAR_AUTH_TOKEN = "cd298146cecdbf794787f0b9fd13e949546b4784"
        SONAR_PROJECT_KEY = "blah"  // Replace with your SonarQube project key
        DOCKER_IMAGE = "cetyl/jenkins_fastapi-cicd:${BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Cetyl/FastAPI-jenkins.git'
            }
        }
        
        stage('Install dependencies') {
            steps {
                sh "sudo apt update"
                sh "sudo apt install python3-pip -y"
                sh "sudo pip install fastapi uvicorn"
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    def deployCmd = "uvicorn main:app --host 0.0.0.0 --port ${APP_PORT} --reload > /dev/null 2>&1 &"
                    sh deployCmd
                    sleep 20
                    def curlCmd = "curl http://${VM_IP}:${APP_PORT}/"
                    sh curlCmd
                }
            }
        }
        
        stage('SonarQube analysis') {
            steps {
                // Securely retrieve SonarQube token from Jenkins credentials
                withCredentials([string(credentialsId: 'SonarQube', variable: 'SONAR_AUTH_TOKEN')]) {
                    // Run SonarQube scanner with credentials
                    sh """
                        sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.host.url=${SONARQUBE_URL} \
                          -Dsonar.login=${SONAR_AUTH_TOKEN}
                    """
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    sh 'docker build -t ${DOCKER_IMAGE} .'
                    def dockerImage = docker.image("${DOCKER_IMAGE}")
                    docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "FastAPI-jenkins"
                GIT_USER_NAME = "Cetyl"
            }
            steps {
                withCredentials([string(credentialsId: 'git', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "rohan.cetyl@gmail.com"
                        git config user.name "Cetyl"
                        BUILD_NUMBER=${BUILD_NUMBER}
                        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" manifests/deployment.yaml
                        git add manifests/deployment.yaml
                        git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh "killall uvicorn"
        }
    }
}

                   
```
### Pipeline Overview
The Jenkins pipeline consists of several stages:

1. Checkout

    - Clones the repository from GitHub to the Jenkins workspace.

2. Install dependencies

    - Updates system packages and installs Python dependencies (fastapi, uvicorn) using pip.

3. Deploy

    - Starts the FastAPI application using uvicorn on port 8000.

4. SonarQube analysis

    - Performs code analysis using SonarQube, with results reported to the configured SonarQube server.

5. Build and Push Docker Image

    - Builds a Docker image of the FastAPI application and pushes it to Docker Hub using credentials stored securely in Jenkins.

6. Update Deployment File via ArgoCD

   - Instead of directly manipulating deployment files, the pipeline triggers ArgoCD for deploying the updated Docker image

  ## Using ArgoCD for Deployment

1. Install ArgoCD

 -  Follow the ArgoCD installation guide to set up ArgoCD in your environment.

2. Configure Application in ArgoCD

- Configure your FastAPI application in ArgoCD:
  - Create an application in ArgoCD that points to your Git repository and Docker image repository.
  - Configure ArgoCD to automatically sync with changes in your Git repository.

3. Deploy Using ArgoCD

- Once the Jenkins pipeline has successfully built and pushed the Docker image, ArgoCD will automatically detect the new image version and deploy it to your environment.

### Usage
To use this Jenkins pipeline:

1. Configure your Jenkins instance with necessary credentials:

   - Docker registry credentials (docker-cred) for pushing Docker images.
   - SonarQube authentication token (SonarQube credentials).


2. Update the pipeline script (Jenkinsfile) as needed:

    - Replace placeholders such as SONAR_PROJECT_KEY, VM_IP, GIT_REPO_NAME, and GIT_USER_NAME with your specific values.


3. Run the Jenkins pipeline manually or trigger it automatically based on your configured triggers (e.g., GitHub webhook).




## Contributing
- Fork the repository, make changes, and submit a pull request.
- Issues and feature requests can be submitted through GitHub Issues.

## License
- This project is licensed under the MIT License.

