pipeline {
    agent {
        node {
            label 'k8s_master'
        }
    }
    
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
                    
                    // Nếu vẫn không tìm thấy service, dùng tất cả với tag latest
                    if (env.CHANGED_SERVICES.trim() == "") {
                        echo "No specific service detected, using ALL services with tag 'latest'"
                        env.CHANGED_SERVICES = services.join(" ")
                        env.USE_LATEST_TAG = "true"
                    } else {
                        env.USE_LATEST_TAG = "false"
                    }
                    
                    echo "Will build images for: ${env.CHANGED_SERVICES}"
                }
            }
        }
        
        stage('Check Docker') {
            steps {
                script {
                    def dockerExists = sh(script: "which docker || true", returnStdout: true).trim()
                    if (dockerExists == "") {
                        error "Docker is not installed on this node. Please install Docker first."
                    } else {
                        def dockerVersion = sh(script: "docker --version", returnStdout: true).trim()
                        echo "Docker is installed: ${dockerVersion}"
                        
                        // Kiểm tra quyền chạy docker
                        def canRunDocker = sh(script: "docker ps >/dev/null 2>&1 && echo 'yes' || echo 'no'", returnStdout: true).trim()
                        if (canRunDocker == "no") {
                            error "Docker is installed but current user cannot access it. Add user to docker group."
                        }
                    }
                }
            }
        }
        
        stage('Build and Push Images') {
            steps {
                script {
                    // Login Docker Hub một lần (sử dụng cách an toàn hơn)
                    withCredentials([string(credentialsId: 'dockerhub', variable: 'DOCKER_PWD')]) {
                        sh 'echo $DOCKER_PWD | docker login -u ${DOCKER_HUB_USERNAME} --password-stdin'
                    }
                    
                    // Lấy danh sách các service đã thay đổi
                    def changedServicesList = env.CHANGED_SERVICES.split(" ")
                    
                    // Build và push image cho từng service
                    for (service in changedServicesList) {
                        echo "Processing ${service}..."
                        
                        // Quyết định tag
                        def imageTag = env.USE_LATEST_TAG == "true" ? "latest" : "${GIT_COMMIT_SHORT}"
                        
                        // Build image từ Dockerfile chung, chỉ định SERVICE_NAME
                        sh """
                            docker build \\
                                -t ${DOCKER_HUB_USERNAME}/${service}:${imageTag} \\
                                -f docker/Dockerfile \\
                                --build-arg SERVICE_NAME=${service} \\
                                .
                        """
                        
                        // Push image
                        sh "docker push ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
                        
                        // Trigger Helm update cho mỗi service - ĐÃ SỬA TÊN JOB
                        build job: 'k8s_update_helm', parameters: [
                            string(name: 'SERVICE_NAME', value: service),
                            string(name: 'IMAGE_TAG', value: imageTag)
                        ], wait: false
                        
                        echo "Successfully processed ${service}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh "docker logout || true"  // Thêm || true để tránh lỗi nếu docker chưa login
        }
        success {
            echo "Successfully built and pushed images for: ${env.CHANGED_SERVICES}"
        }
    }
}