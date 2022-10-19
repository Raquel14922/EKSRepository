pipeline {
    agent any
    tools {
        maven 'M3_8_6'
        terraform 'Terraform'
    }
    parameters {
        booleanParam(name: 'destroy', defaultValue: false, description: 'Destroy Terraform build?')
        booleanParam(name: 'deploy', defaultValue: false, description: 'Deploy Terraform apply?')
        booleanParam(name: 'database', defaultValue: false, description: 'Create tables?')
    }
    environment {
        AWS_ACCESS_KEY_ID       = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY   = credentials('AWS_SECRET_ACCESS_KEY')
        NAME                    = 'dev'
        TF_IN_AUTOMATION        = '1'
        AWS_ACCOUNT_ID          = "262583979852"
        AWS_DEFAULT_REGION      = "us-east-1" 
        IMAGE_REPO_NAME         = "ECR_REPO_NAME"
        IMAGE_TAG               = "IMAGE_TAG"
        REPOSITORY_URI          = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        registry_payment        = '262583979852.dkr.ecr.us-east-1.amazonaws.com/payment-service-${NAME}:v4'
        registry_order          = '262583979852.dkr.ecr.us-east-1.amazonaws.com/order-service:v4'
        registry_kitchen        = '262583979852.dkr.ecr.us-east-1.amazonaws.com/kitchen-service:v4'
    }
    stages {
        stage('Create Infra') {
            when {
                equals expected: true, actual: params.deploy
            }
            steps {
                dir("infra/"){
                    sh "terraform init"
                    sh "terraform apply --auto-approve"
                }
            }
        }
        stage('Destroy Infra') {
            when {
                equals expected: true, actual: params.destroy
            }
            steps {
                dir("infra/"){
                    sh "terraform destroy --auto-approve"
                }
            }
        }
        stage('Database') {
            when {
                equals expected: true, actual: params.database
            }
            steps {
                dir("liquibase/"){
                    sh '/opt/liquibase/liquibase --changeLogFile="changesets/db.changelog-master.xml" update'
                }
            }
        }
        stage("Docker Build") {
            steps {
                dir("payment-service/"){
                    sh "docker build --cache-from payment-service:latest -t payment-service:latest ."
                    //sh "docker pull payment-service:latest"
                    //docker hub publico
                }
                dir("order-service/"){
                    sh "docker build --cache-from order-service:latest -t order-service:latest ."
                }
                dir("kitchen-service/"){
                    sh "docker build --cache-from kitchen-service:latest -t kitchen-service:latest ."
                }
            }
        }
        stage('Logging into AWS ECR') {
            steps {
                withAWS(credentials: 'ecr-credentials', region: 'us-east-1') {
                        script {
                            def login = ecrLogin()
                            sh "${login}"
                            sh '''docker tag payment-service:latest 262583979852.dkr.ecr.us-east-1.amazonaws.com/payment-service:v4'''
                            sh '''docker tag order-service:latest 262583979852.dkr.ecr.us-east-1.amazonaws.com/order-service:v4'''
                            sh '''docker tag kitchen-service:latest 262583979852.dkr.ecr.us-east-1.amazonaws.com/kitchen-service:v4'''
                        }
                }
            }
        }
        stage("Docker Push") {
            steps {
                sh "docker push ${registry_payment}"
                sh "docker push ${registry_order}"
                sh "docker push ${registry_kitchen}"
            }
        }
        stage('Kubectl') {
            steps {
                withAWS(credentials: 'ecr-credentials', region: 'us-east-1') {
                    sh 'aws eks --region us-east-1 update-kubeconfig --name eks-cluster-test'
                    sh 'kubectl get pods'
                    dir("Deployment/"){
                        sh 'kubectl apply -f Payment-deployment.yaml'
                        sh 'kubectl apply -f Kitchen-deployment.yaml'
                        sh 'kubectl apply -f Order-deployment.yaml'
                        //sh 'kubectl apply -f secrets.yaml'
                        sh 'kubectl apply -f ingress-deploy.yaml'
                        sh 'kubectl apply -f nginx_ingress_services.yaml'
                    }
                }
            }
        }
    }
}
