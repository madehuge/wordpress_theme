def remote = [:]
pipeline {
    agent any
    environment {
        // Webhook URL to connect with Microsoft Teams to notify of job status
        OFFICE365_WEBHOOK = ''
        // Whether or not building the theme using npm is required (true or false)
        IS_BUILD_THEME = 'false'
        // Used to fetch the Docker image to build the theme using npm
        DOCKER_NODE_IMAGE = 'node:15.4'
        // Directory to assemble all WordPress files to be deployed
        TARGET = 'deploy'
        // WordPress theme that will be built and deployed
        WP_THEME = 'cema-backend'

        // Public IP address for staging server retrieved from global settings
        QA_IP_ADDRESS = "${GLOBAL_CEMA_QA_STAGING_SERVER_IP}"
        // Directory on staging server to deploy code
        QA_DIR = '/var/www/cema-qa'
        // Production web server user for ownership of files
        QA_WEB_SERVER_USER = 'apache'
        // Jenkins credential id for private key to SSH to staging server
        QA_CREDENTIALS_ID = 'cema-support-qa-staging-server-private-key'

        // Public IP address for staging server retrieved from global settings
        STAGING_IP_ADDRESS = "${GLOBAL_CEMA_QA_STAGING_SERVER_IP}"
        // Directory on staging server to deploy code
        STAGING_DIR = '/var/www/cema-staging'
        // Production web server user for ownership of files
        STAGING_WEB_SERVER_USER = 'apache'
        // Jenkins credential id for private key to SSH to staging server
        STAGING_CREDENTIALS_ID = 'cema-support-qa-staging-server-private-key'

        // Parameters that will turn off SSH key checking
        SSH_KEY_CHECKING = '-o \'UserKnownHostsFile=/dev/null\' -o \'StrictHostKeyChecking=no\''
    }
    options {
        skipDefaultCheckout()
        disableConcurrentBuilds()
    }
    stages {
        stage('Prebuild') {
            steps {
                deleteDir()
                checkout scm
                sh "mkdir .npm"
//                office365Notify status: 'started', branch: 'staging', webhook: "${OFFICE365_WEBHOOK}"
            }
        }

        stage ('Build Theme') {
           agent {
               docker {
                    image "${DOCKER_NODE_IMAGE}"
                    args "-v ${WORKSPACE}/.npm:/.npm"
                    reuseNode true
               }
            }
            when {
                allOf {
                    branch 'staging'
                    expression {IS_BUILD_THEME == 'true'}
                }
            }
            steps {
                dir("wp-content/themes/${WP_THEME}") {
                    sh 'npm i'
                    sh 'npx gulp styles scripts'
                    sh 'rm -rf node_modules'
                    sh 'rm -rf sass'
                    sh 'rm package*.*'
                }
            }
        }

        stage ('Prepare Files') {
            when {
                anyOf {
                    branch 'testing'
                    branch 'staging'
                }
            }
            steps {
                sh "rm -rf .npm"
                sh "mkdir ${TARGET}"
                sh "mv *.php ${TARGET}"
                sh "mv wp-admin ${TARGET}"
                sh "mv wp-includes ${TARGET}"
                sh "mkdir ${TARGET}/wp-content"
                sh "mv wp-content/.htaccess ${TARGET}/wp-content"
                sh "mv wp-content/themes ${TARGET}/wp-content"
                sh "mv wp-content/plugins ${TARGET}/wp-content"
                sh "mv wp-content/index.php ${TARGET}/wp-content"
            }
        }

        stage ('Deploy QA') {
            when {
                branch 'testing'
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "${QA_CREDENTIALS_ID}", keyFileVariable: 'QA_KEY_FILE', usernameVariable: 'QA_USER_NAME')]) {
                    script {
                        remote = sshSetRemote(name: 'QA Server', host: "${QA_IP_ADDRESS}", user: QA_USER_NAME, identityFile: QA_KEY_FILE)
                    }
                    sh "rsync -e \"ssh -i ${QA_KEY_FILE} ${SSH_KEY_CHECKING}\" --rsync-path=\"sudo rsync\" -a --exclude='/.htaccess' --exclude='wp-content/uploads' --exclude='wp-config.php' --exclude='wordfence-waf.php' --delete ${TARGET}/ ${QA_USER_NAME}@${QA_IP_ADDRESS}:${QA_DIR}"
                    sshCommand remote: remote, command: "chown -R ${QA_WEB_SERVER_USER}: ${QA_DIR}", sudo: true
                    sshCommand remote: remote, command: "find ${QA_DIR} -type d -exec chmod 755 {} \\;", sudo: true
                    sshCommand remote: remote, command: "find ${QA_DIR} -type f -exec chmod 644 {} \\;", sudo: true
                    sshCommand remote: remote, command: "chmod 444 ${QA_DIR}/wp-config.php", sudo: true
                    sshCommand remote: remote, command: "find ${QA_DIR} -name .htaccess -exec chmod 644 {} \\;", sudo: true
                }
            }
        }

        stage ('Deploy Staging') {
            when {
                branch 'staging'
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: "${STAGING_CREDENTIALS_ID}", keyFileVariable: 'STAGING_KEY_FILE', usernameVariable: 'STAGING_USER_NAME')]) {
                    script {
                        remote = sshSetRemote(name: 'Staging Server', host: "${STAGING_IP_ADDRESS}", user: STAGING_USER_NAME, identityFile: STAGING_KEY_FILE)
                    }
                    sh "rsync -e \"ssh -i ${STAGING_KEY_FILE} ${SSH_KEY_CHECKING}\" --rsync-path=\"sudo rsync\" -a --exclude='/.htaccess' --exclude='wp-content/uploads' --exclude='wp-config.php' --exclude='wordfence-waf.php' --delete ${TARGET}/ ${STAGING_USER_NAME}@${STAGING_IP_ADDRESS}:${STAGING_DIR}"
                    sshCommand remote: remote, command: "chown -R ${STAGING_WEB_SERVER_USER}: ${STAGING_DIR}", sudo: true
                    sshCommand remote: remote, command: "find ${STAGING_DIR} -type d -exec chmod 755 {} \\;", sudo: true
                    sshCommand remote: remote, command: "find ${STAGING_DIR} -type f -exec chmod 644 {} \\;", sudo: true
                    sshCommand remote: remote, command: "chmod 444 ${STAGING_DIR}/wp-config.php", sudo: true
                    sshCommand remote: remote, command: "find ${STAGING_DIR} -name .htaccess -exec chmod 644 {} \\;", sudo: true
                }
            }
        }
    }

    post {
//         always {
//             office365Notify branch: 'staging', webhook: "${OFFICE365_WEBHOOK}"
//         }
        success {
            cleanWs()
            dir("${env.WORKSPACE}@tmp") {
                deleteDir()
            }
            dir("${env.WORKSPACE}@libs") {
                deleteDir()
            }
        }
    }
}