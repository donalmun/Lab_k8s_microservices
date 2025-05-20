pipeline {
    agent {
        node {
            label 'lab1_agent'
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
                    
                    // Nếu vẫn không tìm thấy service nào, sử dụng tất cả services
                    if (env.CHANGED_SERVICES.trim() == "") {
                        echo "No specific service detected, using ALL services with commit tag ${GIT_COMMIT_SHORT}"
                        env.CHANGED_SERVICES = services.join(" ")
                    }
                    
                    echo "Will build images for: ${env.CHANGED_SERVICES}"
                }
            }
        }
        
        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Login to Docker Hub
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PWD')]) {
                        sh 'echo $DOCKER_PWD | docker login -u $DOCKER_USER --password-stdin'
                    }
                    
                    // Process each service
                    def changedServicesList = env.CHANGED_SERVICES.split(" ")
                    for (service in changedServicesList) {
                        echo "Processing ${service}..."
                        
                        // Luôn sử dụng commit ID làm tag (theo yêu cầu)
                        def imageTag = "${GIT_COMMIT_SHORT}"
                        
                        // Branch main cũng cần tag latest
                        def branchName = env.BRANCH_NAME ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                        def needsLatestTag = (branchName == 'main')
                        
                        // Pull image gốc (đơn giản nhất)
                        sh "docker pull springcommunity/spring-petclinic-${service}"
                        
                        // Tag với tên đơn giản và commit ID
                        sh "docker tag springcommunity/spring-petclinic-${service} ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
                        
                        // Nếu là branch main, tag thêm latest
                        if (needsLatestTag) {
                            sh "docker tag springcommunity/spring-petclinic-${service} ${DOCKER_HUB_USERNAME}/${service}:latest"
                        }
                        
                        // Push image với tag commit ID
                        sh "docker push ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
                        
                        // Push image với tag latest nếu cần
                        if (needsLatestTag) {
                            sh "docker push ${DOCKER_HUB_USERNAME}/${service}:latest"
                        }
                        
                        // Trigger Helm update
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
            sh "docker logout || true"
        }
        success {
            echo "Successfully built and pushed images for: ${env.CHANGED_SERVICES}"
        }
    }
}