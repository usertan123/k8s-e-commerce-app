@Library('Shared') _

pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE_NAME = 'tanmaytech/easyshop-app'
        DOCKER_MIGRATION_IMAGE_NAME = 'tanmaytech/easyshop-migration'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CREDENTIALS = credentials('github-credentials')
        GIT_BRANCH = "main"
        SONAR_HOME = tool "Sonar"
    }
    
    stages {
        stage("Workspace cleanup"){
            steps{
                script{ clean_ws() }
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                script{ code_checkout("https://github.com/usertan123/k8s-e-commerce-app.git","main") }
            }
        }

        stage("Trivy: Filesystem scan") {
            steps { script { trivy_scan(scanType: "fs", path: ".") } }
        }

        stage("OWASP: Dependency check") {
            steps { script { owasp_dependency() } }
        }

        stage("SonarQube: Code Analysis") {
            steps { script { sonarqube_analysis("Sonar","easyshop","easyshop") } }
        }

        stage("SonarQube: Code Quality Gates") {
            steps { script { sonarqube_code_quality() } }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Main App Image') {
                    steps { script { docker_build(imageName: env.DOCKER_IMAGE_NAME,imageTag: env.DOCKER_IMAGE_TAG,dockerfile: 'Dockerfile',context: '.') } }
                }
                stage('Build Migration Image') {
                    steps { script { docker_build(imageName: env.DOCKER_MIGRATION_IMAGE_NAME,imageTag: env.DOCKER_IMAGE_TAG,dockerfile: 'scripts/Dockerfile.migration',context: '.') } }
                }
            }
        }

        stage("Trivy: Docker Image Scan") {
            parallel {
                stage("Scan Main App Image") {
                    steps { script { trivy_scan(scanType: "image", imageName: env.DOCKER_IMAGE_NAME, imageTag: env.DOCKER_IMAGE_TAG) } }
                }
                stage("Scan Migration Image") {
                    steps { script { trivy_scan(scanType: "image", imageName: env.DOCKER_MIGRATION_IMAGE_NAME, imageTag: env.DOCKER_IMAGE_TAG) } }
                }
            }
        }

        stage('Run Unit Tests') {
            steps { script { run_tests(testCommand: "npm test") } }
        }
        
        stage('Push Docker Images') {
            parallel {
                stage('Push Main App Image') {
                    steps { script { docker_push(imageName: env.DOCKER_IMAGE_NAME, imageTag: env.DOCKER_IMAGE_TAG, credentials: 'docker') } }
                }
                stage('Push Migration Image') {
                    steps { script { docker_push(imageName: env.DOCKER_MIGRATION_IMAGE_NAME, imageTag: env.DOCKER_IMAGE_TAG, credentials: 'docker') } }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    update_k8s_manifests(
                        imageTag: env.DOCKER_IMAGE_TAG,
                        manifestsPath: 'kubernetes',
                        gitCredentials: 'github-credentials',
                        gitUserName: 'usertan123',
                        gitUserEmail: 'tan2018carlson@gmail.com'
                    )
                }
            }
        }
    }

    post {
        success {
            emailext (
                subject: "✅ Jenkins Build #${env.BUILD_NUMBER} - SUCCESS",
                body: """
                    <h2 style="color:green;">Build Successful 🎉</h2>
                    <p>Job: ${env.JOB_NAME}<br>
                    Build Number: ${env.BUILD_NUMBER}<br>
                    Status: <b>${currentBuild.currentResult}</b></p>

                    <h3>Attached Reports:</h3>
                    <ul>
                      <li>Trivy Scan Report</li>
                      <li>OWASP Dependency Check Report</li>
                    </ul>
                """,
                to: "tan2018carlson@gmail.com",
                attachmentsPattern: "trivy-*.json, **/dependency-check-report.xml"
            )
        }

        failure {
            emailext (
                subject: "❌ Jenkins Build #${env.BUILD_NUMBER} - FAILED",
                body: """
                    <h2 style="color:red;">Build Failed 🚨</h2>
                    <p>Job: ${env.JOB_NAME}<br>
                    Build Number: ${env.BUILD_NUMBER}<br>
                    Status: <b>${currentBuild.currentResult}</b></p>

                    <h3>Attached Reports:</h3>
                    <ul>
                      <li>Trivy Scan Report</li>
                      <li>OWASP Dependency Check Report</li>
                    </ul>
                """,
                to: "tan2018carlson@gmail.com",
                attachmentsPattern: "trivy-*.json, **/dependency-check-report.xml"
            )
        }
    }
}
