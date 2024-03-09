pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {                        
            steps {                                       
                git branch: 'main', url: 'https://github.com/yash509/DevSecOps-Project----Architect-Comp-Deployment.git'
            }
        }
        stage('deployments') {
            parallel {
                stage('deploy to stg') {
                    steps {
                        echo 'stg deployment done'
                    }
                }
                stage('deploy to prod') {
                    steps {
                        echo 'prod deployment done'
                    }
                }
            }
        }
        stage('Test Build') {
            steps {
                echo 'Building....'
            }
        }
        stage("Sonarqube Analysis") {                         
            steps {                                           
                withSonarQubeEnv('sonar-server') {           
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=arch \
                    -Dsonar.projectKey=arch'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Tag"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t arch-comp ." 
                       sh "docker tag arch-comp yash5090/arch-comp:latest " 
                    }
                }
            }
        }
        stage("Push to DockerHub") {
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker push yash5090/arch-comp:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image yash5090/arch-comp:latest > trivyimage.txt"   
            }
        }
        stage ('Manual Approval'){
          steps {
           script {
             timeout(time: 10, unit: 'MINUTES') {
              approvalMailContent = """
              Project: ${env.JOB_NAME}
              Build Number: ${env.BUILD_NUMBER}
              Go to build URL and approve the deployment request.
              URL de build: ${env.BUILD_URL}
              """
             mail(
             to: 'clouddevopshunter@gmail.com',
             subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", 
             body: approvalMailContent,
             mimeType: 'text/plain'
             )
            input(
            id: "DeployGate",
            message: "Deploy ${params.project_name}?",
            submitter: "approver",
            parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')])  
            }
           }
          }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name arch-comp -p 5000:80 yash5090/arch-comp:latest' 
            }
        }
        stage('Deployment Done') {
            steps {
                echo 'Deployed Succcessfully...'
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
            "Build Number: ${env.BUILD_NUMBER}<br/>" + 
            "URL: ${env.BUILD_URL}<br/>",
            to: 'clouddevopshunter@gmail.com',
            attachmentsPattern: 'trivy.txt'
        }
    }
}
stage('Result') {
  timeout(time: 10, unit: 'MINUTES') {
    mail to: 'clouddevopshunter@gmail.com',
         subject: "${currentBuild.result} CI: ${env.JOB_NAME}",
         body: "Project: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nGo to ${env.BUILD_URL} and approve deployment"
    input message: "Deploy ${params.project_name}?", 
           id: "DeployGate", 
           submitter: "approver", 
           parameters: [choice(name: 'action', choices: ['Success'], description: 'Approve deployment')]
  }
}
