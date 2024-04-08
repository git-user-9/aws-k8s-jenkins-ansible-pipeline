pipeline {
    agent any

    environment {
        ANSIBLE_SERVER = "ansible@<ANSIBLE_SERVER_IP>"
        DOCKER_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Transfer to Ansible Node') {
            steps {
                sshagent(['ansible-server-key']) {
                    // Sync the entire workspace to /opt on Ansible server
                    // Note: This assumes the user wants to sync the repo structure to /opt
                    // So /opt/app/Dockerfile, /opt/k8s/deployment.yaml, etc.
                    sh """
                        rsync -avh -e "ssh -o StrictHostKeyChecking=no" ./ ${ANSIBLE_SERVER}:/opt/
                    """
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                sshagent(['ansible-server-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_SERVER} '
                            cd /opt/app
                            docker build -t docky9/cddproj:v1.${DOCKER_TAG} .
                            docker tag docky9/cddproj:v1.${DOCKER_TAG} docky9/cddproj:latest
                            docker push docky9/cddproj:v1.${DOCKER_TAG}
                            docker push docky9/cddproj:latest
                            docker rmi docky9/cddproj:v1.${DOCKER_TAG} docky9/cddproj:latest
                        '
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sshagent(['ansible-server-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_SERVER} '
                            ansible-playbook /opt/ansible/ansible.yml --extra-vars "build_id=${DOCKER_TAG}"
                        '
                    """
                }
            }
        }
    }
}
