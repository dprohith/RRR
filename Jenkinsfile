pipeline {
    agent any
    parameters {
        string(name: 'CLUSTER', defaultValue: 'Rohitrrr', description: 'ECS Cluster name')
        string(name: 'SERVICE', defaultValue: 'rohitsvc1', description: 'ECS Service name')
        string(name: 'TASK_FAMILY', defaultValue: 'Rohit-TD', description: 'ECS Task family')
        string(name: 'DESIRED_COUNT', defaultValue: '2', description: 'Desired count of ECS tasks')
    }	
    environment {
        registryCredential = 'ecr:us-east-1:jenkins'
        appRegistry = "814019578945.dkr.ecr.us-east-1.amazonaws.com/rohitrrr"
        rohitRegistry = "https://814019578945.dkr.ecr.us-east-1.amazonaws.com"
	slackChannel = '#jenkinscicd'    
    }

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "MAVEN"
    }

    stages {
        stage('Build') {
            steps {
                // Get some code from a GitHub repository
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: 'https://github.com/dprohith/RRR.git']]])

                // Run Maven on a Unix agent.
                sh "mvn clean install"

                // To run Maven on a Windows agent, use
                // bat "mvn -Dmaven.test.failure.ignore=true clean package"
            }
        }
		stage('Code Analysis with CheckStyle') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
		    post {
                success {
                    archiveArtifacts 'target/*.war'
                }
            }
        }
        stage('Sonar Scan') {
            environment {
                scannerHome = tool 'Sonar'
            }
            steps {
                withSonarQubeEnv('SonarServer') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=visualpathit \
                    -Dsonar.projectName=visualpathit \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.checkstyle.reportsPath=target/checkstyle-result.xml'''
                }
                timeout( time: 10, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage ('Docker Build') {
            steps {
                script {
                dockerimage = docker.build( appRegistry + ":${BUILD_NUMBER}", "./Docker-files/app/multistage/")
                }
            }
        }
        stage ('Push DockerImage to ECR') {
            steps {
                script {
                docker.withRegistry( rohitRegistry, registryCredential ){
                    dockerimage.push()
                }
                }
            }
        }
        stage ('Deploy app to ECS') {
            steps {
                withAWS(credentials: 'jenkins', region: 'us-east-1') {
                    sh 'aws ecs update-service --cluster ${CLUSTER} --service ${SERVICE} --task-definition ${TASK_FAMILY}:3 --desired-count ${DESIRED_COUNT}'
                }
            }
        }
        stage('Notify Slack') {
	    steps {
                echo 'Notifying Slack'
                slackSend channel: '#jenkinscicd',
                    color: '#439FE0',
                    message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n For More information see: ${env.BUILD_URL}"

            }
	}
    }
}

