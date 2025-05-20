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
                        env.BUILD_ALL = "true"
                    } else {
                        env.BUILD_ALL = "false"
                    }
                    
                    echo "Will build images for: ${env.CHANGED_SERVICES}"
                }
            }
        }
        
        stage('Build Services') {
            steps {
                script {
                    // Maps services to their ports
                    def servicePorts = [
                        'admin-server': '9090',
                        'api-gateway': '8080',
                        'config-server': '8888',
                        'customers-service': '8081',
                        'discovery-server': '8761',
                        'vets-service': '8083',
                        'visits-service': '8082'
                    ]
                    
                    def changedServicesList = env.CHANGED_SERVICES.split(" ")
                    
                    // Nếu có thay đổi cụ thể, build từ source
                    if (env.BUILD_ALL != "true") {
                        for (service in changedServicesList) {
                            echo "Building ${service} from source..."
                            
                            // Compile service with Maven
                            dir(service) {
                                sh "./mvnw clean package -DskipTests"
                            }
                            
                            // Find JAR file - sử dụng đường dẫn tương đối
                            def jarFile = sh(script: "find ${service}/target -name '*.jar' | grep -v original | head -1", returnStdout: true).trim()
                            
                            if (jarFile) {
                                echo "Found JAR file: ${jarFile}"
                                // Copy to root with simple name for Docker build
                                sh "cp ${jarFile} ${service}.jar"
                            } else {
                                error "Could not find JAR file for ${service}. Build may have failed."
                            }
                        }
                    }
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
                    
                    // Service ports mapping
                    def servicePorts = [
                        'admin-server': '9090',
                        'api-gateway': '8080',
                        'config-server': '8888',
                        'customers-service': '8081',
                        'discovery-server': '8761',
                        'vets-service': '8083',
                        'visits-service': '8082'
                    ]
                    
                    // Process each service
                    def changedServicesList = env.CHANGED_SERVICES.split(" ")
                    
                    // Tạo các list để lưu các service và tag
                    def serviceList = []
                    def tagList = []
                    
                    for (service in changedServicesList) {
                        echo "Processing ${service}..."
                        
                        // Tag là commit ID
                        def imageTag = "${GIT_COMMIT_SHORT}"
                        
                        // Branch main cũng cần tag latest
                        def branchName = env.BRANCH_NAME ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                        def needsLatestTag = (branchName == 'main')
                        
                        // Nếu có thay đổi cụ thể, build từ source với Dockerfile
                        if (env.BUILD_ALL != "true") {
                            def servicePort = servicePorts[service]
                            
                            // Build Docker image từ JAR file
                            sh """
                                docker build \\
                                    -t ${DOCKER_HUB_USERNAME}/${service}:${imageTag} \\
                                    -f docker/Dockerfile \\
                                    --build-arg ARTIFACT_NAME=${service} \\
                                    --build-arg EXPOSED_PORT=${servicePort} \\
                                    .
                            """
                        } else {
                            // Thử pull từ Docker Hub cá nhân trước
                            def pullResult = sh(script: "docker pull ${DOCKER_HUB_USERNAME}/${service}:latest || echo 'not_found'", returnStdout: true).trim()
                            
                            if (pullResult.contains('not_found')) {
                                // Nếu không tìm thấy trong repo cá nhân, pull từ springcommunity
                                echo "Image not found in personal repo, pulling from springcommunity"
                                sh "docker pull springcommunity/spring-petclinic-${service}"
                                sh "docker tag springcommunity/spring-petclinic-${service} ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
                            } else {
                                // Nếu tìm thấy trong repo cá nhân, tag lại với commit ID mới
                                echo "Found existing image in personal repo, re-tagging"
                                sh "docker tag ${DOCKER_HUB_USERNAME}/${service}:latest ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
                            }
                        }
                        
                        // Nếu là branch main, tag thêm latest
                        if (needsLatestTag) {
                            sh "docker tag ${DOCKER_HUB_USERNAME}/${service}:${imageTag} ${DOCKER_HUB_USERNAME}/${service}:latest"
                        }
                        
                        // Push image với tag commit ID
                        sh "docker push ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
                        
                        // Push image với tag latest nếu cần
                        if (needsLatestTag) {
                            sh "docker push ${DOCKER_HUB_USERNAME}/${service}:latest"
                        }
                        
                        // Lưu service và tag để update Helm sau
                        serviceList.add(service)
                        tagList.add(imageTag)
                        
                        echo "Successfully processed ${service}"
                    }
                    
                    // Lưu danh sách service và tag dạng chuỗi phân cách bằng dấu phẩy
                    env.SERVICE_NAMES = serviceList.join(',')
                    env.IMAGE_TAGS = tagList.join(',')
                }
            }
        }
        
        stage('Update Helm Charts') {
            steps {
                script {
                    echo "Checking if there are services to update..."
                    
                    if (env.SERVICE_NAMES == null || env.SERVICE_NAMES.trim() == '') {
                        echo "No services to update. Skipping Helm chart update."
                    } else {
                        def serviceNames = env.SERVICE_NAMES.split(',')
                        def imageTags = env.IMAGE_TAGS.split(',')
                        
                        if (serviceNames.size() != imageTags.size()) {
                            error "Mismatch between service names and image tags!"
                        }
                        
                        echo "Validating services: ${env.SERVICE_NAMES}"
                        echo "With tags: ${env.IMAGE_TAGS}"
                        
                        // Kiểm tra xem có service nào để cập nhật không
                        if (serviceNames.size() > 0) {
                            echo "Updating Helm charts for these services: ${env.SERVICE_NAMES}"
                            
                            // Kích hoạt job nhưng không chờ nó hoàn thành
                            build job: 'k8s_update_helm', 
                                wait: false,
                                parameters: [
                                    string(name: 'SERVICE_NAMES', value: env.SERVICE_NAMES),
                                    string(name: 'IMAGE_TAGS', value: env.IMAGE_TAGS)
                                ]
                            
                            echo "Triggered Helm update job - continuing without waiting"
                        } else {
                            echo "No services to update. Skipping Helm chart update."
                        }
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    if (env.BUILD_ALL != "true") {
                        def changedServicesList = env.CHANGED_SERVICES.split(" ")
                        for (service in changedServicesList) {
                            sh "rm -f ${service}.jar || true"
                        }
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
        failure {
            echo "Build failed. Check logs for details."
        }
    }
}