pipeline {
    agent any

    tools {
        jdk "JDK"
        maven "M3"
    }
    environment {
        AWS_CREDENTIAL_NAME = "AWSkey"
        REGION = "ap-northeast-2"
        DOCKER_IMAGE_NAME= "project01-ecr"
        ECR_REPOSITORY = "257307634175.dkr.ecr.ap-northeast-2.amazonaws.com"
        ECR_DOCKER_IMAGE = "${ECR_REPOSITORY}/${DOCKER_IMAGE_NAME}"
        EKS_JENKINS_CREDENTIAL_ID = 'kubectl-deploy-credentials'
        EKS_CLUSTER_NAME = 'project01-cluster'
        EKS_API = 'https://F75E608F5CBDDE8F828B4CCE5F1F5FC2.gr7.ap-northeast-2.eks.amazonaws.com'
    }
    // 위에 크리덴셜 젠킨스에서 설정한거랑 이름 똑같은지 잘 봐야 함. 나 자꾸 마지막 s 빼먹음
    stages {
        stage('Git Clone') {
            steps {
                echo 'Git Clone'
                git url: 'https://github.com/wnsdh0202/spring-petclinic.git',
                branch: 'wavefront'
            }
            post {
                success {
                    echo 'success clone project'
                }
                failure {
                    error 'fail clone project' // exit pipeline
                }
            }
        }        
        stage ('mvn Build') {
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true install' 
            }
            post {
                success {
                    junit '**/target/surefire-reports/TEST-*.xml' 
                }
            }
        }        
        stage ('Docker Build') {
            steps {
                dir("${env.WORKSPACE}") {
                    sh """
                      docker build -t $ECR_DOCKER_IMAGE:$BUILD_NUMBER .
                      docker tag $ECR_DOCKER_IMAGE:$BUILD_NUMBER $ECR_DOCKER_IMAGE:latest
                    """
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                echo "Push Docker Image to ECR"
                script{
                    // cleanup current user docker credentials
                    sh 'rm -f ~/.dockercfg ~/.docker/config.json || true' 
                    docker.withRegistry("https://${ECR_REPOSITORY}", "ecr:${REGION}:${AWS_CREDENTIAL_NAME}") {
                        docker.image("${ECR_DOCKER_IMAGE}:${BUILD_NUMBER}").push()
                        docker.image("${ECR_DOCKER_IMAGE}:latest").push()
                    }
                    
                }
            }
            post {
                success {
                    echo "Push Docker Image success!"
                }
            }
        }
        stage('Clean Up Docker Images on Jenkins Server') {
            steps {
                echo 'Cleaning up unused Docker images on Jenkins server'

                // Clean up unused Docker images, including those created within the last hour
                sh "docker image prune -f --all --filter \"until=1h\""
            }
        }
        stage('Deploy to eks') {
            steps {
                withKubeConfig([credentialsId: 'kubectl-deploy-credentials',
                    serverUrl: "${EKS_API}",
                    clusterName: "${EKS_CLUSTER_NAME}"]){
                        sh "sed 's/IMAGE_VERSION/v${env.BUILD_ID}/g' service.yaml > output.yaml"
                        // sh "aws eks --region ${REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}"
                        sh "kubectl apply -f output.yaml"
                        sh "rm -f output.yaml"
                            }
            }            
        }
            
        // stage('Upload to S3') {
        //     steps {
        //         echo "Upload to S3"
        //         dir("${env.WORKSPACE}") {
        //             sh 'zip -r deploy.zip ./deploy appspec.yml'
        //             withAWS(region:"${REGION}", credentials:"${AWS_CREDENTIAL_NAME}"){
        //               s3Upload(file:"deploy.zip", bucket:"aws05-codedeploy-bucket")
        //             } 
        //             sh 'rm -rf ./deploy.zip'                 
        //         }        
        //     }
        // }
        // stage('Codedeploy Workload') {
        //     steps {
        //        echo "create Codedeploy group"   
        //         sh '''
        //             aws deploy create-deployment-group \
        //             --application-name aws05-code-deploy \
        //             --auto-scaling-groups aws05-asg \
        //             --deployment-group-name aws05-code-deploy-${BUILD_NUMBER} \
        //             --deployment-config-name CodeDeployDefault.OneAtATime \
        //             --service-role-arn arn:aws:iam::257307634175:role/aws05-codedeploy-service-role
        //             '''
        //         echo "Codedeploy Workload"   
        //         sh '''
        //             aws deploy create-deployment --application-name aws05-code-deploy \
        //             --deployment-config-name CodeDeployDefault.OneAtATime \
        //             --deployment-group-name aws05-code-deploy-${BUILD_NUMBER} \
        //             --s3-location bucket=aws05-codedeploy-bucket,bundleType=zip,key=deploy.zip
        //             '''
        //             sleep(10) // sleep 10s
        //     }
        // }            
    }
}
