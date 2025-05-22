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
        // Thêm biến phát hiện tag release
        GIT_TAG = sh(script: "git describe --tags --exact-match || true", returnStdout: true).trim()
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
                            // Loại bỏ dấu cách ở đầu trước khi thêm service mới
                            if (env.CHANGED_SERVICES.trim() == "") {
                                env.CHANGED_SERVICES = service
                            } else {
                                env.CHANGED_SERVICES = env.CHANGED_SERVICES + " " + service
                            }
                            echo "Changes detected in ${service}"
                        }
                    }
                    
                    // Nếu không tìm thấy service nào, kiểm tra branch name
                    if (env.CHANGED_SERVICES.trim() == "") {
                        def branchName = env.BRANCH_NAME ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                        echo "No service changes detected directly, checking branch name: ${branchName}"
                        
                        for (service in services) {
                            if (branchName.contains(service)) {
                                // Loại bỏ dấu cách ở đầu trước khi thêm service mới
                                if (env.CHANGED_SERVICES.trim() == "") {
                                    env.CHANGED_SERVICES = service
                                } else {
                                    env.CHANGED_SERVICES = env.CHANGED_SERVICES + " " + service
                                }
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
                    def servicePorts = [
                        'admin-server': '9090',
                        'api-gateway': '8080',
                        'config-server': '8888',
                        'customers-service': '8081',
                        'discovery-server': '8761',
                        'vets-service': '8083',
                        'visits-service': '8082'
                    ]
                    
                    // Đảm bảo không có phần tử rỗng trong danh sách service
                    def changedServicesList = env.CHANGED_SERVICES.trim().split("\\s+").findAll { it && it.trim() != "" }
                    echo "Services to build after filtering: ${changedServicesList.join(', ')}"
                    
                    // Nếu có thay đổi cụ thể, build từ source
                    if (env.BUILD_ALL != "true" && changedServicesList.size() > 0) {
                        echo "Building from source..."
                        // Build tất cả dự án từ thư mục gốc
                        sh "./mvnw clean package -DskipTests"
                        
                        for (service in changedServicesList) {
                            if (service.trim() == "") {
                                echo "Empty service name found, skipping"
                                continue
                            }
                            
                            echo "Processing service: '${service}'"
                            
                            // Đường dẫn chính xác đến thư mục service
                            def serviceDir = "spring-petclinic-${service}"
                            def jarPath = "${serviceDir}/target"
                            
                            // Debug
                            sh "echo 'Looking for JAR files in ${jarPath}...'"
                            sh "ls -la ${jarPath}/*.jar || echo 'No JAR files found in ${jarPath}'"
                            
                            // Tìm JAR file với tên chính xác
                            def jarFile = sh(script: "find ${jarPath} -name '*.jar' ! -name '*original*' | head -1 || echo ''", returnStdout: true).trim()
                            
                            if (jarFile && jarFile != '') {
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
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PWD')]) {
                        sh 'echo $DOCKER_PWD | docker login -u $DOCKER_USER --password-stdin'
                    }

                    def servicePorts = [
                        'admin-server': '9090',
                        'api-gateway': '8080',
                        'config-server': '8888',
                        'customers-service': '8081',
                        'discovery-server': '8761',
                        'vets-service': '8083',
                        'visits-service': '8082'
                    ]

                    def changedServicesList = env.CHANGED_SERVICES.trim().split("\\s+").findAll { it && it.trim() != "" }
                    def serviceList = []
                    def tagList = []

                    // Phát hiện build trên tag release
                    def isReleaseTag = (env.GIT_TAG && env.GIT_TAG.startsWith("v"))
                    def imageTag = isReleaseTag ? env.GIT_TAG : "${GIT_COMMIT_SHORT}"
                    def branchName = env.BRANCH_NAME ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    def needsLatestTag = (branchName == 'main' && !isReleaseTag)

                    for (service in changedServicesList) {
                        if (service.trim() == "") continue

                        echo "Processing ${service}..."

                        def servicePort = servicePorts[service]

                        // Build Docker image
                        sh """
                            docker build \\
                                -t ${DOCKER_HUB_USERNAME}/${service}:${imageTag} \\
                                -f docker/Dockerfile \\
                                --build-arg ARTIFACT_NAME=${service} \\
                                --build-arg EXPOSED_PORT=${servicePort} \\
                                .
                        """

                        // Tag latest nếu là main branch và không phải release tag
                        if (needsLatestTag) {
                            sh "docker tag ${DOCKER_HUB_USERNAME}/${service}:${imageTag} ${DOCKER_HUB_USERNAME}/${service}:latest"
                        }

                        // Push image với tag release hoặc commit
                        sh "docker push ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"

                        // Push latest nếu cần
                        if (needsLatestTag) {
                            sh "docker push ${DOCKER_HUB_USERNAME}/${service}:latest"
                        }

                        serviceList.add(service)
                        tagList.add(imageTag)
                    }

                    env.SERVICE_NAMES = serviceList.join(',')
                    env.IMAGE_TAGS = tagList.join(',')
                    env.IS_RELEASE_TAG = isReleaseTag ? "true" : "false"
                }
            }
        }
        
        stage('Update Helm Charts') {
            steps {
                script {
                    if (env.SERVICE_NAMES == null || env.SERVICE_NAMES.trim() == '') {
                        echo "No services to update. Skipping Helm chart update."
                    } else {
                        def serviceNames = env.SERVICE_NAMES.split(',')
                        def imageTags = env.IMAGE_TAGS.split(',')
                        if (serviceNames.size() != imageTags.size()) {
                            error "Mismatch between service names and image tags!"
                        }
                        // Chọn file values phù hợp
                        def isReleaseTag = (env.IS_RELEASE_TAG == "true")
                        def valuesFile = isReleaseTag ? "values-staging.yaml" : "values-dev.yaml"
                        echo "Updating Helm charts for these services: ${env.SERVICE_NAMES} with tags: ${env.IMAGE_TAGS} (file: ${valuesFile})"
                        build job: 'k8s_update_helm',
                            wait: false,
                            parameters: [
                                string(name: 'SERVICE_NAMES', value: env.SERVICE_NAMES),
                                string(name: 'IMAGE_TAGS', value: env.IMAGE_TAGS),
                                string(name: 'VALUES_FILE', value: valuesFile)
                            ]
                        echo "Triggered Helm update job for ${valuesFile} - continuing without waiting"
                    }
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                script {
                    if (env.BUILD_ALL != "true") {
                        def changedServicesList = env.CHANGED_SERVICES.trim().split("\\s+").findAll { it && it.trim() != "" }
                        for (service in changedServicesList) {
                            if (service.trim() != "") {
                                sh "rm -f ${service}.jar || true"
                            }
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