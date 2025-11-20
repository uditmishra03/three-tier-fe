// Frontend Pipeline - Testing webhook trigger
pipeline {
    agent any 
    tools {
        jdk 'jdk'
        nodejs 'nodejs'
    }
    environment  {
        SCANNER_HOME=tool 'sonar-scanner'
        AWS_ACCOUNT_ID = credentials('ACCOUNT_ID')
        AWS_ECR_REPO_NAME = credentials('ECR_REPO01')
        AWS_DEFAULT_REGION = 'us-east-1'
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/"
        // Date-based semantic versioning: YYYYMMDD-BUILD (e.g., 20241120-001)
        // Zero-padded to 3 digits to ensure proper lexicographic sorting
        IMAGE_TAG = "${new Date().format('yyyyMMdd')}-${String.format('%03d', BUILD_NUMBER.toInteger())}"
    }
    stages {
        stage('Sonarqube Analysis & Quality Check') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=three-tier-fe \
                    -Dsonar.projectKey=three-tier-fe '''
                }
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    }
                }
            }
        }
        // stage('OWASP Dependency-Check Scan') {
        //     steps {
        //         dir('Application-Code/frontend') {
        //             dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        //             dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //         }
        //     }
        // }
        stage('Trivy File Scan') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh 'trivy fs . > trivyfs.txt'
                    }
                }
            }
        }
        stage("Docker Image Build & Push with Buildx") {
            steps {
                script {
                    sh 'aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}'
                    sh '''
                        # Create builder if it doesn't exist
                        if ! docker buildx inspect mybuilder > /dev/null 2>&1; then
                            docker buildx create --name mybuilder --use --bootstrap
                        else
                            docker buildx use mybuilder
                        fi
                    '''
                    sh 'docker buildx build --platform linux/amd64 -t ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${IMAGE_TAG} --push .'
                }
            }
        }
        stage("TRIVY Image Scan") {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh 'trivy image ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${IMAGE_TAG} > trivyimage.txt'
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                try {
                    cleanWs()
                } catch (Exception e) {
                    echo "Workspace cleanup skipped: ${e.message}"
                }
            }
        }
        success {
            script {
                try {
                    echo "Pipeline completed successfully!"
                    echo "Frontend image pushed to ECR: ${REPOSITORY_URI}${AWS_ECR_REPO_NAME}:${IMAGE_TAG}"
                    echo "ArgoCD Image Updater will automatically detect and deploy the new image"
                } catch (Exception e) {
                    echo "Pipeline completed successfully!"
                }
            }
        }
    }
}
