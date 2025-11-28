// Needed so the docker image handle `app` is visible across stages
def app

pipeline {
  agent any

  tools {
    maven 'Maven_3.8.4'
  }

  stages {

    stage('Compile and Run Sonar Analysis') {
      steps {
        // Use Jenkins credentials instead of hardcoding the token
        withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
          sh """
            mvn clean verify sonar:sonar \
              -Dsonar.projectKey=acitbuggywebapp \
              -Dsonar.organization=acitbuggywebapp \
              -Dsonar.host.url=https://sonarcloud.io \
              -Dsonar.token=${SONAR_TOKEN}
          """
        }
      }
    }

    stage('Build') {
      steps {
        withDockerRegistry([credentialsId: 'dockerlogin', url: '']) {
          script {
            // Build local Docker image named "asg"
            app = docker.build("asg")
          }
        }
      }
    }

    stage('Push') {
      steps {
        script {
          // Push to ECR with tag 1.2.0
          docker.withRegistry('https://124355683348.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:aws-credentials') {
            app.push("1.2.0")
          }
        }
      }
    }

    stage('Kubernetes Deployment of ACIT Buggy Web Application') {
      steps {
        withKubeConfig([credentialsId: 'kubelogin']) {
          // Optional: this wipes ALL resources in the namespace – be sure you want this
          sh 'kubectl delete all --all -n devsecops || true'
          sh 'kubectl apply -f deployment.yaml --namespace=devsecops'
        }
      }
    }

    stage('Wait For Testing') {
      steps {
        sh 'pwd; sleep 180; echo "Application has been deployed on K8S"'
      }
    }

    stage('RunDASTUsingZAP') {
      steps {
        withKubeConfig([credentialsId: 'kubelogin']) {
          script {
            sh '''
              set -e

              # Get LoadBalancer hostname for the acitbuggy service
              LB_HOST=$(kubectl get service acitbuggy --namespace=devsecops -o json \
                | jq -r '.status.loadBalancer.ingress[0].hostname')

              echo "Running OWASP ZAP against: http://$LB_HOST"

              # Run OWASP ZAP using Docker image (no need to install zap.sh on the agent)
              docker run --rm \
                -v "$PWD:/zap/wrk" \
                owasp/zap2docker-stable \
                zap.sh -cmd \
                  -quickurl "http://$LB_HOST" \
                  -quickprogress \
                  -quickout "zap_report.html"
            '''

            // Archive the ZAP report
            archiveArtifacts artifacts: 'zap_report.html'
          }
        }
      }
    }
  }
}
