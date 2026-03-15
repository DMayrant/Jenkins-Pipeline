pipeline {
    agent any

    stages {
        stage ('Image Pull' ){
            steps {
                sh '''
                docker pull nginx:1.29.0
                '''
            }
        }
        stage ('Trivy Scan 🩻') {
            steps { 
                sh '''
                echo 'Scanning for vulnerabilities...'
                trivy image --severity HIGH,CRITICAL nginx:1.29.0 || true
                '''
            }
        }
        stage ('SNYK scan 🔐') {
            steps {
                sh '''  
                echo 'Executing SNYK scan...'                     
                snyk container test nginx:1.29.0 || true  
                '''
            }
        }
        stage ('Deployment to Kubernetes 🚀') {
            steps {
                sh '''
                set -e

                kubectl create deployment nginx-deploy --image=nginx:1.29.0 --port=80 --replicas=5 --dry-run=client -o yaml > nginx-deploy.yaml
                kubectl apply -f nginx-deploy.yaml 
                kubectl expose deployment nginx-deploy --port=80 --type=ClusterIP --dry-run=client -o yaml > service.yaml
                kubectl apply -f service.yaml
                kubectl describe svc nginx-deploy
                kubectl rollout status deployment nginx-deploy
                kubectl get pods
                '''
            }
        }
        stage ('Security Posture 🩻') {
            steps {
                sh '''
                echo 'Scanning kubernetes cluster'
                
                kubescape scan
                '''
            }
        }
        stage ('OWASP ZAP ⚡️') {
            steps {
                sh '''
                echo 'Starting port-forwarding...'
                kubectl port-forward svc/nginx-deploy 3000:80 &
                PF_PID=$!

                sleep 5

                echo 'Running OWASP ZAP scan'

                docker run --rm \
                -t owasp/zap2docker-stable zap-baseline.py \
                -t http://localhost:3000 \
                -r zap-report.html || true
                kill $PF_PID
                '''
            }
        }
        stage ('Cleanup 🧹') {
            steps {
                sh '''
                echo 'Cleaning up resources'
                kubectl delete -f nginx-deploy.yaml --ignore-not-found=true
                kubectl delete -f service.yaml --ignore-not-found=true
                '''
            }
        }
    }
    post {
        success {
            echo "Pipeline successful! ✅"
        }
        failure {
            echo "Pipeline failed, please check Jenkins logs ⚠️"
        }
    }
}
