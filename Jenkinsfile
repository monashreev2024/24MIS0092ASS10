pipeline {
    agent any

    environment {
        // The working image tag deployed in Question 3
        STABLE_IMAGE = 'monashree1511/metro:v1'
        // Deliberately broken tag that doesn't exist on Docker Hub to trigger failure
        FAULTY_IMAGE = 'monashree1511/metro:v2-broken'
        
        KUBE_CONFIG_CREDS = 'kubernetes-config'
    }

    stages {
        stage('Checkout Project Source') {
            steps {
                git branch: 'main', url: 'https://github.com/monashreev2024/24MIS0092ASS10.git'
            }
        }

        stage('Deploy Faulty Version Release') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDS}", variable: 'KUBECONFIG')]) {
                        echo "⚠️ Initiating Faulty Rollout Sequence..."
                        
                        // Force update container to the broken tag
                        sh "kubectl --kubeconfig=${KUBECONFIG} set image deployment/traffic-system-deployment traffic-dashboard=${FAULTY_IMAGE}"
                        
                        echo "Monitoring rollout status (Timeout threshold: 45s)..."
                        // This intentionally fails as Kubernetes hangs indefinitely on ImagePullBackOff
                        sh "kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/traffic-system-deployment --timeout=45s"
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDS}", variable: 'KUBECONFIG')]) {
                    echo "🚨 ROLLOUT FAILURE DETECTED! Pull execution blocked."
                    echo "Current Cluster Pod Status:"
                    sh "kubectl --kubeconfig=${KUBECONFIG} get pods"
                    
                    echo "🔄 AUTOMATED RECOVERY ACTIVE: Undoing rollout to restore last stable environment..."
                    sh "kubectl --kubeconfig=${KUBECONFIG} rollout undo deployment/traffic-system-deployment"
                    
                    echo "Verifying application availability..."
                    sh "kubectl --kubeconfig=${KUBECONFIG} rollout status deployment/traffic-system-deployment"
                }
            }
        }
    }
}

