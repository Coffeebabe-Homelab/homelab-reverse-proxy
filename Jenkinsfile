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
        DEPLOY_HOST = 'staging-deploy'
        DEPLOY_USER = 'deploy'
        DEPLOY_PATH = '~/staging/reverse-proxy'
        REPOSITORY = 'git@github.com:Coffeebabe-Homelab/homelab-reverse-proxy.git'
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

        stage('Checkout staging') {
            steps {
                git branch: 'staging',
                    url: env.REPOSITORY,
                    credentialsId: 'git-checkout-key'
        
                sh '''
                    git config --global pull.rebase false
                    git config --global user.email "jenkins@mecha-rat"
                    git config --global user.name "Jenkins"
                    git fetch origin --prune
                    git fetch origin staging --prune
                    git checkout -B staging origin/staging
                    git branch --list
                '''
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['staging-ssh-key']) {
                    sh '''#!/bin/sh
                        set -eu
                        : "${DEPLOY_HOST:?Set DEPLOY_HOST as a Jenkins job or folder environment variable}"
                        : "${DEPLOY_USER:?Set DEPLOY_USER as a Jenkins job or folder environment variable}"
                        ssh ${DEPLOY_HOST} "mkdir -p ${DEPLOY_PATH}/nginx/conf.d"
                    '''
                    sh '''
                        scp docker-compose.yml "$DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_PATH/docker-compose.yml"
                        scp nginx/conf.d/homelab.conf "${DEPLOY_HOST}:${DEPLOY_PATH}/nginx/conf.d/homelab.conf"
                        ssh ${DEPLOY_HOST} "cd ${DEPLOY_PATH} \
                            && docker compose config -q \
                            && docker compose up -d --pull always && docker compose exec -T nginx nginx -t"
                    '''
                }
            }
        }
    }
}
