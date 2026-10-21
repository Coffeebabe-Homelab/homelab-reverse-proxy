pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        choice(name: 'TARGET', choices: ['staging'], description: 'Deployment target. Production requires a separate ingress and pipeline.')
    }

    environment {
        DEPLOY_PATH = '/opt/home-lab-reverse-proxy'
    }

    stages {
        stage('Validate target') {
            steps {
                script {
                    if (params.TARGET != 'staging') {
                        error('Only private staging is configured.')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['homelab-ssh-key']) {
                    sh '''#!/bin/sh
                        set -eu
                        : "${DEPLOY_HOST:?Set DEPLOY_HOST as a Jenkins job or folder environment variable}"
                        : "${DEPLOY_USER:?Set DEPLOY_USER as a Jenkins job or folder environment variable}"
                        ssh -o BatchMode=yes -o StrictHostKeyChecking=yes "$DEPLOY_USER@$DEPLOY_HOST" \
                          "mkdir -p '$DEPLOY_PATH/nginx/conf.d'"
                        scp -o BatchMode=yes -o StrictHostKeyChecking=yes \
                          docker-compose.yml "$DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_PATH/docker-compose.yml"
                        scp -o BatchMode=yes -o StrictHostKeyChecking=yes \
                          nginx/conf.d/homelab.conf "$DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_PATH/nginx/conf.d/homelab.conf"
                        ssh -o BatchMode=yes -o StrictHostKeyChecking=yes "$DEPLOY_USER@$DEPLOY_HOST" \
                          "cd '$DEPLOY_PATH' && docker compose config -q && docker compose up -d --pull always && docker compose exec -T nginx nginx -t"
                    '''
                }
            }
        }
    }
}
