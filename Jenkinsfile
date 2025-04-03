pipeline {
    agent none
    environment {
        AWS_ACCESS_KEY_ID = credentials('aws_access_key_id') // Use Jenkins credentials
        AWS_SECRET_ACCESS_KEY = credentials('aws_secret_access_key') // Use Jenkins credentials
        githubuser = credentials('github_user_creds').username // Use Jenkins credentials
        githubpassword = credentials('github_user_creds').password // Use Jenkins credentials
        ec2_public_ip = ""
    }

    stages {
        stage('Create aws infra') {
            agent { label 'jenkins-ssh-terraform' }
            environment {
                PATH = "${env.PATH}:/home/jenkins/terraform"
            }
            steps {
                git branch: '${GIT_BRANCH}', url: '${GIT_URL}' // Use provided branch and URL
                sh "terraform version"
                dir("iac-starter/prod/terraform") {
                    sh "terraform init"
                    sh "terraform apply --auto-approve"
                    script {
                        ec2_public_ip = sh(returnStdout: true, script: "terraform state show aws_instance.my-public-ec2 | grep -w public_ip | awk -F'\"' '{print \$2}'").trim()
                    }
                    sh "echo ${ec2_public_ip}"
                }
            }
            post {
                always {
                    dir("iac-starter/prod/terraform") {
                        archiveArtifacts artifacts: 'my_tf_key.pem', fingerprint: true
                    }
                }
            }
        }

        stage('Configure ec2 infra and deploy app') {
            agent { label 'jenkins-ssh-ansible' }
            environment {
                ANSIBLE_HOST_KEY_CHECKING = 'False'
            }
            steps {
                git branch: '${GIT_BRANCH}', url: '${GIT_URL}' // Use provided branch and URL
                sh 'ansible --version'
                dir("iac-starter/prod/ansible") {
                    step([
                        $class: 'CopyArtifact',
                        filter: 'my_tf_key.pem',
                        fingerprintArtifacts: true,
                        optional: true,
                        projectName: env.JOB_NAME,
                        selector: [$class: 'SpecificBuildSelector', buildNumber: env.BUILD_NUMBER]
                    ])
                    sh 'sudo chmod 0600 my_tf_key.pem'
                    sh 'echo [ec2] > temp_hosts'
                    sh "echo '${ec2_public_ip} ansible_connection=ssh ansible_user=ec2-user ansible_ssh_private_key_file=./my_tf_key.pem' >> temp_hosts"
                    sh 'cat temp_hosts'
                    sh "ansible-playbook playbooks/configure-ec2.yaml -i ./temp_hosts --extra-vars \"githubuser=${env.githubuser} githubpassword=${env.githubpassword}\""
                }
            }
        }
    }
}
