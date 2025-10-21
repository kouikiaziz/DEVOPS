def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
    agent any
    tools {
        maven 'M2_HOME'
        jdk 'JAVA_HOME'
    }
    environment {

        SCANNER_HOME = tool 'sonarqube'

        NEXUS_VERSION = 'nexus3'
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "192.168.56.10:8081"
        NEXUS_REPOSITORY = "vprofile-repo"
        NEXUS_REPO_ID = "vprofile-repo"
        NEXUS_CREDENTIAL_ID = "sonatype-credentials"
        ARTVERSION = "${env.BUILD_ID}"

    }
    
    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/kouikiaziz/DEVOPS.git'
            }
        }

        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
        
        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
                
            }
        }




//  stage("Quality Gate") {
//     steps {
//         script {
//             timeout(time: 1, unit: 'MINUTES') {
//                 waitForQualityGate abortPipeline: false, credentialsId: 'sonarqubeToken'
//             }
//         }
//     }
// }



    stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml"
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}")
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path
                    artifactExists = fileExists artifactPath
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION"
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        )
                    } else {
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }


        stage("OWASP Dependency Check Scan") {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan . 
                    --disableYarnAudit 
                    --disableNodeAudit 
                ''',
                odcInstallation: 'dp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }



stage("Build Docker Image") {
    steps {
        script {
            // ✅ Use default values if variables are missing
            def imageName = "myappdevops"
            def buildNumber =  "local"

            env.IMAGE_TAG = "${imageName}:${buildNumber}"

//remove old images if there.
            sh "docker stop ${imageName} || true"
            sh "docker remove ${imageName} || true"
            sh "docker image remove ${imageName}:${buildNumber} || true"
            sh "docker image remove ${imageName}:latest || true"
//build new image
//docker tag ${imageName}:latest ${env.IMAGE_TAG}
            sh """
                echo '🧱 Building Docker image locally...'
                docker build -t ${imageName}:local .
                
                echo "✅ Built local image: ${env.IMAGE_TAG}"
            """
        }
        
    }
}

        stage("Trivy Scan Image") {
            steps {
                script {

                    def imageName = "myappdevops"
                    def buildNumber =  "local"
                    env.IMAGE_TAG = "${imageName}:${buildNumber}"

                    sh """
                    echo '🔍 Running Trivy scan on ${env.IMAGE_TAG}'
                    trivy image -f json -o trivy-image.json ${env.IMAGE_TAG}
                    trivy image -f table -o trivy-image.txt ${env.IMAGE_TAG}
                    """
                }
            }
        }
        

       

        stage("Deploy to Container") {
            steps {
                script {
                    sh "docker rm -f myappdevops || true"
                    sh "docker run -d --name myappdevops -p 80:8080 ${env.IMAGE_TAG}"
                }
            }
        }

        stage("DAST Scan with OWASP ZAP") {
            steps {
                script {
                    echo '🔍 Running OWASP ZAP baseline scan...'

                    // Run ZAP but ignore exit code
                    def exitCode = sh(script: '''
                        docker run --rm --user root --network host -v $(pwd):/zap/wrk:rw \
                        -t zaproxy/zap-stable zap-baseline.py \
                        -t http://localhost \
                        -r zap_report.html -J zap_report.json
                    ''', returnStatus: true)

                    echo "ZAP scan finished with exit code: ${exitCode}"

                    // Read the JSON report if it exists
                    if (fileExists('zap_report.json')) {
                        def zapJson = readJSON file: 'zap_report.json'

                        def highCount = zapJson.site.collect { site ->
                            site.alerts.findAll { it.risk == 'High' }.size()
                        }.sum()

                        def mediumCount = zapJson.site.collect { site ->
                            site.alerts.findAll { it.risk == 'Medium' }.size()
                        }.sum()

                        def lowCount = zapJson.site.collect { site ->
                            site.alerts.findAll { it.risk == 'Low' }.size()
                        }.sum()

                        echo "✅ High severity issues: ${highCount}"
                        echo "⚠️ Medium severity issues: ${mediumCount}"
                        echo "ℹ️ Low severity issues: ${lowCount}"
                    } else {
                        echo "ZAP JSON report not found, continuing build..."
                    }
                }
            }
            post {
                always {
                    echo '📦 Archiving ZAP scan reports...'
                    archiveArtifacts artifacts: 'zap_report.html,zap_report.json', allowEmptyArchive: true
                }
            }
        }


    }
    
    post {
        always {
            script {
                // 🔹 Common values
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'GitHub User'
                def buildUrl = "${env.BUILD_URL}"

                // 📧 Email Notification
            emailext (
                subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """                                   
                    <p>Maven App-tier DevSecops CICD pipeline status.</p>
                    <p>Project: ${env.JOB_NAME}</p>
                    <p>Build Number: ${env.BUILD_NUMBER}</p>
                    <p>Build Status: ${buildStatus}</p>
                    <p>Started by: ${buildUser}</p>
                    <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'kouikiaziz@gmail.com',
                from: 'kouikiaziz@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivyfs.txt,trivy-image.json,trivy-image.txt,dependency-check-report.xml,zap_report.html,zap_report.json'
                    )
            }
        }
    }

}