pipeline {
    agent any

    environment {
        // The stable image tag built in Question 3
        STABLE_IMAGE = 'monashree1511/metro:v1'
        // A deliberately faulty image tag that does not exist on Docker Hub
        FAULTY_IMAGE = 'monashree1511/metro:v2-broken'
        
        KUBE_CONFIG_CREDS = 'kubernetes-config'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com'
            }
        }

        stage('Deploy Faulty Version Rollout') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDS}", variable: 'KUBECONFIG')]) {
                        echo "--- Simulating Faulty Deployment Release ---"
                        
                        // Forcefully update the container image to a non-existent tag
                        sh "kubectl --kubeconfig=${KUBECONFIG} set image deployment/traffic-system-deployment traffic-dashboard=${FAULTY_IMAGE}"
                        
                        echo "Monitoring rollout status (Timeout set to 60s)..."
                        // This command will fail because Kubernetes cannot pull the non-existent v2-broken image
                        sh "kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/traffic-system-deployment --timeout=60s"
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDS}", variable: 'KUBECONFIG')]) {
                    echo "🚨 CRITICAL: Rollout failed! Identifying failed deployment..."
                    sh "kubectl --kubeconfig=${KUBECONFIG} get pods"
                    
                    echo "🔄 AUTOMATIC RECOVERY: Executing rollback to the previous stable release..."
                    sh "kubectl --kubeconfig=${KUBECONFIG} rollout undo deployment/traffic-system-deployment"
                    
                    echo "Checking restored application status..."
                    sh "kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/traffic-system-deployment"
                }
            }
        }
        success {
            echo "Deployment was successful."
        }
    }
}
