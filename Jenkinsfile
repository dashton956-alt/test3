pipeline {
    agent any

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
            defaultValue: 'https://github.com/your-org/your-ansible-repo.git',
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
            defaultValue: 'github-creds-id',
            description: 'Jenkins credentials ID for GitHub access.'
        )
        string(
            name: 'NETBOX_CREDS_ID',
            defaultValue: 'netbox-token-id',
            description: 'Jenkins credentials ID for NetBox API token.'
        )
    }

    environment {
        GITHUB_CREDENTIALS = credentials(params.GITHUB_CREDS_ID)
        NETBOX_TOKEN = credentials(params.NETBOX_CREDS_ID)
        // OTHER_DEPLOY_CRED = credentials('other-deploy-creds-id') // <-- For other targets
    }

    stages {
        stage('Install Ansible & Lint') {
            steps {
                // sh 'pip install ansible ansible-lint' // <-- Uncomment if needed
                sh 'ansible --version || true' // Check if Ansible is installed
                sh 'ansible-lint --version || true' // Check if ansible-lint is installed
            }
        }
        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: params.GIT_BRANCH]],
                    userRemoteConfigs: [[
                        url: params.GIT_REPO,
                        credentialsId: env.GITHUB_CREDENTIALS
                    ]]
                ])
            }
        }
        stage('Lint Ansible Playbook') {
            steps {
                // Optionally redirect output to a log file
                sh "ansible-lint ${params.PLAYBOOK_PATH} | tee ansible-lint.log"
            }
        }
        stage('Copy & Commit Playbook') {
            steps {
                script {
                    // Copy playbook to destination folder
                    sh "mkdir -p ${params.DEST_FOLDER}"
                    sh "cp ${params.PLAYBOOK_PATH} ${params.DEST_FOLDER}/"
                    // Git add, commit, and push with CI/CD build number
                    // withCredentials([usernamePassword(credentialsId: env.GITHUB_CREDENTIALS, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        git config user.name "ci-cd-bot"
                        git config user.email "ci-cd-bot@example.com"
                        git add ${params.DEST_FOLDER}/$(basename ${params.PLAYBOOK_PATH})
                        git commit -m "CI/CD completed playbook: build #${BUILD_NUMBER}"
                        git push origin HEAD:${params.GIT_BRANCH}
                    '''
                    // }
                }
            }
        }
        stage('Deploy') {
            when {
                anyOf {
                    expression { params.DEPLOY_TARGET == 'netbox' }
                    expression { params.DEPLOY_TARGET == 'other' }
                }
            }
            steps {
                script {
                    if (params.DEPLOY_TARGET == 'netbox') {
                        echo 'Deploying playbook to NetBox...'
                        // sh 'ansible-playbook -i ${params.INVENTORY} --extra-vars "token=${env.NETBOX_TOKEN} build_number=${BUILD_NUMBER}" ${params.PLAYBOOK_PATH}' // <-- Uncomment and edit as needed
                        // Example: Send build number to NetBox via API
                        // sh 'curl -X POST -H "Authorization: Token ${env.NETBOX_TOKEN}" -H "Content-Type: application/json" --data "{\"build_number\": \"${BUILD_NUMBER}\"}" https://netbox.example/api/your-endpoint/'
                        // Add NetBox deployment logic here
                    } else {
                        echo "Deploying playbook to ${params.DEPLOY_TARGET}..."
                        // sh 'ansible-playbook -i ${params.INVENTORY} --extra-vars "build_number=${BUILD_NUMBER}" ${params.PLAYBOOK_PATH}' // <-- Uncomment and edit as needed
                        // Add other deployment logic here
                    }
                }
            }
        }
        stage('Notifications') {
            steps {
                echo 'No notifications configured.'
                // slackSend(color: 'good', message: "Ansible Lint & Deploy Pipeline Succeeded! Build #${BUILD_NUMBER}") // <-- Uncomment and configure
                // emailext(subject: 'Pipeline Success', body: "Build #${BUILD_NUMBER} succeeded", recipientProviders: [developers()]) // <-- Uncomment and configure
            }
        }
        stage('Cleanup') {
            steps {
                echo 'No cleanup steps.'
                // sh 'rm -f sensitive_file' // <-- Add cleanup commands as needed
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
