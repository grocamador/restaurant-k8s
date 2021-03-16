pipeline {
    agent any
    
    environment {
        //be sure to replace "grocamador" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "grocamador/train-schedule"
    //     CHKP_CLOUDGUARD_ID = credentials("chkp-id")
    //    CHKP_CLOUDGUARD_SECRET = credentials("chkp-key")
        SG_CLIENT_ID = credentials("source-id")
        SG_SECRET_KEY = credentials("source-key")
        KUBECONFIG = credentials("my-kubeconfig")

        }
    
    stages {
       stage('ShiftLeft secure Code Scan') {   
            steps {   
            echo 'Scan of code source'    
                    script {      
                        try {
                            sh 'ls'
                            sh 'chmod +x shiftleft' 
                            sh './shiftleft code-scan -s . -x graddle/**'
                        } catch (Exception e) {
                            input "Code scan showed some security issues, Are you sure you want to continue?"  
                        }
                   }
            }
         
         }
        
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
            stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                    
                    }
                }
            }
        }
        stage('Image Assurance scanning') {   
            steps {   
            echo 'Image vulnerability scanning'    
                   script {      
                        try {
                            sh "docker pull grocamador/train-schedule:${env.BUILD_NUMBER}"
                            sh "docker save -o train-schedule.tar grocamador/train-schedule:${env.BUILD_NUMBER}"
                            sh 'chmod +x shiftleft' 
                            sh './shiftleft image-scan -i train-schedule.tar'
                         
                        } catch (Exception e) {
                            input "Image scan found vulnerabilities, Are you sure you want to continue?"  
                        }
                   }
            }
         
         }
        stage('Push Docker Image to latest') {
            when {
                branch 'master'
            }
            steps {
                script {
                    
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("latest")
                    }
                }
            }
        }
        stage('Clean') {
        steps{
            echo 'cleaning up artifacts'
                script{
                try{
                sh 'rm train-schedule.tar'
                echo 'Image deleted'
                } catch (Exception e) {
                echo 'Already deleted'  
                }
                }
            }
        }
        stage('Deploy to stage') {
            when {
                branch 'master'
            }
            steps {
            sh ("""     
                  kubectl delete -f train-schedule-kube-stage.yml
                  kubectl apply -f train-schedule-kube-stage.yml
                """)
 
                // Other tentative that didnt work:
 //            kubectl set image deployment/train-schedule-deployment-stage train-schedule-stage=grocamador/train-schedule
 //            kubectl set image deployment/train-schedule-deployment-stage train-schedule-stage=grocamador/train-schedule:latest
            }
        }
        
       stage("Deploy to Production"){
            when {
                branch 'master'
            }
             steps {              
                input 'Deploy to Production?'
                milestone(1)
//              With KUBECTL and Kubeconfig       
              sh ("""                
                  echo \$KUBECONFIG
                  kubectl delete -f train-schedule-kube.yml
                  kubectl apply -f train-schedule-kube.yml
                """)

                 
//                 kubernetesDeploy(
//                    kubeconfigId: 'kubeconfig',
//                    configs: 'deploy.yml',
//                    enableConfigSubstitution: true
//                )

                
             }
         }
    }
}
