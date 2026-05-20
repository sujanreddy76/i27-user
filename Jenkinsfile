//This Jenkinsfile is for i27-user Deployment
pipeline{
    agent{
        label 'k8s-slave'
    }
    // { choice(name: 'CHOICES', choices: ['one', 'two'], description: '') }
    parameters{
        choice(name: 'buildOnly',
           choices: 'no\nyes',
           description: 'This will only build the application'
        )
        choice(name: 'sonarScanOnly',
           choices: 'no\nyes',
           description: 'This will scan the application'
        )
        choice(name: 'dockerPush',
           choices: 'no\nyes',
           description: 'This will trigger the app build, docker build and docker push'
        )
        choice(name: 'deployToDev',
           choices: 'no\nyes',
           description: 'This will deploy the application to Dev env'
        )
        choice(name: 'deployToTest',
           choices: 'no\nyes',
           description: 'This will deploy the application to Test env'
        )
        choice(name: 'deployToStage',
           choices: 'no\nyes',
           description: 'This will deploy the application to Stage env'
        )
        choice(name: 'deployToProd',
           choices: 'no\nyes',
           description: 'This will deploy the application to Prod env'
        )
    }
    environment{
        APPLICATION_NAME = "user"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB = "docker.io/sujanreddy76"
        DOCKER_CREDS = credentials('dockerhub_creds') // username and password
    }
    //tools locations configured in jenkins-master
    tools{
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    stages{
        stage('Build'){
            when {
                anyOf {
                    expression { params.buildOnly == 'yes'}
                    expression { params.sonarScanOnly == 'yes' } 
                    expression { params.dockerPush == 'yes' }               
                }
            }
            //This is where Build for user application happens
            steps{
                echo "Building ${env.APPLICATION_NAME} Application"
                // -DskipTests=true → Compiles test classes but skips test execution, whereas -Dmaven.test.skip=true → skips both test compilation and test execution.
                sh 'mvn clean package -DskipTests=true' //(or) mvn clean package -Dmaven.test.skip=true
                archiveArtifacts artifacts: 'target/*.jar'
            }
        }
        // stage('Unit Tests'){
        //     steps{
        //         echo "********* Performing Unit Tests for ${env.APPLICATION_NAME} Application ********"
        //         sh 'mvn test'
        //     }
        // }
        stage('SonarQube') {
            when {
                anyOf {
                    expression {
                        params.sonarScanOnly == 'yes' ||
                        params.dockerPush == 'yes'
                    }
                }
            }
            steps{
                //Code Quality needs to be implemented in this stage
                //Before we execute or write the code, make sure sonarqube-scanner plugin is installed in jenkins.
                // and sonar details are configured in the manage jenkins->system
                echo "********* Starting Sonar Scans with Quality Gates *********"
                withSonarQubeEnv('SonarQube') { // 'SonarQube' is the name we configured in Manage Jenkins > system > sonarqube , (It should match exactly)
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=i27-eureka \
                            -Dsonar.host.url=http://34.16.53.216:9000 \
                            -Dsonar.login=sqa_59d0ea7929b3c7b27378c5dabe539ddb896a39a5
                """
                } 
                timeout(time: 2, unit: 'MINUTES') { //SECONDS, MINUTES, HOURS, DAYS 
                      waitForQualityGate abortPipeline: true
                }
                    
            }
        }
        // stage('BuildFormat') {
        //     steps{
        //         script {
        //             //Existing: i27-eureka-0.0.1-SNAPSHOT.jar
        //             //Destination: 127-eureka-buildNumber-branchName.jar(packaging)
        //             sh """
        //               echo "Testing JAR Source: i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING}"
        //               echo "Testing JAR Destination Format: i27-${env.APPLICATION_NAME}-${currentBuild.number}-${BRANCH_NAME}.${env.POM_PACKAGING}"
        //             """
        //         }
        //     }
        // }
        stage('Docker Build and Push'){
            when {
                expression {
                        params.dockerPush == 'yes'
                    }
            }
            steps{
                script{
                    dockerBuildAndPush().call()

                }
            }
        }
        stage('Deploy to Dev Env'){
            when {
                expression {
                        params.deployToDev == 'yes'
                    }
            }
            steps{
                script{
                    imageValidation().call()
                    dockerDeploy('dev', '5232', '8232').call()
                }

            }
            // a mail should trigger based on the status(success or failure)
            // Jenkins URL should be sent as an Email.
        }
        stage('Deploy to Test Env'){
            when {
                expression {
                        params.deployToTest == 'yes'
                    }
            }
            steps{
                script{
                    imageValidation().call()
                    dockerDeploy('tst', '6232', '8232').call()
                }

            }
        }
        stage('Deploy to Stage Env'){
            when {
                allOf {
                    expression {
                        params.deployToStage == 'yes'
                    }
                    anyOf{
                        branch 'release/*'
                        tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP"
                    }

                }
            }
            steps{
                script{
                    imageValidation().call()
                    dockerDeploy('stg', '7232', '8232').call()
                }

            }
        }
        stage('Deploy to Prod Env'){
            when {
                allOf {
                    expression {
                        params.deployToProd == 'yes'
                    }
                    tag pattern: "v\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}", comparator: "REGEXP" //v1.2.3 is the correct one, v123 is the wrong one.
                }
            }
            steps{
                timeout(time: 300, unit: 'SECONDS'){ //SECONDS, MINUTES, HOURS
                    input message: "Deploying ${env.APPLICATION_NAME} to production??", ok:'yes', submitter: 'sujanSRE,sivaTechlead'
                }
                script{
                    dockerDeploy('prod', '8232', '8232').call()
                }

            }
        }
    }
}

//Method for Application Build(.jar)
def buildApp(){
    return{
        echo "Building ${env.APPLICATION_NAME} Application"
        // -DskipTests=true → Compiles test classes but skips test execution, whereas -Dmaven.test.skip=true → skips both test compilation and test execution.
        sh 'mvn clean package -DskipTests=true' //(or) mvn clean package -Dmaven.test.skip=true
        archiveArtifacts artifacts: 'target/*.jar'        
    }
}

//Method for Docker build and push
def dockerBuildAndPush(){
    return {
        echo "********** Building Docker Image ****************"
        sh "cp ${WORKSPACE}/target/i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd"
        sh "docker build --no-cache --build-arg JAR_SOURCE=i27-${env.APPLICATION_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} ./.cicd/" //docker.io/sujanreddy76/eureka:commitid
        echo "*********** Login to Docker Registry **************"
        sh "docker login -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}"
        echo "*********** Push Image to Docker Registry **************"
        sh "docker push ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
    }
}

//Method for imageValidation
def imageValidation(){
    return {
        println("***** Attempting to pull the docker image *******")
        try {
            sh "docker pull ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}"
            println("******** Image is pulled successfully********")
        }
        catch(Exception e) {
            println("******* OOPS, the docker image with this name: ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT} not available in the repo, so building application, creating docker image and pushing the image **********")
            buildApp().call()
            dockerBuildAndPush().call()
        }
    }
}

//Method for Docker Deployment as container to different environment
def dockerDeploy(envDeploy, hostPort, contPort){
    return{
        echo "*************** Deploying to $envDeploy Environment ****************"
        withCredentials([usernamePassword(credentialsId: 'john_docker_vm_password', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                // we will communicate to the server
                script{
                    try{
                        //stop the container
                        sh "sshpass -v -p '$PASSWORD' ssh -o StrictHostKeyChecking=no '$USERNAME'@$dev_ip \"docker stop ${env.APPLICATION_NAME}-$envDeploy\""

                        //remove the container
                        sh "sshpass -v -p '$PASSWORD' ssh -o StrictHostKeyChecking=no '$USERNAME'@$dev_ip \"docker rm ${env.APPLICATION_NAME}-$envDeploy\""
                    }
                    catch(err){
                        echo "Error Caught: $err"
                    }
                    //command syntax to use sshpass:
                    //sshpass -p !4u2tryhack ssh -o StrictHostKeyChecking=no username@host.example.com
                    // Create container
                    sh "sshpass -v -p '$PASSWORD' ssh -o StrictHostKeyChecking=no '$USERNAME'@$dev_ip \"docker container run -dit -p $hostPort:$contPort --name ${env.APPLICATION_NAME}-$envDeploy ${env.DOCKER_HUB}/${env.APPLICATION_NAME}:${GIT_COMMIT}\""
                }
            }
    }
}

//For user lets use the below port numbers
// Container port will be 8232 , only hostport changes
//dev: HostPort = 5232
//tst: HostPort = 6232
//stg: HostPort = 7232
//prod: HostPort = 8232
