pipeline {
    triggers {
        pollSCM('*/1 * * * *')
    }
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
            defaultValue: 'GitHub',
            description: 'Jenkins credentials ID for GitHub access.'
        )
        string(
            name: 'NETBOX_CREDS_ID',
            defaultValue: 'Netbox',
            description: 'Jenkins credentials ID for NetBox API token.'
        )
    }

    stages {
        stage('Checkout & Find Files') {
            agent any
            steps {
                // ...existing code...
                withCredentials([string(credentialsId: params.GITHUB_CREDS_ID, variable: 'GIT_TOKEN')]) {
                    // Clone the repo to a temp dir
                    sh '''
                        rm -rf repo_tmp && mkdir repo_tmp
                        git clone --branch ${GIT_BRANCH} https://${GIT_TOKEN}:x-oauth-basic@${GIT_REPO#https://} repo_tmp
                    '''
                }
                script {
                    // Find all files in N8N_Netbox_Pending folder
                    env.FILES_TO_PROCESS = sh(
                        script: '''cd repo_tmp && find N8N_Netbox_Pending -type f | xargs''',
                        returnStdout: true
                    ).trim()
                    if (!env.FILES_TO_PROCESS) {
                        echo 'No files found in N8N_Netbox_Pending. Skipping pipeline.'
                        currentBuild.result = 'SUCCESS'
                        error('No files to process.')
                    } else {
                        echo "Files to process: ${env.FILES_TO_PROCESS}"
                    }
                }
            }
        }
        stage('Lint Ansible Playbook') {
            agent any
            steps {
                // ...existing code...
                script {
                    env.LINT_FAIL_FILES = ''
                    env.LINT_PASS_FILES = ''
                    def mdFile = "lint-failures-${env.BUILD_NUMBER}.md"
                    writeFile file: "repo_tmp/${mdFile}", text: "# Lint Failures for Build #${env.BUILD_NUMBER}\n\n"
                    for (file in env.FILES_TO_PROCESS.tokenize()) {
                        def lintLog = "ansible-lint-${file.replaceAll('[^a-zA-Z0-9_.-]', '_')}.log"
                        echo "Linting repo_tmp/${file}..."
                        def result = sh(
                            script: "ansible-lint repo_tmp/${file} > repo_tmp/${lintLog} 2>&1",
                            returnStatus: true
                        )
                        if (result == 0) {
                            echo "Lint PASSED for ${file}"
                            env.LINT_PASS_FILES += file + ' '
                        } else {
                            echo "Lint FAILED for ${file}. See below:"
                            def lintErr = readFile("repo_tmp/${lintLog}")
                            echo lintErr
                            env.LINT_FAIL_FILES += file + ' '
                            def prev = readFile("repo_tmp/${mdFile}")
                            def mdBlock = """## ${file}\n\n```\n${lintErr}\n```\n\n"""
                            writeFile file: "repo_tmp/${mdFile}", text: prev + mdBlock
                        }
                    }
                }
            }
        }
        stage('Copy & Commit Playbook') {
                        agent any
                        steps {
                                withCredentials([string(credentialsId: params.GITHUB_CREDS_ID, variable: 'GIT_TOKEN')]) {
                                        script {
                                                sh '''
                                                        cd repo_tmp
                                                        mkdir -p N8N_Netbox_complete N8N_Netbox_Lint_failure
                                                        # Move passed files to complete, failed to failure
                                                        for f in ${LINT_PASS_FILES}; do
                                                            [ -f "$f" ] && mv "$f" N8N_Netbox_complete/;
                                                        done
                                                        for f in ${LINT_FAIL_FILES}; do
                                                            [ -f "$f" ] && mv "$f" N8N_Netbox_Lint_failure/;
                                                        done
                                                        # Remove any remaining files from pending folder
                                                        rm -f N8N_Netbox_Pending/* || true
                                                '''
                                                // Commit and push passed files and pending folder cleanup to main branch
                                                sh '''
                                                        cd repo_tmp
                                                        git config user.name "ci-cd-bot"
                                                        git config user.email "ci-cd-bot@example.com"
                                                        git add N8N_Netbox_complete/* || true
                                                        git add -u N8N_Netbox_Pending || true
                                                        if git diff --cached --quiet; then
                                                            echo "No completed files or pending folder changes to commit."
                                                        else
                                                            git commit -m "CI/CD completed playbooks and cleaned pending: build #${BUILD_NUMBER}"
                                                            git push https://${GIT_TOKEN}:x-oauth-basic@${GIT_REPO#https://} HEAD:${GIT_BRANCH}
                                                        fi
                                                '''
                                                // If there are failed files, create a branch, commit, and push
                                                sh '''
                                                        cd repo_tmp
                                                        if [ -n "${LINT_FAIL_FILES}" ]; then
                                                            branch=lint-failed-${BUILD_NUMBER}
                                                            git checkout -b $branch
                                                            git add N8N_Netbox_Lint_failure/* lint-failures-${BUILD_NUMBER}.md || true
                                                            if git diff --cached --quiet; then
                                                                echo "No failed files to commit."
                                                            else
                                                                git commit -m "Lint failures: build #${BUILD_NUMBER}"
                                                                git push https://${GIT_TOKEN}:x-oauth-basic@${GIT_REPO#https://} HEAD:$branch
                                                            fi
                                                            git checkout ${GIT_BRANCH}
                                                        fi
                                                '''
                                        }
                                }
                        }
        }
        stage('Deploy') {
            agent any
            when {
                allOf {
                    anyOf {
                        expression { params.DEPLOY_TARGET == 'netbox' }
                        expression { params.DEPLOY_TARGET == 'other' }
                    }
                    expression { env.LINT_PASS_FILES?.trim() }
                }
            }
            steps {
                // ...existing code...
                script {
                    def passedFiles = env.LINT_PASS_FILES?.tokenize() ?: []
                    if (passedFiles.size() == 0) {
                        echo 'No files passed linting. Skipping deploy.'
                        return
                    }
                    if (params.DEPLOY_TARGET == 'netbox') {
                        withCredentials([string(credentialsId: params.NETBOX_CREDS_ID, variable: 'NETBOX_TOKEN')]) {
                            for (file in passedFiles) {
                                def playbookPath = "repo_tmp/N8N_Netbox_complete/" + file.tokenize('/').last()
                                echo "Deploying ${playbookPath} to NetBox..."
                                withEnv(["NB_TOKEN=${NETBOX_TOKEN}"]) {
                                    sh "ansible-playbook ${playbookPath} --extra-vars netbox_token=\"$NB_TOKEN\""
                                }
                            }
                        }
                    } else {
                        for (file in passedFiles) {
                            def playbookPath = "repo_tmp/N8N_Netbox_complete/" + file.tokenize('/').last()
                            echo "Deploying ${playbookPath} to ${params.DEPLOY_TARGET}..."
                            sh "ansible-playbook -i ${params.INVENTORY} --extra-vars 'build_number=${BUILD_NUMBER}' ${playbookPath}"
                        }
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
