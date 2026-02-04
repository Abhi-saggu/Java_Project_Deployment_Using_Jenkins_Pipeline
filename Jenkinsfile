def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline
{
    agent any

    tools
    {
        maven "MAVEN3.9"
        jdk "JDK21"
    }

    environment
    {
        SONAR_PROJECT_KEY = 'DAV_College'
        SONAR_PROJECT_NAME = 'DAV_College'
        EC2_USER='ubuntu'
        EC2_HOST='13.203.36.250'
        EC2_KEY_PATH='/var/lib/jenkins/.ssh/Deploy_WebServer_Key.pem'
        TOMCAT_WEBAPP_DIR='/var/lib/tomcat10/webapps'
    }

    stages
    {
        stage("Fetch Code")
        {
            steps
            {
                git branch: 'master', url: 'https://github.com/Abhi-saggu/Java_Project_Deployment_Using_Jenkins_Pipeline.git'
            }
        }

        stage("Unit Test")
        {
            steps
            {
                sh 'mvn test'
            }
        }

        stage("Code Build")
        {
            steps
            {
                sh 'mvn install -DskipTests'
            }
            post
            {
                success
                {
                    echo "Build Success..."
                    archiveArtifacts artifacts: '**/*.war'
                }
                failure
                {
                    echo "Build Failure..."
                }
            }
        }

        stage("Checkstyle Analysis")
        {
            steps
            {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage("Sonarqube upload")
        {
            steps
            {
                withSonarQubeEnv('SonarServer')
                {
                    timeout(time: 10, unit: 'MINUTES')
                    {
                        sh """
                        ${tool 'Sonar8.0'}/bin/sonar-scanner \
                        -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                        -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.checkstyle.reportPaths=target/checkstyle-result.xml
                        """
                    }
                }
            }
        }

        stage("Quality Gate")
        {
            steps
            {
                script
                {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK')
                    {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                    else
                    {
                        echo "Quality Gate Passed"
                    }
                }
            }
        }

        stage("Artifact Upload")
        {
            steps
            {
               nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol:'http',
                    nexusUrl: '13.235.182.17:8081',
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofilerpo',
                    credentialsId: 'nexuslogin',
                    artifacts: [[
                        artifactId: 'vproapp',
                        classifier: '',
                        file: 'target/vprofile-v2.war',
                        type: 'war'
                    ]]
                )
            }
        }

        stage("Deploy To Server")
        {
            steps
            {
                script
                {
                    sh 'echo "================= Deploying To EC2 ================="'
                    def artifactFile = 'target/*.war'
                    def artifact=findFiles(glob:artifactFile)[0]
                    def artifactName=artifact.name
                    def artifactPath=artifact.path
                    def renameartifactPath="target/ROOT.war"

                    echo "Deploying ${renameartifactPath} To EC2 Server"

                    //Rename only if it not already ROOT.war
                    if(artifactName!='ROOT.war')
                    {
                        echo "Renaming ${artifactName} to ROOT.war"
                        sh "cp ${artifactPath} ${renameartifactPath}"
                    }
                    else
                    {
                        renameartifactPath=artifactPath
                        echo "Arifact Already Named Root.war no need to Change"
                    }

                    sh"""
                        ssh -o StrictHostKeyChecking=no -i ${EC2_KEY_PATH} ${EC2_USER}@${EC2_HOST} 'sudo rm -rf ${TOMCAT_WEBAPP_DIR}/ROOT'
                        scp -o StrictHostKeyChecking=no -i ${EC2_KEY_PATH}  ${renameartifactPath} ${EC2_USER}@${EC2_HOST}:${TOMCAT_WEBAPP_DIR}/ROOT.war
                        ssh -o StrictHostKeyChecking=no -i ${EC2_KEY_PATH} ${EC2_USER}@${EC2_HOST} 'sudo systemctl restart tomcat10'
                    """
                }
            }
        }
    }

    post
    {
        always
        {
            script
            {
                echo 'Slack Notification'
                slackSend(
                    channel: '#all-chandigarhgovservice',
                    color:COLOR_MAP[currentBuild.currentResult],
                    message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More Information at : ${env.BUILD_URL}"
                )
            }
        }
    }
}
