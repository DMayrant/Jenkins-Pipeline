pipeline {
    agent any

    stages {
        stage ('Install kubectl') {
            steps {
                sh '''
                echo 'Installing kubectl'

                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                export PATH=$PATH:$PWD
                ./kubectl version --client
                '''
            }
        }
        stage ('Install SNYK Scan') {
            steps {
                sh '''
                echo 'Installing SNYK scan'

                curl --compressed https://downloads.snyk.io/cli/stable/snyk-linux -o snyk
                
                chmod +x snyk

                ./snyk --version
                '''
            }
        }
        stage ('Install kubescape') {
            steps {
                sh '''
                set -e
                
                echo 'Installing kubescape'

                curl -L https://github.com/kubescape/kubescape/releases/latest/download/kubescape-linux-amd64 -o kubescape

                chmod +x kubescape
                
                ./kubescape version
                '''
            }
        }
        stage ('Docker Scout scan') {
            steps { 
                sh '''
                echo 'Scanning images for vulnerabilities 🔍'
                docker pull nginx:1.29.0
                docker scout cves nginx:1.29.0
                '''
            }
        }
        stage ('SNYK scan') {
            steps {
                sh '''
                docker pull nginx:1.29.0
                ./snyk container test nginx:1.29.0 || true
                '''
            }
        }
        stage ('kubernetes service account') {
            steps {
                sh '''
                echo 'Creating service account 📝'
               
                ./kubectl create sa jenkins --dry-run=client -o yaml > jenkins-sa.yaml
                ./kubectl apply -f jenkins-sa.yaml
                '''
            }
        }
        stage ('testing') {
            steps {
                sh '''
                echo 'Generating YAML manifests 🗃️'
               
                ./kubectl create deployment nginx-deploy --image=nginx:1.29.0 --port=80 --replicas=3 --dry-run=client -o yaml > nginx-deploy.yaml
                ./kubectl run curl --image=curlimages/curl --restart=Never --dry-run=client -o yaml > curl.yaml 
                ./kubectl expose deploy nginx-deploy --port=80 --target-port=80 --type=NodePort --dry-run=client -o yaml > nginx-svc.yaml
                '''
            }
        }
        stage ('Apply') {
            steps {
                sh '''
                echo 'Applying workloads 🧰'
                
                ./kubectl apply -f nginx-deploy.yaml
                ./kubectl apply -f curl.yaml
                ./kubectl apply -f nginx-svc.yaml
                ''' 
            }
        }
        stage ('Logs') {
            steps {
                sh '''
                echo 'Checking logs 📊'
                
                ./kubectl logs deployment/nginx-deploy --tail=10 > nginx-logs.log
                ./kubectl wait --for=condition=Ready pod -l run=curl --timeout=60s
                ./kubectl logs -l run=curl --tail=10
                ./kubectl get events --sort-by=.lastTimestamp | tail -n 15
                '''
            }
        }
        stage ('deployment health check') {
            steps {
                sh '''
                echo 'Checking deployment status 🧪'
                
                ./kubectl rollout status deployment/nginx-deploy
                ./kubectl rollout history deployment/nginx-deploy
                '''
            }
        }
        stage ('Rollback') {
            steps {
                sh '''
                ./kubectl rollout undo deployment/nginx-deploy
                ./kubectl describe deployment nginx-deploy
                '''
            }
        }
        stage ('internal service discovery') {
            steps {
                sh '''
                ./kubectl exec $(./kubectl get pod -l run=curl -o jsonpath="{.items[0].metadata.name}") -- curl -s nginx-deploy:80
                '''
            }
        }
        stage ('port-forward') {
            steps {
                sh '''
                echo "Port-forwarding 🌎"
                
                ./kubectl port-forward svc/nginx-deploy 3000:80 & 
                sleep 5 
                curl http://localhost:3000
                '''
            }
        }
        stage ('Security Posture') {
            steps {
                sh '''
                echo 'Scanning kubernetes cluster'
                
                ./kubescape scan
                '''
            }

        }
        stage {'Cleanup 🧹'} {
            steps {
                sh '''
                ./kubectl delete -f nginx-deploy.yaml --ignore-not-found=true
                ./kubectl delete -f curl.yaml --ignore-not-found=true
                ./kubectl delete -f nginx-svc.yaml --ignore-not-found=true
                ./kubectl delete -f jenkins-sa.yaml --ignore-not-found=true

                ./kubectl get all
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
