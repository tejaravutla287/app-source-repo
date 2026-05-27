pipeline {
    agent any
    environment {
        DOCKER_IMAGE     = 'YOUR_DOCKERHUB_USERNAME/simple-app'
        MANIFEST_REPO    = '://github.com'
        SONAR_ORG        = 'YOUR_SONARCLOUD_ORGANIZATION_KEY'
        SONAR_PROJ       = 'YOUR_SONARCLOUD_PROJECT_KEY'
    }
    stages {
        stage('Maven Compile & Build') {
            steps { sh 'mvn clean package -DskipTests' }
        }
        stage('SonarCloud Scan Validation') {
            steps {
                withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                    sh "mvn sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.token=${SONAR_TOKEN} -Dsonar.organization=${SONAR_ORG} -Dsonar.projectKey=${SONAR_PROJ}"
                }
            }
        }
        stage('Verify Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') { error "Pipeline stopped: Quality Gate failed: ${qg.status}" }
                    }
                }
            }
        }
        stage('Execute Unit Tests') {
            steps { sh 'mvn test' }
        }
        stage('Package & Push Container') {
            steps {
                script {
                    docker.withRegistry('https://docker.io', 'docker-hub-creds') {
                        def appImage = docker.build("${DOCKER_IMAGE}:${BUILD_NUMBER}")
                        appImage.push()
                    }
                }
            }
        }
        stage('Manifest GitOps Delivery Loop') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'git-creds', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh '''
                            git config --global user.email "jenkins-bot@poc.com"
                            git config --global user.name "Jenkins GitOps Engine"
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@${MANIFEST_REPO} target-manifests
                            cd target-manifests
                            sed -i "s|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g" deployment.yaml
                            git add deployment.yaml
                            git commit -m "chore: auto-bump tag to release-${BUILD_NUMBER} [skip ci]"
                            git push origin main
                        '''
                    }
                }
            }
        }
    }
}
