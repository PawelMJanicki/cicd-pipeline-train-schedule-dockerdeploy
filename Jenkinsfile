pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master' //only for the master branch
            }
            steps {
                script { //allows to do performs a scripts style even in declarative pipeline. Dockerfile must be created
                    app = docker.build("pawelmjanicki/train-schedule") //scripted style needed as Docker plugin doesn't support fully declarative approach
                    app.inside {
                        sh 'echo $(curl localhost:8080)' //execute comamnd within the Docker container craeted from the build. This is smoketest
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
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') { //global credentails called docker_hub_login must be craeted on Jenkins site
                        app.push("${env.BUILD_NUMBER}") //give version number for an each push - tag
                        app.push("latest") //latest tag added to the newest push
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?' //pause for an approval
                milestone(1) //safeguard for an deploying old version over the new one
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) { //Global credentials called webserver_login must existing on Jenkins site
                    script {
                      // sshpass executes one command over SSH. This steps pull the newest image from the DockerHub
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull pawelmjanicki/train-schedule:${env.BUILD_NUMBER}\"" //prod_ip global variable must be set in Configure Jenkins section in Manage Jenkins
                        // stoping old container from older version of image and removing it. For the first launch catch block will be executed, also it will be executed if containers will be stopped and deleted manually
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        //Running container from the newer image
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d pawelmjanicki/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
