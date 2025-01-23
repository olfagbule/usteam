pipeline {
   agent any
   environment {
       NVD_API_KEY = credentials('nvd-key')
   }
   stages {
       stage('Code Checkout') {
           steps {
               git branch: 'main',
               credentialsId: 'git-cred',
               url: 'https://github.com/olfagbule/usteam.git'
           }
       }
       stage('Code Analysis') {
           steps {
              withSonarQubeEnv('sonarqube') {
                 sh "mvn sonar:sonar"
              }
           }
       }
       stage("Quality Gate") {
           steps {
             timeout(time: 2, unit: 'MINUTES') {
               waitForQualityGate abortPipeline: true
             }
           }
       }
       // stage('Test NVD API Key') {
       //     steps {
       //         sh """
       //             curl -H apiKey: ${NVD_API_KEY}" 'https://services.nvd.nist.gov/rest/json/cves/2.0?resultsPerPage=1'
       //         """
       //     }
       // }
       stage('Dependency check') {
           steps {
               dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY} --nvdApiDelay 6000 --log dependency-check.log', odcInstallation: 'DP-Check'
               dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
           }
       }
       stage('Infrastructure Security Scan') {
           steps {
               sh '''
                   python3 -m pip install --upgrade pip --user
                   python3 -m pip install checkov --quiet --user
                   export PATH=$PATH:$(python3 -m site --user-base)/bin
                   checkov --version || echo "Checkov installation failed"
               '''
               sh '''
                   export PATH=$PATH:$(python3 -m site --user-base)/bin
                   checkov -d . --output cli || true
               '''
           }
       }
       stage('Build Artifact') {
           steps {
               sh 'mvn -f pom.xml clean package -DskipTests -Dcheckstyle.skip'
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
               nexusUrl: 'nexus.selfdevops.space',
               nexusVersion: 'nexus3',
               protocol: 'https',
               repository: 'nexus-repo',
               version: '1.0'
           }
       }
       stage('Build docker image') {
           steps {
               sshagent (['ansible-key']) {
                   sh 'ssh -t -t ec2-user@15.237.27.148 -o strictHostKeyChecking=no "ansible-galaxy collection install community.docker"'
                   sh 'ssh -t -t ec2-user@15.237.27.148 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook /opt/docker/docker-image.yml"'
               }
           }
       }
       stage('Trivy image scan') {
           steps {
              sh "trivy image cloudhight/testapp > trivyfs.txt"
           }
       }
       stage('Trigger Ansible to deploy app') {
           steps {
               sshagent (['ansible-key']) {
                   sh 'ssh -t -t ec2-user@15.237.27.148 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook /opt/docker/docker-container.yml"'
                   sh 'ssh -t -t ec2-user@15.237.27.148 -o strictHostKeyChecking=no "cd /etc/ansible && ansible-playbook /opt/docker/newrelic-container.yml"'
               }
           }
       }
       stage('slack notification') {
           steps {
               slackSend channel: '9th-dec-jenkins-pipeline-containerization-project', message: 'Application deploy successfully ', teamDomain: 'Cloudhight', tokenCredentialId: 'slack-cred'
           }
       }
   }
}
