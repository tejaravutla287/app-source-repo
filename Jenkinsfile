pipeline {
    agent any
    environment {
        DOCKER_IMAGE     = 'bhanutejaravutla/simple-app'
        // FIX: Provide the complete path directly to your target manifests repo
        MANIFEST_REPO    = 'github.com'
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
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh """
                        # \$DOCKER_PASSWORD uses a backslash to evaluate safely at runtime without exposing the secret
                        echo "\$DOCKER_PASSWORD" | docker login -u "\$DOCKER_USERNAME" --password-stdin
                        
                        # Build and push using the dynamic Jenkins environment tags
                        docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        
                        # Clean up local images to preserve EC2 disk space
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
                            
                            # Wipe old checked out data 
                            rm -rf target-manifests
                            
                            # FIX: Hardcode the domain structure to bypass any environment cache leaks
                            git clone https://\$GIT_USERNAME:\$GIT_PASSWORD@://github.com target-manifests
                            
                            cd target-manifests
                            sed -i "s|image: bhanutejaravutla/simple-app:.*|image: bhanutejaravutla/simple-app:${BUILD_NUMBER}|g" deployment.yaml
                            
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
