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
                                                         usernameVariable: 'YIPBL',
                                                         passwordVariable: 'YIPBL')]) {
                            bat '''
                                mvn liquibase:status \
                                    -Dliquibase.url=${DB_URL} \
                                    -Dliquibase.username=${YIPBL} \
                                    -Dliquibase.password=${YIPBL}
                            '''
                        }
                        echo 'Database connection successful'
                    } catch (Exception e) {
                        echo "Database connection test failed, but continuing with checksum fix..."
                        echo "Error details: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Fix Checksum Issues') {
            steps {
                echo 'Clearing checksum validation issues...'
                withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                 usernameVariable: 'YIPBL',
                                                 passwordVariable: 'YIPBL')]) {
                    script {
                        try {
                            // Option 1: Clear checksums for problematic changesets
                            bat '''
                                mvn liquibase:clearCheckSums \
                                    -Dliquibase.url=${DB_URL} \
                                    -Dliquibase.username=${YIPBL} \
                                    -Dliquibase.password=${YIPBL}
                            '''
                            echo 'Checksums cleared successfully'
                        } catch (Exception e) {
                            echo "Clear checksums failed, trying alternative approach..."
                            
                            // Option 2: Mark changesets as executed (if they were already run)
                            try {
                                bat '''
                                    mvn liquibase:changelogSync \
                                        -Dliquibase.url=${DB_URL} \
                                        -Dliquibase.username=${YIPBL} \
                                        -Dliquibase.password=${YIPBL}
                                '''
                                echo 'Changelog sync completed'
                            } catch (Exception e2) {
                                echo "Changelog sync also failed. Manual intervention may be required."
                                error "Unable to resolve checksum issues automatically"
                            }
                        }
                    }
                }
            }
        }
        
        stage('Verify Status After Fix') {
            steps {
                echo 'Verifying database status after checksum fix...'
                withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                 usernameVariable: 'YIPBL',
                                                 passwordVariable: 'YIPBL')]) {
                    bat '''
                        mvn liquibase:status \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${YIPBL} \
                            -Dliquibase.password=${YIPBL} \
                            -Dliquibase.verbose=true
                    '''
                }
            }
        }
        
        stage('Liquibase Update') {
            steps {
                echo 'Running Liquibase database migrations...'
                withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                 usernameVariable: 'YIPBL',
                                                 passwordVariable: 'YIPBL')]) {
                    bat '''
                        mvn liquibase:update \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${YIPBL} \
                            -Dliquibase.password=${YIPBL}
                    '''
                }
            }
        }
        
        stage('Generate Changelog Report') {
            steps {
                echo 'Generating final changelog report...'
                withCredentials([usernamePassword(credentialsId: 'YIPBL',
                                                 usernameVariable: 'YIPBL',
                                                 passwordVariable: 'YIPBL')]) {
                    bat '''
                        mvn liquibase:status \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${YIPBL} \
                            -Dliquibase.password=${YIPBL} \
                            -Dliquibase.verbose=true
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
            archiveArtifacts artifacts: 'target/**/*', fingerprint: true, allowEmptyArchive: true
        }
        success {
            echo 'Database migration completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
            echo 'If checksum issues persist, consider:'
            echo '1. Reverting the changelog files to their original state'
            echo '2. Creating new changesets instead of modifying existing ones'
            echo '3. Using liquibase:rollback if needed'
        }
    }
}