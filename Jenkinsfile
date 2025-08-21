pipeline {
    agent none

    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['netbox', 'other'],
            description: 'Where to deploy the playbook after linting?'
        )
        string(
            name: 'PLAYBOOK_PATH',
            defaultValue: 'playbook.yml',
            description: 'Relative path to the Ansible playbook to lint and deploy.'
        )
        string(
            name: 'GIT_REPO',
            defaultValue: 'https://github.com/dashton956-alt/test3.git',
            description: 'GitHub repository URL containing the playbook.'
        )
        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch to checkout.'
        )
        string(
            name: 'INVENTORY',
            defaultValue: '',
            description: 'Path to Ansible inventory file (optional).'
        )
        string(
            name: 'DEST_FOLDER',
            defaultValue: 'completed_playbooks',
            description: 'Folder in repo to copy completed playbooks to.'
        )
        string(
            name: 'GITHUB_CREDS_ID',
            defaultValue: '40b29ecf-72cd-4262-bb38-fe1dbe9bf47e',
            description: 'Jenkins credentials ID for GitHub access.'
        )
        string(
            name: 'NETBOX_CREDS_ID',
            defaultValue: '8cdf19b1-2e05-4b2d-b317-66a148163fa4',
            description: 'Jenkins credentials ID for NetBox API token.'
        )
    }

    stages {
        stage('Checkout') {
            agent any
            steps {
                // Install Ansible and ansible-lint if not present
                sh 'which ansible || (apt-get update && apt-get install -y python3-pip && pip3 install --break-system-packages ansible ansible-lint)'
                withCredentials([string(credentialsId: params.GITHUB_CREDS_ID, variable: 'GIT_TOKEN')]) {
                    // Clone the repo to a temp dir
                    sh '''
                        rm -rf repo_tmp && mkdir repo_tmp
                        git clone --branch ${GIT_BRANCH} https://${GIT_TOKEN}:x-oauth-basic@${GIT_REPO#https://} repo_tmp
                    '''
                }
            }
        }
        stage('Lint Ansible Playbook') {
            agent any
            steps {
                sh "ansible-lint repo_tmp/${params.PLAYBOOK_PATH} | tee ansible-lint.log"
            }
        }
        stage('Copy & Commit Playbook') {
            agent any
            steps {
                withCredentials([string(credentialsId: params.GITHUB_CREDS_ID, variable: 'GIT_TOKEN')]) {
                    script {
                        sh "cd repo_tmp && mkdir -p N8N_Netbox_complete && mv ../${params.PLAYBOOK_PATH} N8N_Netbox_complete/"
                        sh '''
                            cd repo_tmp
                            git config user.name "ci-cd-bot"
                            git config user.email "ci-cd-bot@example.com"
                            git add N8N_Netbox_complete/$(basename ../${params.PLAYBOOK_PATH})
                            git commit -m "CI/CD completed playbook: build #${BUILD_NUMBER}"
                            git push https://${GIT_TOKEN}:x-oauth-basic@${GIT_REPO#https://} HEAD:${GIT_BRANCH}
                        '''
                    }
                }
            }
        }
        stage('Deploy') {
            agent any
            when {
                anyOf {
                    expression { params.DEPLOY_TARGET == 'netbox' }
                    expression { params.DEPLOY_TARGET == 'other' }
                }
            }
            steps {
                script {
                    if (params.DEPLOY_TARGET == 'netbox') {
                        withCredentials([string(credentialsId: params.NETBOX_CREDS_ID, variable: 'NETBOX_TOKEN')]) {
                            echo 'Deploying playbook to NetBox...'
                            // sh 'ansible-playbook -i ${params.INVENTORY} --extra-vars "token=$NETBOX_TOKEN build_number=${BUILD_NUMBER}" ${params.PLAYBOOK_PATH}'
                            // sh 'curl -X POST -H "Authorization: Token $NETBOX_TOKEN" -H "Content-Type: application/json" --data "{\"build_number\": \"${BUILD_NUMBER}\"}" https://netbox.example/api/your-endpoint/'
                        }
                    } else {
                        echo "Deploying playbook to ${params.DEPLOY_TARGET}..."
                        // sh 'ansible-playbook -i ${params.INVENTORY} --extra-vars "build_number=${BUILD_NUMBER}" ${params.PLAYBOOK_PATH}'
                    }
                }
            }
        }
        stage('Notifications') {
            agent any
            steps {
                echo 'No notifications configured.'
                // slackSend(color: 'good', message: "Ansible Lint & Deploy Pipeline Succeeded! Build #${BUILD_NUMBER}")
                // emailext(subject: 'Pipeline Success', body: "Build #${BUILD_NUMBER} succeeded", recipientProviders: [developers()])
            }
        }
        stage('Cleanup') {
            agent any
            steps {
                echo 'No cleanup steps.'
                // sh 'rm -f sensitive_file'
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: '**/ansible-lint.log', allowEmptyArchive: true
            // archiveArtifacts artifacts: '**/*.yml', allowEmptyArchive: true // <-- Uncomment to archive playbooks
            // archiveArtifacts artifacts: '**/deploy.log', allowEmptyArchive: true // <-- Uncomment to archive deploy logs
        }
        failure {
            echo 'No failure notifications configured.'
            // slackSend(color: 'danger', message: "Ansible Lint & Deploy Pipeline FAILED! Build #${BUILD_NUMBER}") // <-- Uncomment and configure
            // emailext(subject: 'Pipeline Failure', body: "Build #${BUILD_NUMBER} failed", recipientProviders: [developers()]) // <-- Uncomment and configure
        }
    }
}
