pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'JDK17'
    }

    environment {
        SONAR_TOKEN = credentials('sonar-token')
        IMAGE_NAME = "myapp"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/SamarpitaPattnaik/Boardgame.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 export PATH=$JAVA_HOME/bin:$PATH'
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    mvn sonar:sonar \
                    -Dsonar.projectKey=myapp \
                    -Dsonar.login=${SONAR_TOKEN} \
                    -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }
        
        stage('OWASP Dependency Check') {
    steps {
        sh '''
        /opt/dependency-check/bin/dependency-check.sh \
        --project myapp \
        --scan . \
        --format HTML \
        --out dependency-check-report \
        --data /var/lib/jenkins/dependency-check-data \
        --noupdate \
        --disableOssIndex
        '''
    }
}

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }
        stage('Trivy Scan') {
            steps {
                sh '''
                trivy image \
                --exit-code 0 \
                --severity HIGH,CRITICAL \
                ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                docker stop myapp || true
                docker rm myapp || true
                docker run -d \
                --name myapp \
                -p 8081:8080 \
                ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'dependency-check-report/**/*.*', allowEmptyArchive: true
             publishHTML([
            reportDir: 'dependency-check-report',
            reportFiles: 'index.html',
            reportName: 'OWASP Dependency Check Report',
            keepAll: true,
            alwaysLinkToLastBuild: true,
            allowMissing: true
        ])
        }
        success {
            echo 'Pipeline completed successfully! 🚀'
        }
        failure {
            echo 'Pipeline failed. Check logs.'
        }
    }
}
