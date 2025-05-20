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
                            
                            // Find JAR file
                            def jarFile = sh(script: "find ${service}/target -name '*.jar' | head -1", returnStdout: true).trim()
                            
                            // Copy to root with simple name for Docker build
                            sh "cp ${jarFile} ${service}.jar"
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
                            // Nếu không có thay đổi cụ thể, pull và tag
                            sh "docker pull springcommunity/spring-petclinic-${service}"
                            sh "docker tag springcommunity/spring-petclinic-${service} ${DOCKER_HUB_USERNAME}/${service}:${imageTag}"
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
    }
}