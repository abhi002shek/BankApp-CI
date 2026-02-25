pipeline {
    agent any
    
    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }
    
    stages {
        stage('Compile') {
            steps {
                sh "mvn clean compile"
            }
        }
        
        stage('Package') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh "docker build -t bankapp:latest ."
            }
        }
    }
}
