pipeline{
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('docker-repo')
        BASTION_IP = credentials('bastion-ip')
        ANSIBLE_IP = credentials('ansible-ip')
        NVD_API_KEY= credentials('nvd-key')
        BASTION_ID= credentials('bastion-id')
        AWS_REGION= 'eu-west-1'
    }
    triggers {
        pollSCM('* * * * *') // Runs every minute
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
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
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
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/petclinicapps .'
            }
        }
        stage('Log Into Nexus Docker Repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image -f table -o trivyreport.txt $NEXUS_REPO/petclinicapps"
            }
        }
        stage('Push to Nexus Docker Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/petclinicapps'
            }
        }
        stage('prune docker images') {
            steps {
                sh 'docker image prune -f'
            }
        }
        stage ('Deploying to Stage Environment') {
          steps {
              script {
                // Start SSM session to bastion with port forwarding for SSH (port 22)
                sh '''
                  aws ssm start-session \
                    --target ${BASTION_ID} \
                    --region ${AWS_REGION} \
                    --document-name AWS-StartPortForwardingSession \
                    --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                    &
                  sleep 5  # Wait for port forwarding to establish
                '''
                // SSH through the tunnel to Ansible server on port 22
                sshagent(['ansible-key']) {
                  sh '''
                    ssh -o StrictHostKeyChecking=no \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                        ubuntu@${ANSIBLE_IP} \
                        "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
                  '''
                }
                // Terminate the SSM session
                sh 'pkill -f "aws ssm start-session"'
              }
          }
        }
        // stage('Deploy to stage') {
        //     steps {
        //         sshagent(['ansible-key']) {
        //             sh '''
        //               ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@${BASTION_IP}" ubuntu@${ANSIBLE_IP} "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
        //             '''
        //         }
        //     }
        // }
        stage('check stage website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://stage.pmolabs.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.pmolabs.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage ('Deploying to prod Environment') {
          steps {
              script {
                // Start SSM session to bastion with port forwarding for SSH (port 22)
                sh '''
                  aws ssm start-session \
                    --target ${BASTION_ID} \
                    --region ${AWS_REGION} \
                    --document-name AWS-StartPortForwardingSession \
                    --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                    &
                  sleep 5  # Wait for port forwarding to establish
                '''
                // SSH through the tunnel to Ansible server on port 22
                sshagent(['ansible-key']) {
                  sh '''
                    ssh -o StrictHostKeyChecking=no \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                        ubuntu@${ANSIBLE_IP} \
                        "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                  '''
                }
                // Terminate the SSM session
                sh 'pkill -f "aws ssm start-session"'
              }
          }
        }
        stage('check prod website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://prod.pmolabs.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.pmolabs.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The prod petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The prod petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
    
