# FastAPI App - Hello Web Apps!
 This repository contains a simple FastAPI application that responds with "Hello web apps!".

## Prerequisites
Before running the application, ensure you have the following installed:

Python 3.x

pip (Python package installer)

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
        VM_IP = 'your_vm_ip_address'  // Replace with your VM's IP address
    }
    
    stages {
        stage('Checkout') {
            steps {
                // Checkout your Git repository
                git branch: 'main', url: 'https://github.com/Cetyl/fastAPI.git'
            }
        }
        
        stage('Install dependencies') {
            steps {
                // Install required Python packages
                sh "sudo apt update"
                sh "sudo apt install python3-pip -y"
                sh "sudo pip install fastapi uvicorn"
            }
        }
        
        stage('Deploy') {
            steps {
                // Run FastAPI application using uvicorn
                script {
                    // Start FastAPI app in background
                    def deployCmd = "uvicorn main:app --host 0.0.0.0 --port ${APP_PORT} --reload > /dev/null 2>&1 &"
                    sh deployCmd
                    
                    // Wait for the server to start up
                    sleep 20  // Adjust as needed based on your application's startup time
                    
                    // Verify deployment by making a curl request to the VM
                    def curlCmd = "curl http://${VM_IP}:${APP_PORT}/"
                    sh curlCmd
                }
            }
        }
    }
    
    post {
        always {
            // Clean up or post-processing steps if needed
            // For example, stopping the server after tests
            sh "killall uvicorn"
        }
    }
}
```

## Jenkins Job Setup

1. Create a new Jenkins pipeline job.
2. Paste the above pipeline script into the job configuration.
3. Replace your_vm_ip_address with the actual IP address where your Jenkins server is running.
4. Save the job configuration.

## Run Jenkins Job

Trigger the Jenkins job to execute the pipeline. Jenkins will:

-  Checkout the code from this repository.

-  Install dependencies (Python packages).

-  Deploy the FastAPI application.

-  Verify deployment by making a request to the specified IP and port.

-  Clean up after the job completes.
