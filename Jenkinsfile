pipeline {
    agent any
    environment {
        DOCKER_IMAGE     = 'bhanutejaravutla/simple-app'
        // FIX: Provide the complete path directly to your target manifests repo
        SONAR_ORG        = 'tejaravutla287'
        SONAR_PROJ       = 'tejaravutla287'
    }

    stages {
        stage('Maven Compile & Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        
                stage('SonarCloud Scan Validation') {
            steps {
                withSonarQubeEnv('SonarCloud') { 
                    withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                        script {
                            env.MAVEN_OPTS = "-Xms256m -Xmx512m -XX:MaxMetaspaceSize=128m"
                        }
                        sh """
                            mvn sonar:sonar \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.token=${SONAR_TOKEN} \
                            -Dsonar.organization=${SONAR_ORG} \
                            -Dsonar.projectKey=${SONAR_PROJ} \
                            -Dsonar.workers=1 \
                            -Dsonar.scanAllFiles=false \
                            -Dsonar.language=java \
                            -Dsonar.java.binaries=target/classes
                        """
                    }
                }
            }
        }

        
        stage('Verify Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if ( qg.status != 'NONE') { 
                            error "Pipeline stopped: Quality Gate failed: ${qg.status}" 
                        } else {
                            echo "Quality Gate check cleared with status: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Execute Unit Tests') {
            steps { sh 'mvn test' }
        }
        
        stage('Package & Push Container') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
        
                    sh """
                    echo "Docker Image: ${DOCKER_IMAGE}"
                    echo "Build Number: ${BUILD_NUMBER}"
        
                    echo "\$DOCKER_PASSWORD" | docker login -u "\$DOCKER_USERNAME" --password-stdin
        
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
        
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
        
                    docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                    """
                }
            }
        }

        stage('Manifest GitOps Delivery Loop') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'git-pat', variable: 'GIT_TOKEN')]) {
                        
                        sh '''
                        git config --global user.email "jenkins-bot@poc.com"
                        git config --global user.name "Jenkins GitOps Engine"
                    
                        rm -rf target-manifests
                    
                        git clone https://${GIT_TOKEN}@github.com/tejaravutla287/gitops-repo.git target-manifests
                    
                        cd target-manifests
                    
                        sed -i "s|image: .*|image: bhanutejaravutla/simple-app:${BUILD_NUMBER}|" deployment.yaml
                    
                        git add .
                        git commit -m "Update image to build ${BUILD_NUMBER}"
                        git push
                        '''
                    }
                }
            }
        }
    }
}
