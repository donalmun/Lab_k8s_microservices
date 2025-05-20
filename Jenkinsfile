pipeline {
    agent any
    environment {
        DOCKER_HUB_USERNAME = "donalmun"
        DOCKER_HUB_CREDS = credentials('dockerhub')
        GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
    }
    stages {
        stage('Detect Changed Services') {
            steps {
                script {
                    def services = ['admin-server', 'api-gateway', 'config-server', 'customers-service', 'discovery-server', 'vets-service', 'visits-service']
                    def changedFiles = sh(script: "git diff-tree --no-commit-id --name-only -r HEAD", returnStdout: true).trim()
                    
                    env.CHANGED_SERVICES = ""
                    
                    // Tìm tất cả service có thay đổi
                    for (service in services) {
                        if (changedFiles.contains("/${service}/") || changedFiles.contains("${service}/")) {
                            echo "Changes detected in ${service}"
                            env.CHANGED_SERVICES = env.CHANGED_SERVICES + " " + service
                        }
                    }
                    
                    // Nếu không tìm thấy service nào, kiểm tra branch name
                    if (env.CHANGED_SERVICES.trim() == "") {
                        def branchName = env.BRANCH_NAME ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                        echo "No service changes detected directly, checking branch name: ${branchName}"
                        
                        for (service in services) {
                            if (branchName.contains(service)) {
                                env.CHANGED_SERVICES = env.CHANGED_SERVICES + " " + service
                                echo "Branch name indicates changes in ${service}"
                            }
                        }
                    }
                    
                    if (env.CHANGED_SERVICES.trim() == "") {
                        error "Could not determine which service(s) changed"
                    }
                    
                    env.CHANGED_SERVICES = env.CHANGED_SERVICES.trim()
                    echo "Will build images for: ${env.CHANGED_SERVICES}"
                }
            }
        }
        
        stage('Build and Push Images') {
            steps {
                script {
                    // Login Docker Hub một lần
                    sh "echo ${DOCKER_HUB_CREDS_PSW} | docker login -u ${DOCKER_HUB_USERNAME} --password-stdin"
                    
                    // Lấy danh sách các service đã thay đổi
                    def changedServicesList = env.CHANGED_SERVICES.split(" ")
                    
                    // Build và push image cho từng service
                    for (service in changedServicesList) {
                        echo "Processing ${service}..."
                        
                        // Build image
                        dir(service) {
                            sh "docker build -t ${DOCKER_HUB_USERNAME}/${service}:${GIT_COMMIT_SHORT} ."
                        }
                        
                        // Push image
                        sh "docker push ${DOCKER_HUB_USERNAME}/${service}:${GIT_COMMIT_SHORT}"
                        
                        // Trigger Helm update cho mỗi service
                        build job: 'k8s_update_helm', parameters: [
                            string(name: 'SERVICE_NAME', value: service),
                            string(name: 'IMAGE_TAG', value: "${GIT_COMMIT_SHORT}")
                        ], wait: false
                        
                        echo "Successfully processed ${service}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh "docker logout"
        }
        success {
            echo "Successfully built and pushed images for: ${env.CHANGED_SERVICES}"
        }
    }
}