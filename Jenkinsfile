pipeline {
	agent any

	tools {
		maven 'mvn'
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

        stage('Test') {
            steps {
                script {
                    // Run Maven unit tests and generate reports
                    sh "mvn test"
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
                    sh "docker build -t ${ECR_REPO_URI}:${BUILD_NUMBER} -f Dockerfile ."
                }
            }
        }

        stage('ECR login and Push Docker Image') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws_creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URI}"
                        sh "docker push ${ECR_REPO_URI}:${BUILD_NUMBER}"
                    }
                    // sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}"
                }
            }
        }

        stage('kube config creation') {
            steps{
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws_creds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]){
                        sh 'aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}'
                        sh 'cat ~/.kube/config'
                    }
                }
            }
        }

        stage('Deploy to Staging Helm') {
            steps {
                sh 'pwd'
                sh '''
                    helm upgrade --install vprofile ${CHART_PATH} \
                    --namespace staging \
                    --create-namespace \
                    -f ${CHART_PATH}/values-staging.yaml 
                   '''
            }
        }
    }
}