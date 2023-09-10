pipeline {
    agent {
        label 'maven-slave'
        // label 'node-slave || java-slave'
    } 
    parameters {
        choice(name: 'scanOnly',
            choices: 'no\nyes',
            description: 'This will scan the application'
        )
        choice(name: 'buildOnly',
            choices: 'no\nyes',
            description: 'This will only build the Application'
        )
        choice(name: 'dockerPush',
            choices: 'no\nyes',
            description: 'This will trigger app build, docker build and docker push '
        )
        choice(name: 'deployToDev',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Dev env'
        )
        choice(name: 'deployToTest',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Test env'
        )
        choice(name: 'deployToStage',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Stage env'
        )
        choice(name: 'deployToProd',
            choices: 'no\nyes',
            description: 'This will Deploy the app to Prod env'
        )
    }
    environment {
        APPLICATION_NAME = "i27-product"
        GIT_CREDS = credentials('devopsjobready_git_creds')
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_CREDS = credentials('dockerhub_creds')
        DOCKER_HUB = "docker.io/devopswithcloudhub"
        DOCKER_REPO = "jenkinsspring"
        USER_NAME = "devopswithcloudhub"
        SONAR_URL = "http://34.125.152.223:9000"
        //SONAR_TOKEN = "squ_72217d0f24dcfcf4859078eea362807f8bc34911"
        SONAR_TOKEN = credentials('sonar_creds')
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    stages {
        stage ('Build') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                        params.buildOnly == 'yes'
                    }
                }
            }
            // build happens here 
            steps {
                script { //learner-eureka-0.0.1-SNAPSHOT.jar
                    //git branch: ${BRANCH_NAME}, credentialsId: 'learner_siva_git_creds', url: 'https://github.com/varresiva/learner-eureka.git'
                    buildApp().call()
                }
            }
        } /* 
        stage ('Unit Tests- Junit and Jacoco') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                        params.buildOnly == 'yes'
                    }
                }
            }
            steps {
                echo "Performing Unit tests for ${env.APPLICATION_NAME} application"
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }*/
        stage ('Sonar') {
            when {
                anyOf {
                    expression {
                        params.scanOnly == 'yes'
                       // params.dockerPush == 'yes'
                        //params.buildOnly == 'yes'
                    }
                }
            }
            steps {
                // working sonar belo
                // -Dsonar.login=sqa_1edfe4616b6cd55719d9842ef754a79d38f809e5
                echo "Starting Sonar Scan"
                withSonarQubeEnv('i27-sonar-server') {
                    sh """
                    echo "Starting Sonar Scan"
                    mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=i27-eureka \
                        -Dsonar.host.url=${env.SONAR_URL} \
                        -Dsonar.login=${env.SONAR_TOKEN}
                """
                }
                timeout (time: 2, unit: 'MINUTES') {
                    script {
                        waitForQualityGate abortPipeline: true
                    }
                }

            }
        }
        stage ('Docker Build and Push') {
            when {
                anyOf {
                    expression {
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps {
                //Building jar: 
                // /home/maha/jenkins/workspace/i27-apis_master/target/learner-eureka-0.0.1-SNAPSHOT.jar
                script {
                    dockerBuildandPush().call() 
                }
            }
        }
        stage ('Deploy to Dev'){
            when {
                expression {
                    params.deployToDev == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('dev', '5132', '8132').call()
                    // dockerDeploy('dev', 'hostPort', 'contPort).call()
                }
            }
        }
        stage ('Deploy to Test'){
            when {
                expression {
                    params.deployToTest == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('tst', '6132', '8132').call()
                }
            }
        }
        stage ('Deploy to Stage'){
            when {
                expression {
                    params.deployToStage == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('stage', '7132', '8132').call()
                }
            }
        }
        stage ('Deploy to Prod'){
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                        }
                    }
                    anyOf {
                            branch 'release/*'
                    }
                }

            }
            steps {
                script {
                    imageValidation().call()
                    dockerDeploy('prod', '8132', '8132').call()
                }
            }
        }
        stage('Clean') {
            steps {
                cleanWs()
            }
        }
    }
}

def dockerDeploy(envDeploy, hostPort, contPort) { // dockerDeploy('dev', 'hostPort', 'contPort).call()
    return {
        echo "Deploying to $envDeploy env"
        echo "Docker hub is ${env.DOCKER_HUB}"
        withCredentials([usernamePassword(credentialsId: 'maha_user_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
            script {
                sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip \"docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT\""
                try {
                // If we execute the below command without try block it will fail for the first time, and if container is not avilable even
                echo "Stopping the Container"
                sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip \"docker stop ${env.APPLICATION_NAME}-$envDeploy\""
                echo "Docker removing the Container"
                sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip \"docker rm ${env.APPLICATION_NAME}-$envDeploy\""
                } catch (err) {
                    echo: 'Caught the error: $err'
                }
                sh "sshpass -p '$PASSWORD' -v ssh -o StrictHostKeyChecking=no $USERNAME@$dev_ip \"docker run --restart always --name ${env.APPLICATION_NAME}-$envDeploy -p $hostPort:$contPort -d ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT\""
            }
        }
    }
}

def imageValidation () {
    return {
        println("Pulling the docker image")
        try {
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT"
        }
        catch (Exception e) {
            println('OOPS, docker image with this tag is not available')
            buildApp().call()
            dockerBuildandPush().call()
        }
    }
}

def dockerBuildandPush() {
    return {
        echo "********************** Building Docker Image***************************"
        echo "whoami"
        sh "whoami"
        sh "hostname -i"
        sh "cp ${workspace}/target/${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd"
        sh "sudo docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} --build-arg JAR_DEST=${env.APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${env.POM_PACKAGING} \
            -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT  ./.cicd"
        echo "Pushing the image to repo"
        echo "******** Logging to Docker Registry********"
        sh "docker login ${env.DOCKER_HUB} -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:$GIT_COMMIT"
    }
}
def buildApp() {
    return {
        echo "Building the ${env.APPLICATION_NAME} application"
        sh 'mvn clean package -DskipTests=true'
        archive 'target/*.jar'
    }
}
