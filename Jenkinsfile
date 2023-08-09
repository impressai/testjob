pipeline {
    agent any
    
    stages {
        stage('Setup') {
            steps {
                script {
                    // Stash any changes in the working directory
                    gitStash()
                    
                    // Configure Git user
                    sh 'git config user.name "vaisaghvt"'
                    sh 'git config user.email "vaisagh@impress.ai"'
                    
                    // Create and populate .env file
                    sh 'touch .env'
                    sh 'echo DEBUG=1 >> .env'
                    sh 'echo HTTPS=0 >> .env'
                    sh 'echo SES_REGION=us-east-1 >> .env'
                    sh 'echo SES_ACCESS_KEY_ID=$SES_ACCESS_KEY_ID >> .env'
                    sh 'echo SES_SECRET_ACCESS_KEY=$SES_SECRET_ACCESS_KEY >> .env'
                    sh 'echo TWILIO_ACCOUNT_ID=a >> .env'
                    sh 'echo TWILIO_AUTH=9 >> .env'
                    sh 'echo TWILIO_FROM=+1234567 >> .env'
                    sh 'echo WEBPACK_DEBUG_V1=1 >> .env'
                    sh 'echo WEBPACK_DEBUG_V2=0 >> .env'
                }
            }
        }
        
        stage('Docker Setup') {
            steps {
                sh 'sudo yum install -y docker aws && sudo systemctl start docker'
                sh 'sudo usermod -a -G docker ec2-user'
                sh 'sudo chmod 666 /var/run/docker.sock'
                
                script {
                    // Install Docker Compose
                    sh 'sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose'
                    sh 'sudo chmod +x /usr/local/bin/docker-compose'
                    sh 'sudo service docker start'
                }
            }
        }
        
        stage('AWS Setup') {
            steps {
                script {
                    // Set AWS environment variables
                    withEnv(["AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}", "AWS_DEFAULT_REGION=ap-southeast-1"]) {
                        sh 'aws configure set default.region ap-southeast-1'
                        sh 'eval $(aws ecr get-login --region ap-southeast-1 --no-include-email)'
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                sh 'rm -rf "static/build/react-v2/"'
                sh 'rm -rf "build_json/"'
                sh 'rm webpack-stats-v2.prod.json'
                sh 'echo \'{"status":"done","chunks":{}}\' > webpack-stats-v2.prod.json'
                sh 'mv jenkins-compose.yml docker-compose.yml'
                
                script {
                    // Remove Docker buildx containers
                    sh 'docker buildx stop'
                    sh 'docker buildx rm --all-inactive -f'
                    sh 'docker buildx prune -f'
                    
                    // Docker login
                    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
                }
            }
        }
        
        stage('Build and Deploy') {
            steps {
                script {
                    sh 'docker-compose build'
                    sh 'docker-compose down --remove-orphans'
                    sh 'docker-compose up'
                    
                    // Replace webpack stats
                    sh 'python3 replace_webpack_stats.py "15"'
                    
                    // Stash changes
                    gitStashPush('docker-compose.yml')
                    gitStashPush('jenkins-compose.yml')
                }
            }
        }
        
        stage('Finalize') {
            steps {
                script {
                    sh 'git status'
                    sh 'git checkout ${BRANCH}'
                    sh 'git add .'
                    sh 'git commit -m "auto-bundles-push-`date +\'%Y-%m-%d\'`"'
                    sh 'docker-compose down --remove-orphans'
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline succeeded!'
        }
        
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }
}
