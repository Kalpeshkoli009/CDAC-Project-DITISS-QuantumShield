pipeline {
    agent {
        label 'jenkins_agent'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        VENV = "venv"
        IMAGE_REPO = "flask-app"
        SONAR_HOME = tool "sonar"
        IMAGE_TAG = "${BUILD_NUMBER}"
        
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage ('SonarQube Analysis'){
            steps {
                withSonarQubeEnv('sonarqube-server'){
                    sh '''
                      $SONAR_HOME/bin/sonar-scanner \
                      -Dsonar.projectKey=flask-app \
                      -Dsonar.sources=.
                      
                      '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Setup Python Environment') {
            steps {
                ansiColor('xterm') {
                    sh '''
                        python3 -m venv $VENV
                        . $VENV/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                    '''
                }
            }
        }

        stage('Lint Code') {
            steps {
                ansiColor('xterm') {
                    sh '''
                        . $VENV/bin/activate
                        flake8 app.py || true
                    '''
                }
            }
        }

        stage('Run Tests') {
            steps {
                ansiColor('xterm') {
                    sh '''
                        . $VENV/bin/activate
                        pytest || true
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                ansiColor('xterm') {
                    sh '''
                        . $VENV/bin/activate
                        python app.py &
                        sleep 5
                        curl -f http://127.0.0.1:5000/health
                    '''
                }
            }
        }

        /* ---------- DOCKER STAGES (ADDED) ---------- */

        stage('Build Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    ansiColor('xterm') {
                        script {
                            env.IMAGE_NAME = "${DOCKER_USER}/${IMAGE_REPO}"
                            env.IMAGE_TAG = "${BUILD_NUMBER}"
                        }
                        
                        sh '''
                            docker build -t $IMAGE_NAME:$IMAGE_TAG .
                            docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
                        '''
                    }
                }
            }
        }


        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "======================================"
                    echo " Trivy Image Security Scan"
                    echo " Image: $IMAGE_NAME:$IMAGE_TAG"
                    echo "======================================"

                    # Download Trivy HTML template if not present
                    if [ ! -f html.tpl ]; then
                    curl -sSL \
                        https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl \
                        -o html.tpl
                    fi

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format template \
                        --template "@html.tpl" \
                        --output trivy-report.html \
                        --no-progress \
                        $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }


        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    ansiColor('xterm') {
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                ansiColor('xterm') {
                    sh '''
                        docker push $IMAGE_NAME:$IMAGE_TAG
                        docker push $IMAGE_NAME:latest
                        
                    '''
                }
            }
        }

   
        stage('Commit K8s Manifests') {
            
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-private-token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                    git fetch origin

                    git checkout deploy
                    git pull origin deploy

                    sed -i "s|image: .*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|" k8s-manifest/deployment.yaml

                    git config user.email "jenkins@ci.local"
                    git config user.name "jenkins"

                    git add k8s-manifest/deployment.yaml
                    git commit -m "update image to ${BUILD_NUMBER}" || echo "No changes"

                    git push https://${GIT_USER}:${GIT_PASS}@github.com/JatinDeshmukh009/DevSecOps-Projec-DITISS-25.git deploy
                    '''
                }
            }
        }
        
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.html', fingerprint: true
            sh 'pkill -f app.py || true'
            cleanWs()
        }
        success {
            echo "✅ Pipeline completed successfully"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
