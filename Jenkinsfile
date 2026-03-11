pipeline {
    agent any
    environment {
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        APP_NAME = 'learnJenkinsApp'
        AWS_DEFAULT_REGION = 'ap-southeast-1'
        AWS_ECS_CLUSTER = 'LearnJenkinsApp-Cluster-Prod'
        AWS_ECS_SERVICE = 'LearnJenkinsApp-Service-Prod'
        AWS_ECS_TASK_DEFINTION = 'LearnJenkinsApp-TaskDefinition-Prod'
        AWS_ECR_ENDPOINT = 'XXXX.XXXX.XXX.amazonaws.com'
    }
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
                    echo "Test stage"
                    ls build/index.html
                    npm run test
                    serve -s build --listen 3000 & 
                    sleep 10
                    npx playwright test --reporter=html --list    
                '''
            }

            post {
                always {
                    junit 'test-results/junit.xml'
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
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

        stage('aws s3') {
            agent {
                docker {
                    image 'myawscli'
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
                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''
                }
            }
        }

        stage('aws ecr') {
            agent {
                docker {
                    image 'myawscli'
                    reuseNode true
                    args '-u root -v /var/run/docker.dock:/var/run/docker.sock --entrypoint=""'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'amzn4814_access', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                sh'''
                    docker build -t $AWS_ECR_ENDPOINT/$APP_NAME:$REACT_APP_VERSION
                    aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_ECR_ENDPOINT
                    docker push $AWS_ECR_ENDPOINT/$APP_NAME:$REACT_APP_VERSION
                '''
                }
            }
        }

        stage('aws ecs') {
            agent {
                docker {
                    image 'amyawscli'
                    reuseNode true
                    args '-u root --entrypoint=""'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'amzn4814_access', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh'''
                        aws --version
                        sed -i "s/#IMAGE_NAME#/$REACT_APP_VERSION/g" aws\task-defintion-prod.json
                        LATEST_TASK_DEF_REVISION=$(aws ecs register-task-definition --cli-input-json file://aws/task-defintion-prod.json > jq '.taskDefinition.revision')
                        aws ecs update-service --cluster $AWS_ECS_CLUSTER --service $AWS_ECS_SERVICE --task-definition $AWS_ECS_TASK_DEFINTION:$LATEST_TASK_DEF_REVISION
                        aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER --services $AWS_ECS_SERVICE
                    '''
                }
            }
        }

        /*
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
        */
    }
}