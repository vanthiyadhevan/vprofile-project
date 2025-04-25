pipeline {
	agent any

	tools {
		maven 'maven'
	}

	environment {
		AWS_CRDS = credentials('aws_creds')
        AWS_REGION = credentials('aws_region')
        CLUSTER_NAME = credentials('clustername')

        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        CHART_PATH = 'helm/vprofile/'
        STAGING_NAMESPACE = 'staging'
        PROD_NAMESPACE = 'prod'

        ECR_REPO_NAME_VPROFILE = credentials('ecr_repo_name_vprofile')
        ECR_REPO_URI_VPROFILE = credentials('ecr_repo_uri_vprofile')

        GITHUB_TOKEN = credentials('github')
	}

	stages {

        stage('Installing Dependencies') {
            options { timestamps() }
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('Unit Test') {
            steps {
                script {
                    // Run Maven unit tests and generate reports
                    sh "mvn test"
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            } 
        }

        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO_URI_VPROFILE}:${BUILD_NUMBER} -f Dockerfile ."
                }
            }
        }

        stage('Trivy Vulnerability Scanner') {
            steps {
                // sh 'echo $PATH && which trivy && trivy --version'
                sh  ''' 
                    trivy image $ECR_REPO_URI_VPROFILE:$BUILD_NUMBER \
                        --severity LOW,MEDIUM,HIGH \
                        --exit-code 0 \
                        --quiet \
                        --format json -o trivy-image-MEDIUM-results.json

                    trivy image $ECR_REPO_URI_VPROFILE:$BUILD_NUMBER \
                        --severity CRITICAL \
                        --exit-code 0 \
                        --quiet \
                        --format json -o trivy-image-CRITICAL-results.json
                '''
            }
            post {
                always {
                    sh '''
                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-MEDIUM-results.html trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/html.tpl" \
                            --output trivy-image-CRITICAL-results.html trivy-image-CRITICAL-results.json

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-MEDIUM-results.xml  trivy-image-MEDIUM-results.json 

                        trivy convert \
                            --format template --template "@/usr/local/share/trivy/templates/junit.tpl" \
                            --output trivy-image-CRITICAL-results.xml trivy-image-CRITICAL-results.json          
                    '''
                }
            }
        }

        stage('ECR login and Push Docker Image') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws_creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URI_VPROFILE}"
                        sh "docker push ${ECR_REPO_URI_VPROFILE}:${BUILD_NUMBER}"
                    }
                }
            }
        }

        // stage('kube config creation') {
        //     steps{
        //         script {
        //             withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws_creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]){
        //                 sh 'aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}'
        //                 sh 'cat ~/.kube/config'
        //             }
        //         }
        //     }
        // }

        // stage('Deploy to Staging Helm') {
        //     steps {
        //         sh 'pwd'
        //         sh '''
        //             helm upgrade --install vprofile ${CHART_PATH} \
        //             --namespace staging \
        //             --create-namespace \
        //             -f ${CHART_PATH}/values-staging.yaml \
        //             --set appimage=${ECR_REPO_URI_VPROFILE}/${ECR_REPO_NAME_VPROFILE} \
        //             --set apptag=${BUILD_NUMBER} \
        //             --kubeconfig ${KUBECONFIG} --debug
        //            '''
        //     }
        // }

        stage('Upload - AWS S3') {
            when {
                branch 'staging'
            }
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws_creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh  '''
                            ls -ltr
                            mkdir reports-$BUILD_ID
                            cp -rf coverage/ reports-$BUILD_ID/
                            cp -rf target/surefire-reports/ reports-$BUILD_ID/
                            cp -rf target/checkstyle-result.xml reports-$BUILD_ID/
                            cp trivy*.* reports-$BUILD_ID/
                            ls -ltr reports-$BUILD_ID/
                        '''
                        s3Upload(
                            file:"reports-$BUILD_ID", 
                            bucket:'staging-test-reports', 
                            path:"jenkins-$BUILD_ID/"
                        )
                }
            }
        }
    }
    post {
        always {
            //Add channel name
            slackSend channel: '#jenkins-cicd', color: '#FF0000', message: "Find Status of Pipeline:- ${currentBuild.currentResult} ${env.JOB_NAME} ${env.BUILD_NUMBER} ${BUILD_URL}"
            // message: "Find Status of Pipeline:- ${currentBuild.currentResult} ${env.JOB_NAME} ${env.BUILD_NUMBER} ${BUILD_URL}"
        }
    }
}