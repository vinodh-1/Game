pipeline {
    agent any

    tools {
        nodejs 'NodeJS18'
        // Make sure this exists in Global Tool Config
        // Name must match exactly
        sonarQubeScanner 'sonar-scanner'
    }

    environment {
        DOCKER_USER = "vinodh21"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "${BUILD_NUMBER}"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'

        // ✅ FIXED URL
        NEXUS_URL = "http://13.204.93.94:8081/repository/puzzlegame"

        RECIPIENTS = "vinodhmaninadh2001@gmail.com"
        CLUSTER_NAME = "mycluster"
        PROJECT_NAME = "Sliding Puzzle Game"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vinodh-1/Game.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sq') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=game \
                        -Dsonar.sources=src \
                        -Dsonar.projectName=game-App \
                        -Dsonar.projectVersion=${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        // ✅ Optional but recommended
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                if [ -d dist ]; then
                    tar -czf app-${BUILD_NUMBER}.tar.gz dist
                else
                    tar -czf app-${BUILD_NUMBER}.tar.gz .
                fi
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file app-${BUILD_NUMBER}.tar.gz \
                    $NEXUS_URL/app-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .
                docker tag $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG $DOCKER_USER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Docker_CRED',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                    docker push $DOCKER_USER/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                aws eks update-kubeconfig --region ap-south-1 --name $CLUSTER_NAME

                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }

        stage('Install Helm') {
            steps {
                sh '''
                if ! command -v helm &> /dev/null; then
                    curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                    tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                    mv linux-amd64/helm /usr/local/bin/helm
                fi
                '''
            }
        }

        stage('Deploy Monitoring') {
            steps {
                sh '''
                helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                helm repo update

                helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace
                '''
            }
        }

        stage('Expose Grafana & Prometheus') {
            steps {
                sh '''
                sleep 30

                kubectl patch svc monitoring-grafana -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'

                kubectl patch svc monitoring-kube-prometheus-prometheus -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'
                '''
            }
        }
    }

    post {

        success {
            script {

                sleep 40

                def APP_URL = sh(script: "kubectl get svc puzzle-game-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true", returnStdout: true).trim()
                def GRAFANA_URL = sh(script: "kubectl get svc monitoring-grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true", returnStdout: true).trim()
                def PROM_URL = sh(script: "kubectl get svc monitoring-kube-prometheus-prometheus -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true", returnStdout: true).trim()

                def DOCKER_IMAGE = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"

                emailext(
                    subject: "🚀 Deployment Successful - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                    <h2>Deployment Successful</h2>
                    <p><b>Project:</b> ${PROJECT_NAME}</p>
                    <p><b>Docker:</b> ${DOCKER_IMAGE}</p>

                    <p><a href="http://${APP_URL}">Application</a></p>
                    <p><a href="http://${GRAFANA_URL}">Grafana</a></p>
                    <p><a href="http://${PROM_URL}:9090">Prometheus</a></p>

                    <p><a href="${env.BUILD_URL}">Jenkins Build</a></p>
                    """,
                    to: "${env.RECIPIENTS}"
                )
            }
        }

        failure {
            emailext(
                subject: "❌ Deployment Failed - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                <h2>Deployment Failed</h2>
                <p><a href="${env.BUILD_URL}">Check Logs</a></p>
                """,
                to: "${env.RECIPIENTS}"
            )
        }

        always {
            archiveArtifacts artifacts: 'app-*.tar.gz', fingerprint: true, allowEmptyArchive: true
        }
    }
}
