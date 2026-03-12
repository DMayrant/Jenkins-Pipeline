pipeline {
    agent any

    stages {
        stage ('image scan') {
            steps { 
                sh '''
                echo 'Scanning images for vulnerabilities 🔍'
                docker scout cves nginx:1.29.0
                '''
            }
        }
        stage ('kubernetes service account') {
            steps {
                sh '''
                echo 'Creating service account 📝'
                kubectl create sa jenkins --dry-run=client -o yaml > jenkins-sa.yaml
                kubectl apply -f jenkins-sa.yaml
                '''
            }
        }
        stage ('testing') {
            steps {
                sh '''
                echo 'Generating YAML manifests 🗃️'
                kubectl create deploy nginx-deploy --image=nginx:1.29.0 --port=80 --replicas=3 --dry-run=client -o yaml > nginx-deploy.yaml
                kubectl run curl --image=curlimages/curl --dry-run=client -o yaml > curl.yaml 
                kubectl expose deploy nginx-deploy --port=80 --target-port=80 --type=NodePort --dry-run=client -o yaml > nginx-svc.yaml
                '''
            }
        }
        stage ('Apply') {
            steps {
                sh '''
                echo 'Applying workloads 🧰'
                kubectl apply -f nginx-app.yaml
                kubectl apply -f curl.yaml
                kubectl apply -f nginx-svc.yaml
                ''' 
            }
        }
        stage ('Logs') {
            steps {
                sh '''
                echo 'Checking logs 📊'
                kubectl logs deploy/nginx-deploy --tail=10 > nginx-logs.log
                kubectl logs pod/curl 
                kubectl get events --sort-by=.lastTimestamp | tail -n 15
                '''
            }
        }
        stage ('deployment health check') {
            steps {
                sh '''
                echo 'Checking deployment status 🧪'
                kubectl rollout status deployment/nginx-deploy
                kubectl rollout history deployment/nginx-deploy
                '''
            }
        }
        stage ('internal service discovery') {
            steps {
                sh '''
                kubectl exec curl -- curl http://nginx-deploy.default.svc.cluster.local:80
                '''
            }
        }
        stage ('port-forward') {
            steps {
                sh '''
                echo "Port-forwarding 🌎"
                kubectl port-forward svc/nginx-deploy 3000:80 & 
                sleep 5 
                curl http://localhost:3000
                '''
            }
        }
        stage ('Security Posture') {
            steps {
                sh '''
                echo 'Scanning kubernetes cluster'
                kubescape scan
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