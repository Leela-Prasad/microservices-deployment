node {
   stage('Preparation') { 
      git 'https://github.com/Leela-Prasad/fleetman-position-tracker'
   }
   stage('Build') {
        sh "mvn package"
   }
   stage('Results') {
      junit '**/target/surefire-reports/TEST-*.xml'
      archive 'target/*.jar'
   }
   stage('Deploy') {
      withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'AWS Credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        ansiblePlaybook credentialsId: 'SSH-Credentails', playbook: 'deploy.yaml' 
      }
   }
}
