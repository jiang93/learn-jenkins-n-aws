pipeline {
    agent any
    stages {
        stage('build') {
            agent {
                docker {
                    image 'node:20-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version 
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('test') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true 
                    args '-p 3000:3000'
                }
            }
            steps {
                sh '''
                    echo "Testing stage..."
                    ls build/index.html
                    npm run test
                    serve -s build --listen 3000 & 
                    sleep 10
                    npx playwright test --reporter=html --list   

                '''
            }
            post {
                always {
                    junit 'jest-results/junit.xml'
                }
            }
        }

        stage('deploy staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true 
                }
            }
            steps {
                sh '''
                    echo "Deploying to staging site..."           
                '''

            }
        }

        stage('approval') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('aws') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'
                    reuseNode true
                    args '--entrypoint=""'
                }
            }
            environment {
                AWS_S3_BUCKET = 'amzn4184-jenkins-s3'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'amzn4814_access', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh'''
                        aws --version
                        aws s3 sync . s3://$AWS_S3_BUCKET
                    '''
                }
            }
        }

        stage('deploy prod') {
            agent {
                    docker {
                        image 'my-playwright'
                        reuseNode true 
                    }
            }
            steps {
                sh '''
                    echo "Deploying to production site..."                
                '''
            }
        }
    }
}