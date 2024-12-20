def registry = 'https://satishk.jfrog.io'
def imageName = 'satishk.jfrog.io/satish-docker-local/sample_app'
def version = '2.1.2'

pipeline {
    agent {
        node {
            label 'maven'
        }
    }
    environment {
        PATH = "/opt/apache-maven-3.9.4/bin:$PATH"
    }
    stages {
        stage("Copy Dependencies") {
            steps {
                echo "<--------------- Copying Dependencies Started --------------->"
                sh 'mvn dependency:copy-dependencies'
                echo "<--------------- Copying Dependencies Completed --------------->"
            }
        }

        stage("Build") {
            steps {
                echo "<----------- Build Started ----------->"
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                echo "<----------- Build Completed ----------->"
            }
        }

        stage("Test") {
            steps {
                echo "<----------- Unit Test Started ----------->"
                sh 'mvn surefire-report:report'
                echo "<----------- Unit Test Completed ----------->"
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'satish-sonarqube-scanner'
            }
            steps {
                withSonarQubeEnv('satish-sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage("Jar Publish") {
            steps {
                script {
                    echo "<--------------- Jar Publish Started --------------->"
                    def server = Artifactory.newServer(url: registry + "/artifactory", credentialsId: "jfrog_cred")
                    def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}"
                    def uploadSpec = """{
                        "files": [
                            {
                                "pattern": "jarstaging/(*)",
                                "target": "maven-libs-release-local/{1}",
                                "flat": "false",
                                "props": "${properties}",
                                "exclusions": ["*.sha1", "*.md5"]
                            }
                        ]
                    }"""
                    def buildInfo = server.upload(uploadSpec)
                    buildInfo.env.collect()
                    server.publishBuildInfo(buildInfo)
                    echo "<--------------- Jar Publish Ended --------------->"
                }
            }
        }

        stage("Docker Build") {
            steps {
                script {
                    echo "<--------------- Docker Build Started --------------->"
                    app = docker.build(imageName + ":" + version)
                    echo "<--------------- Docker Build Ends --------------->"
                }
            }
        }

        stage("Docker Publish") {
            steps {
                script {
                    echo "<--------------- Docker Publish Started --------------->"
                    docker.withRegistry(registry, 'jfrog_cred') {
                        app.push()
                    }
                    echo "<--------------- Docker Publish Ended --------------->"
                }
            }
        }

        stage("Deploy") {
            steps {
                script {
                    echo "<--------------- Helm Deploy Started --------------->"
                    sh 'helm install sample-app sample-app-1.0.1'
                    echo "<--------------- Helm Deploy Ends --------------->"
                }
            }
        }
    }
}
