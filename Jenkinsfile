pipeline {
    agent any
    environment {
        DOCKER_IMAGE     = 'bhanutejaravutla/simple-app'
        // FIX: Ensure this points directly to your actual GitHub Manifest Repository path
        MANIFEST_REPO    = '://github.com'
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
                        if (qg.status != 'OK' && qg.status != 'NONE') { 
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
                // Use standard environment variable injection instead of the complex plugin wrapper
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh """
                        # 1. Log straight into DockerHub using the terminal CLI
                        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
                        
                        # 2. Build the container image using the native Buildx engine
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                        
                        # 3. Push the finalized container layers to the registry
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        
                        # 4. Clean up local untagged image cache to preserve EC2 disk space
                        docker rmi ${DOCKER_IMAGE}:${BUILD_NUMBER} || true
                    """
                }
            }
        }

        
        stage('Manifest GitOps Delivery Loop') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'git-creds', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git config --global user.email "jenkins-bot@poc.com"
                            git config --global user.name "Jenkins GitOps Engine"
                            rm -rf target-manifests
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${MANIFEST_REPO} target-manifests
                            cd target-manifests
                            sed -i "s|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment.yaml
                            git add deployment.yaml
                            git commit -m "chore: auto-bump tag to release-${BUILD_NUMBER} [skip ci]" || echo "No changes to commit"
                            git push origin main
                        """
                    }
                }
            }
        }
    }
}
