pipeline {
    agent { label 'built-in' }
    environment {
        SERVICES = getServices()
        REGISTRY_URL = "docker.io"
        DOCKER_IMAGE_BASENAME = "tnjenkin"
    }

    stages {
        stage('Checkout source') {
            steps {
                checkout scm

                script {
                    try {
                        if (env.BRANCH_NAME == 'main') {
                            sh 'git fetch --tags'
                            
                            def commitId = sh(script: "git rev-parse --short HEAD || true", returnStdout: true).trim()
                            def tag = sh(script: "git tag --contains ${commitId} || true", returnStdout: true).trim()
                            if (tag) {
                                env.GIT_TAG = tag
                                echo "Found Tag: ${env.GIT_TAG}"
                            } else {
                                env.GIT_TAG = "latest"
                            }
                        } else {
                            error("Failed to determine tag")
                        }
                    } catch (Exception e) {
                        error("Failed to determine tag : ${e.getMessage()}")
                    }
                }
            }
        }


        stage('Detect Changes') {
            steps {
                script {
                    env.SERVICES = getServices()
                    if (env.SERVICES == "NONE") {
                        echo "No relevant changes detected. Skipping build."
                        error("No relevant changes detected")
                    } else {
                        echo "Detected changes in services: ${env.SERVICES}"
                    }
                }
            }
        }
        stage('Run Tests') {
            when {
                expression { env.SERVICES.trim() && env.SERVICES != "NONE" }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    def parallelTests = [:]

                    services.each { service ->
                        parallelTests[service] = {
                            stage("Test: ${service}") {
                                try {
                                    echo "🔍 Running tests for ${service}"
                                    // Chạy test với maven, kích hoạt báo cáo test và coverage
                                    sh "mvn test -pl ${service} -Djacoco.skip=false"
        
                                    // Archive báo cáo test JUnit để Jenkins có thể hiển thị
                                    junit "${service}/target/surefire-reports/*.xml"
        
                                    // Publish báo cáo coverage, ví dụ dùng JaCoCo plugin
                                    jacoco execPattern: "${service}/target/jacoco.exec",
                                           classPattern: "${service}/target/classes",
                                           sourcePattern: "${service}/src/main/java",
                                           exclusionPattern: '**/*Test*'
        
                                    // Kiểm tra coverage threshold thủ công (giả định xuất ra file coverage.txt)
                                    def coverageStr = sh(
                                        script: "grep -Po '(?<=INSTRUCTION COVERAGE: )\\d+(\\.\\d+)?' ${service}/target/site/jacoco/index.html || echo 0",
                                        returnStdout: true
                                    ).trim()
        
                                    def coverage = coverageStr.toFloat()
                                    echo "Coverage for ${service}: ${coverage}%"
        
                                    if (coverage < 70) {
                                        error("Coverage for ${service} is below 70%")
                                    }

                                } catch (Exception e) {
                                    echo "❌ Tests failed for ${service}: ${e.getMessage()}"
                                    error("Tests failed for ${service}")
                                }
                            }
                        }
                    }
                parallel parallelTests
        }
    }
}


        stage('Build Services') {
            when {
                expression { env.SERVICES.trim() }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    def parallelBuilds = [:]

                    services.each { service ->
                        parallelBuilds[service] = {
                            stage("Build: ${service}") {
                                try {
                                    echo "🚀 Building: ${service}"
                                    sh "mvn clean package -pl ${service} -DfinalName=app -DskipTests"
                                    
                                    def jarfile = "${service}/target/app.jar"
                                    archiveArtifacts artifacts: jarfile, fingerprint: true
                                } catch (Exception e) {
                                    echo "❌ Build failed for ${service}: ${e.getMessage()}"
                                    error("Build failed for ${service}")
                                }
                            }
                        }
                    }

                    parallel parallelBuilds
                }
            }
        }


        stage('Build Docker Image') {
            when {
                expression { env.SERVICES.trim() && env.GIT_TAG }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    def parallelDockerBuilds = [:]

                    services.each { service ->
                        parallelDockerBuilds[service] = {
                            stage("Docker Build: ${service}") {
                                try {
                                    echo "🐳 Building Docker Image for: ${service}"
                                    sh "docker build --build-arg ARTIFACT_NAME=${service}/target/app -t ${DOCKER_IMAGE_BASENAME}/${service}:${env.GIT_TAG} -f docker/Dockerfile ."
                                } catch (Exception e) {
                                    echo "❌ Docker Build failed for ${service}: ${e.getMessage()}"
                                    error("Docker Build failed for ${service}")
                                }
                            }
                        }
                    }

                    parallel parallelDockerBuilds
                }
            }
        }

        stage('Login to Docker Registry') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login ${REGISTRY_URL} -u $DOCKER_USER --password-stdin"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { env.SERVICES.trim() && env.GIT_TAG }
            }
            steps {
                script {
                    def services = env.SERVICES.split(',')
                    def parallelDockerPush = [:]

                    services.each { service ->
                        parallelDockerPush[service] = {
                            stage("Docker Push: ${service}") {
                                try {
                                    echo "🐳 Push Docker Image for: ${service}"
                                    sh "docker push ${DOCKER_IMAGE_BASENAME}/${service}:${env.GIT_TAG}"
                                } catch (Exception e) {
                                    echo "❌ Docker Push failed for ${service}: ${e.getMessage()}"
                                    error("Docker Push failed for ${service}")
                                }
                            }
                        }
                    }

                    parallel parallelDockerPush 
                }
            }
        }
   }

    post {
        always {
            echo 'Cleaning up...'
            sh "docker logout ${REGISTRY_URL}"
        }
        success {
            publishChecks(
                name: 'PipelineResult',
                title: 'Code Coverage Check Success',
                status: 'COMPLETED',
                conclusion: 'SUCCESS',
                summary: 'Pipeline completed successfully.',
                detailsURL: env.BUILD_URL
            )
        }

        failure {
            publishChecks(
                name: 'PipelineResult',
                title: 'Code Coverage Check Fail',
                status: 'COMPLETED',
                conclusion: 'FAILURE', 
                summary: 'Pipeline failed. Check logs for details.',
                detailsURL: env.BUILD_URL
            )
        }
    }

}

def getServices() {
    def services = [
        'spring-petclinic-customers-service', 
        'spring-petclinic-vets-service',
        'spring-petclinic-visits-service',
        'spring-petclinic-admin-server', 
        'spring-petclinic-config-server',
        'spring-petclinic-discovery-server',
        'spring-petclinic-genai-service',
        'spring-petclinic-api-gateway',
    ]

    // Lấy danh sách file thay đổi giữa commit hiện tại và commit trước đó
    def changedFiles = sh(
        script: "git diff --name-only HEAD~1 HEAD",
        returnStdout: true
    ).trim().split('\n')

    def changedServices = services.findAll { service ->
        changedFiles.any { changedFile ->
            // Kiểm tra nếu file thay đổi nằm trong thư mục service
            changedFile.startsWith("${service}/")
        }
    }

    if (changedServices.isEmpty()) {
        return "NONE"
    }
    return changedServices.join(',')
}
