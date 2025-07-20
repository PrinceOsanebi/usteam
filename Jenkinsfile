pipeline {
    agent any

    environment {
        NEXUS_USER     = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO     = credentials('nexus-ip-port')
        BASTION_IP     = credentials('bastion-ip')
        ANSIBLE_IP     = credentials('ansible-ip')
        NVD_API_KEY    = credentials('nvd-key')
        BASTION_ID     = credentials('bastion-id')
        AWS_REGION     = 'eu-west-1'
    }

    parameters {
        choice(name: 'action', choices: ['apply', 'destroy'], description: 'Select the action to perform')
    }

    triggers {
               pollSCM('* * * * *') // Uncomment if required
    }

    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Dependency check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey $NVD_API_KEY",
                        odcInstallation: 'DP-Check'
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }

        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader(
                    artifacts: [[artifactId: 'spring-petclinic',
                                 classifier: '',
                                 file: 'target/spring-petclinic-2.4.2.war',
                                 type: 'war']],
                    credentialsId: 'nexus-cred',
                    groupId: 'Petclinic',
                    nexusUrl: 'nexus.pmolabs.space',
                    nexusVersion: 'nexus3',
                    protocol: 'https',
                    repository: 'nexus-repo',
                    version: '1.0'
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t "$NEXUS_REPO/petclinicapps" .'
            }
        }

        stage('Log Into Nexus Docker Repo') {
            steps {
                sh 'echo "$NEXUS_PASSWORD" | docker login --username "$NEXUS_USER" --password-stdin "$NEXUS_REPO"'
            }
        }

        stage('Trivy image Scan') {
            steps {
                sh 'trivy image -f table "$NEXUS_REPO/petclinicapps" > trivyfs.txt'
            }
        }

        stage('Push to Nexus Docker Repo') {
            steps {
                sh 'docker push "$NEXUS_REPO/petclinicapps"'
            }
        }

        stage('Prune Docker Images') {
            steps {
                sh 'docker image prune -f'
            }
        }

        stage('Deploying to Stage Environment') {
            steps {
                script {
                    sh '''
                      aws ssm start-session \
                        --target $BASTION_ID \
                        --region $AWS_REGION \
                        --document-name AWS-StartPortForwardingSession \
                        --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' &
                      sleep 5
                    '''
                    sshagent(['ansible-key']) {
                        sh '''
                          ssh-keygen -R "[localhost]:9999" || true
                          ssh -o StrictHostKeyChecking=no \
                              -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                              ec2-user@$ANSIBLE_IP \
                              "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
                        '''
                    }
                    sh 'pkill -f "aws ssm start-session" || true'
                }
            }
        }

        stage('Check Stage Website Availability') {
            steps {
                sh 'sleep 90'
                script {
                    def response = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.pmolabs.space', returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "✅ Stage environment is up. Status: ${response}", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "❌ Stage environment is down. Status: ${response}", tokenCredentialId: 'slack')
                    }
                }
            }
        }

        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10, unit: 'MINUTES') {
                    input message: 'Approve Production Deployment?', submitter: 'admin'
                }
            }
        }

        stage('Deploying to Prod Environment') {
            steps {
                script {
                    sh '''
                      aws ssm start-session \
                        --target $BASTION_ID \
                        --region $AWS_REGION \
                        --document-name AWS-StartPortForwardingSession \
                        --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' &
                      sleep 5
                    '''
                    sshagent(['ansible-key']) {
                        sh '''
                          ssh-keygen -R "[localhost]:9999" || true
                          ssh -o StrictHostKeyChecking=no \
                              -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                              ec2-user@$ANSIBLE_IP \
                              "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                        '''
                    }
                    sh 'pkill -f "aws ssm start-session" || true'
                }
            }
        }

        stage('Check Prod Website Availability') {
            steps {
                sh 'sleep 90'
                script {
                    def response = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.pmolabs.space', returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "✅ Prod environment is up. Status: ${response}", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "❌ Prod environment is down. Status: ${response}", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
