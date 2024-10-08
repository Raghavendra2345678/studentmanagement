pipeline {
    agent any

    stages {
        stage('System Info') {
            steps {
                script {
                    echo 'Gathering system information...'

                    // Check Git version and installation path
                    echo 'Checking Git version and installation path...'
                    sh '''
                        git --version
                        which git
                    '''

                    // Check Maven version and installation path
                    echo 'Checking Maven version and installation path...'
                    sh '''
                        ./mvnw -v
                        which mvn || echo "Maven wrapper (mvnw) used."
                    '''

                    // Check Java version and installation path
                    echo 'Checking Java version and installation path...'
                    sh '''
                        java -version
                        which java
                    '''

                    // Check Docker version and installation path
                    echo 'Checking Docker version and installation path...'
                    sh '''
                        docker --version
                        which docker
                    '''

                    // Check Docker Compose version and installation path
                    echo 'Checking Docker Compose version and installation path...'
                    sh '''
                        docker compose version
                        which docker-compose || which docker || echo "Docker Compose included in Docker CLI."
                    '''
                }
            }
        }

        stage('Environment Info') {
            steps {
                script {
                    echo 'Printing environment variables...'
                    sh '''
                        echo "JENKINS_HOME: $JENKINS_HOME"
                        echo "WORKSPACE: $WORKSPACE"
                        echo "BUILD_ID: $BUILD_ID"
                        echo "BUILD_NUMBER: $BUILD_NUMBER"
                        echo "NODE_NAME: $NODE_NAME"
                        echo "EXECUTOR_NUMBER: $EXECUTOR_NUMBER"
                        echo "JOB_NAME: $JOB_NAME"
                        echo "GIT_COMMIT: $GIT_COMMIT"
                    '''
                }
            }
        }

        stage('Checkout') {
            steps {
                script {
                    echo 'Checking out the code from GitHub...'
                }
                git branch: 'main', credentialsId: 'github-login', url: 'https://github.com/Ravikumar-Pawar/studentmanagement.git'
            }
        }




stage('SonarQube Analysis') {
    steps {
        script {
            echo 'Running SonarQube analysis...'
        }
        withSonarQubeEnv('sonarqube') {
            withCredentials([usernamePassword(credentialsId: 'sonar-login', passwordVariable: 'SONAR_PASSWORD', usernameVariable: 'SONAR_USERNAME')]) {
                sh '''./mvnw sonar:sonar \
                -Dsonar.login=$SONAR_USERNAME \
                -Dsonar.password=$SONAR_PASSWORD \
                -Dsonar.projectKey=studentmanagement \
                -Dsonar.projectName="Student Management System" \
                -Dsonar.projectVersion=1.0 \
                -Dsonar.sources=src/main/java \
                -Dsonar.tests=src/test/java \
                -Dsonar.jacoco.reportPaths=target/jacoco.exec \
                -Dsonar.sourceEncoding=UTF-8 \
                -Dsonar.java.binaries=target/classes \
                -Dsonar.junit.reportPaths=target/surefire-reports
                '''
            }
        }
    }
}



        stage('Build') {
            steps {
                script {
                    echo 'Building the project...'
                }
                sh './mvnw clean install'
            }
        }

        stage('Artifactory Push') {
            steps {
                script {
                    echo 'Pushing build artifacts to Artifactory...'
                }
                withCredentials([usernamePassword(credentialsId: 'artifactory-access', passwordVariable: 'ARTIFACTORY_PASSWORD', usernameVariable: 'ARTIFACTORY_USERNAME')]) {
                    sh '''
                        curl -u $ARTIFACTORY_USERNAME:$ARTIFACTORY_PASSWORD -T target/studentmanagement-*.jar "http://localhost:8081/artifactory/example-repo-local/studentmanagement/studentmanagement-1.0.jar"
                    '''
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                }
                sh 'docker build -t studentmanagement:latest .'
            }
        }

        stage('Push Docker Image to Hub') {
            steps {
                script {
                    echo 'Pushing Docker image to Docker Hub...'

                    // Log in to Docker Hub
                    withCredentials([usernamePassword(credentialsId: 'docker-login', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
                    }

                    // Tag and push the image
                    sh 'docker tag studentmanagement:latest ravikumarpawar/studentmanagement:latest'
                    sh 'docker push ravikumarpawar/studentmanagement:latest'
                }
            }
        }

        stage('Deploy with Docker Swarm') {
            steps {
                script {
                    echo 'Deploying with Docker Swarm...'

                    // Define Docker Compose file path
                    def composeFile = 'docker-compose.yml'

                    // Remove existing stack services
                    echo 'Removing existing stack services...'
                    sh '''
                        docker stack rm studentmanagement || true
                        sleep 10  # Allow time for removal
                    '''

                    // Deploy the stack
                    echo 'Deploying stack with Docker Compose...'
                    sh "docker stack deploy -c ${composeFile} studentmanagement"

                    // Confirm the application is running
                    echo 'Confirming the application is running...'
                    sh '''
                        sleep 30  # Give some time for the containers to start
                        echo "Checking Docker stack status..."
                        docker stack ps studentmanagement
                        docker service ls
                        # Check if all services are in 'Running' state (case-insensitive)
                        if docker stack ps studentmanagement | grep -i "running" > /dev/null; then
                            echo "Docker stack is running."
                        else
                            echo "Docker stack is not running."
                            exit 1
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            echo 'Check if the application is accessible at http://127.0.0.1:8085/students'
        }
        failure {
            echo 'Pipeline failed.'
            sh '''
                echo "Execute the following commands to troubleshoot Swarm-related issues:"
                echo "docker swarm leave --force"
                echo "docker swarm init"
            '''

        }
    }
}
