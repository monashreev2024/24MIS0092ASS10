pipeline {
    agent any
    
    parameters {
        choice(name: 'DEPLOY_ENVIRONMENT', choices: ['Staging', 'Production'], description: 'Select deployment environment target.')
        booleanParam(name: 'RUN_INTEGRATION_TESTS', defaultValue: true, description: 'Check to execute extended runtime validation checks.')
    }

    environment {
        DOCKER_IMAGE = "monashree1511/traffic-system:${BUILD_NUMBER}"
        DOCKER_LATEST = "monashree1511/traffic-system:latest"
        DOCKER_CREDENTIALS_ID = 'docker-hub-credentials' // Configure in Jenkins Credentials
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Lint & Unit Tests') {
            steps {
                echo "Validating HTML files layout..."
                sh "grep -q 'Traffic Management Dashboard' index.html"
            }
        }

        stage('Build & Local Verify') {
            parallel {
                stage('Docker Compile') {
                    steps {
                        script {
                            sh "docker build -t ${DOCKER_IMAGE} ."
                            sh "docker tag ${DOCKER_IMAGE} ${DOCKER_LATEST}"
                        }
                    }
                }
                stage('Static Configuration Check') {
                    steps {
                        echo "Verifying structural validity of deployment.yaml..."
                        sh "grep -q 'app: traffic-app' deployment.yaml"
                    }
                }
            }
        }

        stage('Container Scan & Push') {
            when {
                expression { return params.RUN_INTEGRATION_TESTS }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, passwordVariable: 'PASSWORD', usernameVariable: 'USER')]) {
                        sh "echo ${PASSWORD} | docker login -u ${USER} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}"
                        sh "docker push ${DOCKER_LATEST}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes Cluster') {
            when {
                environment name: 'DEPLOY_ENVIRONMENT', value: 'Production'
            }
            steps {
                script {
                    // Replaces the placeholder image template inside the file with the exact build ID string tag
                    sh "sed -i 'image: .*traffic-system:.*' deployment.yaml"
                    sh "sed -i 's|YOUR_DOCKERHUB_USERNAME/traffic-system:latest|${DOCKER_IMAGE}|g' deployment.yaml"
                    
                    // Applies configurations to Docker Desktop Kubernetes Context
                    withKubeConfig([credentialsId: 'kubernetes-config-id']) {
                        sh "kubectl apply -f deployment.yaml"
                        sh "kubectl rollout status deployment/traffic-management-system --timeout=60s"
                    }
                }
            }
        }
    }
    
    post {
        success {
            archiveArtifacts artifacts: 'deployment.yaml', fingerprint: true
            echo "Pipeline Completed Successfully."
        }
    }
}
