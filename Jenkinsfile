pipeline {
    agent any
    
    tools { 
        maven 'Maven_3.8.4'  
    }
    
    environment {
        SONAR_TOKEN = credentials('sonarcloud-token')
        ECR_REGISTRY = '124355683348.dkr.ecr.us-east-2.amazonaws.com'
        IMAGE_NAME = 'asg'
        IMAGE_TAG = "1.2.0-${BUILD_NUMBER}"
    }
    
    stages {
        stage('CompileandRunSonarAnalysis') {
            steps {	
                sh '''
                    mvn clean verify sonar:sonar \
                    -Dsonar.projectKey=acitbuggywebapp \
                    -Dsonar.organization=acitbuggywebapp \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.token=${SONAR_TOKEN}
                '''
            }
        }
        
        stage('Build Docker Image') { 
            steps { 
                script {
                    app = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REGISTRY}", 'ecr:us-east-2:aws-credentials') {
                        app.push("${IMAGE_TAG}")
                        app.push("latest")
                    }
                }
            }
        }
        
        stage('Update Deployment Manifest') {
            steps {
                script {
                    sh """
                        sed -i 's|image:.*|image: ${ECR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}|g' deployment.yaml
                    """
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubelogin']) {
                    sh '''
                        kubectl create namespace devsecops --dry-run=client -o yaml | kubectl apply -f -
                        kubectl apply -f deployment.yaml --namespace=devsecops
                        kubectl rollout status deployment/acitbuggy-deployment -n devsecops --timeout=5m
                    '''
                }
            }
        }
        
        stage('Wait for LoadBalancer') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubelogin']) {
                        timeout(time: 5, unit: 'MINUTES') {
                            sh '''
                                echo "Waiting for LoadBalancer to be ready..."
                                while true; do
                                    LB_HOSTNAME=$(kubectl get svc acitbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
                                    LB_IP=$(kubectl get svc acitbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
                                    
                                    if [ ! -z "$LB_HOSTNAME" ] || [ ! -z "$LB_IP" ]; then
                                        echo "LoadBalancer is ready!"
                                        break
                                    fi
                                    
                                    echo "Still waiting for LoadBalancer..."
                                    sleep 10
                                done
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Verify Application') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubelogin']) {
                        timeout(time: 5, unit: 'MINUTES') {
                            sh '''
                                LB_URL=$(kubectl get svc acitbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                                if [ -z "$LB_URL" ]; then
                                    LB_URL=$(kubectl get svc acitbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                                fi
                                
                                echo "Application URL: http://$LB_URL"
                                
                                # Wait for application to respond
                                for i in {1..30}; do
                                    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://$LB_URL || echo "000")
                                    if [ "$HTTP_CODE" = "200" ] || [ "$HTTP_CODE" = "302" ] || [ "$HTTP_CODE" = "301" ]; then
                                        echo "Application is responding! HTTP Code: $HTTP_CODE"
                                        break
                                    fi
                                    echo "Waiting for application to be ready... Attempt $i/30 (HTTP: $HTTP_CODE)"
                                    sleep 10
                                done
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Run DAST Using ZAP') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubelogin']) {
                        try {
                            sh '''
                                LB_URL=$(kubectl get svc acitbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                                if [ -z "$LB_URL" ]; then
                                    LB_URL=$(kubectl get svc acitbuggy -n devsecops -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                                fi
                                
                                echo "Running ZAP scan on: http://$LB_URL"
                                
                                # Run ZAP using Docker
                                docker run --rm \
                                    -v ${WORKSPACE}:/zap/wrk:rw \
                                    -t ghcr.io/zaproxy/zaproxy:stable \
                                    zap-baseline.py \
                                    -t http://$LB_URL \
                                    -r zap_report.html \
                                    -I
                                
                                echo "ZAP scan completed"
                            '''
                        } catch (Exception e) {
                            echo "ZAP scan failed: ${e.message}"
                            currentBuild.result = 'UNSTABLE'
                        } finally {
                            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true
                            publishHTML([
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: '.',
                                reportFiles: 'zap_report.html',
                                reportName: 'ZAP DAST Report'
                            ])
                        }
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
