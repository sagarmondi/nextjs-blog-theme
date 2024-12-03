pipeline {
    agent any
    
    environment {
        // Deployment details
        PRODUCTION_IP_ADDRESS = '52.43.163.218'
        APP_PORT = '3005'  // Unique port for the blog application
        APP_NAME = 'nextjs-blog'
        
        // Semgrep and other tool configurations
        SEMGREP_APP_TOKEN = credentials('SEMGREP_APP_TOKEN')
        NUCLEI_TEMPLATES_PATH = '/home/ec2-user/nuclei-templates/'
    }
    
    tools {
        nodejs "nodejs"
    }
    
    stages {
               
        stage('Install Dependencies') {
            steps {
                script {
                    sh '''
                        npm install -g yarn pm2
                        yarn install
                        
                        # Install TypeScript types
                        yarn add --dev @types/react @types/node typescript
                        
                        # Ensure TypeScript configuration exists
                        if [ ! -f tsconfig.json ]; then
                            yarn tsc --init
                        fi
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    sh '''
                        # Clear any previous builds
                        rm -rf .next
                        
                        # Configure Next.js build caching
                        mkdir -p .next/cache
                        
                        # Build the application with verbose output
                        NEXT_TELEMETRY_DISABLED=1 yarn build
                    '''
                }
            }
        }
        
        stage('SAST - Semgrep Scan') {
            // Make this stage conditional to continue even if build fails
            when {
                expression { 
                    currentBuild.resultIsBetterOrEqualTo('SUCCESS') 
                }
            }
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh '''
                            pip3 install semgrep
                            
                            # Run Semgrep scan with comprehensive checks
                            semgrep ci \
                                --config=p/default \
                                --config=p/react \
                                --config=p/next.js \
                                --json -o semgrep-results.json || true
                            
                            # Extract and list used rules
                            jq -r '.results[].extra.rule_id // .results[].check_id' semgrep-results.json | 
                            sort | uniq > semgrep-used-rules.txt
                            
                            echo "Semgrep Rules Used:"
                            cat semgrep-used-rules.txt
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'semgrep-results.json,semgrep-used-rules.txt', allowEmptyArchive: true
                }
            }
        }
        
        stage('Deploy to Production') {
            environment {
                DEPLOY_SSH_KEY = credentials('aws_ssh_key')
            }
            steps {
                script {
                    sh '''
                        chmod 600 $DEPLOY_SSH_KEY
                        ssh -o StrictHostKeyChecking=no -i $DEPLOY_SSH_KEY ec2-user@$PRODUCTION_IP_ADDRESS '
                            # Setup Node.js environment
                            curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash
                            source ~/.bashrc
                            nvm install --lts
                            
                            # Prepare application directory
                            mkdir -p ~/apps/'"$APP_NAME"'
                            cd ~/apps/'"$APP_NAME"'
                            
                            # Clone or update repository
                            if [ ! -d ".git" ]; then
                                git clone https://github.com/sagarmondi/nextjs-blog-theme.git .
                            else
                                git pull
                            fi
                            
                            # Install dependencies
                            yarn install
                            
                            # Install TypeScript types
                            yarn add --dev @types/react @types/node typescript
                            
                            # Build the application
                            NEXT_TELEMETRY_DISABLED=1 yarn build
                            
                            # Stop existing PM2 process if exists
                            pm2 delete '"$APP_NAME"' || true
                            
                            # Start new application instance
                            PORT='"$APP_PORT"' pm2 start --name '"$APP_NAME"' npm -- start
                            
                            # Ensure PM2 saves and starts on boot
                            pm2 save
                            pm2 startup
                        '
                    '''
                }
            }
        }
        
        stage('DAST - Nuclei Scan') {
            steps {
                script {
                    sh '''
                        # Install Nuclei if not exists
                        which nuclei || {
                            wget https://github.com/projectdiscovery/nuclei/releases/download/v3.1.3/nuclei_3.1.3_linux_amd64.zip
                            unzip nuclei_3.1.3_linux_amd64.zip
                            sudo mv nuclei /usr/local/bin/
                        }
                        
                        # Perform Nuclei scan
                        nuclei -u http://$PRODUCTION_IP_ADDRESS:$APP_PORT \
                               -t $NUCLEI_TEMPLATES_PATH \
                               -o nuclei-results.txt
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'nuclei-results.txt', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo "Next.js Blog Deployment Pipeline Completed"
            cleanWs()
        }
        
        failure {
            echo "Blog Deployment Failed. Sending notifications..."
        }
    }
}
