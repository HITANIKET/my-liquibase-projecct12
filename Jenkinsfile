pipeline {
    agent any
    
    environment {
        DB_URL = 'jdbc:oracle:thin:@45.114.142.57:1525/YIPDV'
        MAVEN_HOME = tool 'Maven'
        PATH = "${MAVEN_HOME}/bin:${PATH}"
    }
    
    tools {
        maven 'Maven-3.9.9'
        jdk 'JDKHITANIKET'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Validate') {
            steps {
                echo 'Validating project structure...'
                bat 'mvn validate'
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling the project...'
                bat 'mvn clean compile'
            }
        }
        
        stage('Test Connection') {
            steps {
                echo 'Testing database connection...'
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                         usernameVariable: 'DB_USERNAME',
                                                         passwordVariable: 'DB_PASSWORD')]) {
                            bat '''
                                mvn liquibase:status \
                                    -Dliquibase.url=${DB_URL} \
                                    -Dliquibase.username=${DB_USERNAME} \
                                    -Dliquibase.password=${DB_PASSWORD}
                            '''
                        }
                        echo 'Database connection successful'
                    } catch (Exception e) {
                        error "Database connection failed: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Liquibase Update') {
            steps {
                echo 'Running Liquibase database migrations...'
                withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    bat '''
                        mvn liquibase:update \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${DB_USERNAME} \
                            -Dliquibase.password=${DB_PASSWORD}
                    '''
                }
            }
        }
        
        stage('Generate Changelog Report') {
            steps {
                echo 'Generating changelog report...'
                withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    bat '''
                        mvn liquibase:status \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${DB_USERNAME} \
                            -Dliquibase.password=${DB_PASSWORD} \
                            -Dliquibase.verbose=true
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
            archiveArtifacts artifacts: 'target/**/*', fingerprint: true
        }
        success {
            echo 'Database migration completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}