pipeline {
    agent { label 'built-in' }

    environment {
        REGISTRY_URL = "docker.io"
        DOCKER_IMAGE_BASENAME = "anthonynhat"  // đổi thành "thuanlp" nếu cần
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
                            } else {
                                env.GIT_TAG = "latest"
                            }
                        } else {
                            def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                            env.GIT_TAG = commitId
                        }
                        echo "GIT_TAG: ${env.GIT_TAG}"
                    } catch (Exception e) {
                        echo "Failed to determine tag: ${e.getMessage()}"
                        env.GIT_TAG = "latest"
                    }
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    env.CHANGED_SERVICES = getChangedServices()
                    if (env.BRANCH_NAME == 'main') {
                        env.SERVICES_TO_BUILD = getAllServices().join(',')
                        echo "Main branch: Building all services"
                    } else {
                        if (env.CHANGED_SERVICES == "NONE") {
                            echo "No relevant changes detected. Skipping build."
                            error("No relevant changes detected")
                        } else {
                            echo "Detected changed services: ${env.CHANGED_SERVICES}"
                            env.SERVICES_TO_BUILD = env.CHANGED_SERVICES
                        }
                    }
                }
            }
        }

        stage('Run Unit Test') {
            when {
                expression { env.BRANCH_NAME != 'main' && env.SERVICES_TO_BUILD?.trim() }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.split(',')
                    def coverageResults = []
                    def buildableServices = []
                    def parallelTests = [:]

                    services.each { service ->
                        parallelTests[service] = {
                            stage("Test: ${service}") {
                                try {
                                    sh "mvn test -pl ${service} -DskipTests=false"
                                    sh "mvn jacoco:report -pl ${service}"

                                    def reportPath = "${service}/target/site/jacoco/index.html"
                                    def resultPath = "${service}/target/surefire-reports/*.txt"
                                    def coverage = 0

                                    if (fileExists(reportPath)) {
                                        archiveArtifacts artifacts: resultPath, fingerprint: true
                                        archiveArtifacts artifacts: reportPath, fingerprint: true

                                        coverage = sh(
                                            script: """
                                            grep -oP '(?<=<td class="ctr2">)\\d+%' ${reportPath} | head -1 | sed 's/%//'
                                            """,
                                            returnStdout: true
                                        ).trim()

                                        if (!coverage) {
                                            echo "⚠️ Coverage extraction failed"
                                            coverage = 0
                                        } else {
                                            coverage = coverage.toInteger()
                                        }
                                    } else {
                                        echo "⚠️ No coverage report found"
                                    }

                                    echo "📊 ${service}: ${coverage}%"
                                    coverageResults << "${service}:${coverage}%"

                                    if (coverage < 70) {
                                        error "❌ ${service} test coverage too low: ${coverage}%"
                                    } else {
                                        buildableServices << service
                                    }
                                } catch (Exception e) {
                                    echo "❌ Error testing ${service}: ${e.getMessage()}"
                                }
                            }
                        }
                    }

                    parallel parallelTests

                    env.CODE_COVERAGES = coverageResults.join(', ')
                    env.SERVICES_TO_BUILD = buildableServices.join(',')
                }
            }
        }

        stage('Build Services') {
            when {
                expression { env.SERVICES_TO_BUILD?.trim() }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.split(',')
                    def parallelBuilds = [:]

                    services.each { service ->
                        parallelBuilds[service] = {
                            stage("Build: ${service}") {
                                try {
                                    echo "🚀 Building: ${service}"
                                    sh "mvn clean package -pl ${service} -DfinalName=app -DskipTests"
                                    archiveArtifacts artifacts: "${service}/target/app.jar", fingerprint: true
                                } catch (Exception e) {
                                    echo "❌ Build failed: ${e.getMessage()}"
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
                expression { env.SERVICES_TO_BUILD?.trim() && env.GIT_TAG }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.split(',')
                    def parallelDockerBuilds = [:]

                    services.each { service ->
                        parallelDockerBuilds[service] = {
                            stage("Docker Build: ${service}") {
                                try {
                                    echo "🐳 Building Docker for: ${service}"
                                    sh "docker build --build-arg ARTIFACT_NAME=${service}/target/app -t ${DOCKER_IMAGE_BASENAME}/${service}:${env.GIT_TAG} -f docker/Dockerfile ."
                                } catch (Exception e) {
                                    echo "❌ Docker Build failed: ${e.getMessage()}"
                                    error("Docker build failed for ${service}")
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
                        sh '''
                            echo "$DOCKER_PASS" | docker login ${REGISTRY_URL} -u "$DOCKER_USER" --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { env.SERVICES_TO_BUILD && env.SERVICES_TO_BUILD.trim() && env.GIT_TAG }
            }
            steps {
                script {
                    def services = env.SERVICES_TO_BUILD.split(',')
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
                title: '✅ Pipeline Success',
                status: 'COMPLETED',
                conclusion: 'SUCCESS',
                summary: 'Pipeline completed successfully.',
                detailsURL: env.BUILD_URL
            )
        }

        failure {
            publishChecks(
                name: 'PipelineResult',
                title: '❌ Pipeline Failed',
                status: 'COMPLETED',
                conclusion: 'FAILURE',
                summary: 'Pipeline failed. Check logs.',
                detailsURL: env.BUILD_URL
            )
        }
    }
}

// Detect changed services in feature branches
def getChangedServices() {
    def changedFiles = sh(script: "git diff --name-only origin/${env.BRANCH_NAME}~1 origin/${env.BRANCH_NAME}", returnStdout: true).trim().split("\n")
    def services = getAllServices()

    def affected = services.findAll { service ->
        changedFiles.any { file -> file.startsWith(service + "/") }
    }

    if (affected.isEmpty()) return "NONE"
    return affected.join(',')
}

// Full list of services
def getAllServices() {
    return [
        'spring-petclinic-customers-service', 
        'spring-petclinic-vets-service',
        'spring-petclinic-visits-service',
        'spring-petclinic-admin-server', 
        'spring-petclinic-config-server',
        'spring-petclinic-discovery-server',
        'spring-petclinic-genai-service',
        'spring-petclinic-api-gateway'
    ]
}
