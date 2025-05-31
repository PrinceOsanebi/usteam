pipeline {
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-docker-repo')
        NVD_API_KEY= credentials('nvd-key')
        ANSIBLE_IP = credentials('ansible-ip')
        BASTION_IP = credentials('bastion-ip')
    }
    stages {
        stage('Code analisys stage') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Quality gate') {
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
        stage('Build artefacts') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }
                
        stage('Push artifacts to nexus-repo') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
                classifier: '',
                file: 'target/spring-petclinic-2.4.2.war',
                type: 'war']],
                credentialsId: 'nexus-cred',
                groupId: 'Petclinic',
                nexusUrl: 'nexus.edenboutique.space',
                nexusVersion: 'nexus3',
                protocol: 'https',
                repository: 'nexus-repo',
                version: '1.0'
            }
        }

        stage('Build Docker image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/petclinicapps .'
            }
        }
        
        stage('Login to Nexus repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Push image to Nexus repo') {
            steps {
                sh 'docker push $NEXUS_REPO/petclinicapps'
            }
        }
        stage('Trivy image scan') {
            steps {
                sh "trivy image $NEXUS_REPO/petclinicapps > trivy.txt"
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                         ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ec2-user@${BASTION_IP}" ec2-user@${ANSIBLE_IP} "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
                   '''
                }
            }
        }
        stage('check stage website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://stage.edenboutique.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.edenboutique.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic website is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic wordpress website appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
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
        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                         ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ec2-user@${BASTION_IP}" ec2-user@${ANSIBLE_IP} "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                   '''
                }
            }
        }
        stage('check prod website availability') {
            steps {
                 sh "sleep 200"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://www.edenboutique.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://www.edenboutique.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The prod petclinic website is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The prod petclinic wordpress website appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
