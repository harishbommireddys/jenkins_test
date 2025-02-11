pipeline {
    agent any

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "MVN3"
    }

    stages {
        stage('Build') {
            steps {
                // Get some code from a GitHub repository
                git credentialsId: 'git', url: 'https://github.com/harishbommireddys/jenkins_test.git'

                // Run Maven on a Unix agent.
                sh "mvn -Dmaven.test.failure.ignore=true -f api-gateway clean package"

                // To run Maven on a Windows agent, use
                // bat "mvn -Dmaven.test.failure.ignore=true clean package"
            }//steps

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                    junit 'api-gateway/target/surefire-reports/TEST-*.xml'
                    archiveArtifacts 'api-gateway/target/*.jar'
                }//sucess
            }//post
        }//stage
    }//
}
