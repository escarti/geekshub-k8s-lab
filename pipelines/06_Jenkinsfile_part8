pipeline {
    agent any
    triggers {
        /* Vamos a pullear el repo activamente porque en nuestra instalación local sería complicado usar Webhooks */
        pollSCM('* * * * */1')
    }

    options {
        disableConcurrentBuilds()
    }
    environment {
        registry = "escarti/geekshub-django" /* Usad vuestro docker-hub registry */
        registryCredential = 'Docker'
        imageTag = "${env.GIT_BRANCH + '_' + env.BUILD_NUMBER}"
        apiServer = "https://192.168.99.101:8443"
        devNamespace = "default"
        minikubeCredential = 'minikube-auth-token'
        deploymentRepo = "geekshub-django-deployment"
    }
    stages { 
        stage('Build image') {
            steps {
                script {
                    dockerImage = docker.build(registry + ":$imageTag", "--cache-from $registry:latest --network host .")
                }
            }
        }
        stage('Test') {
            steps {
                sh "IMAGE=$registry TAG=$imageTag docker-compose -f docker-compose_test.yaml up --abort-on-container-exit --exit-code-from webapp"
            }
        }     
        stage('Upload image to registry') {
            steps{
                script {
                    docker.withRegistry( 'https://registry.hub.docker.com', registryCredential ) {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
        stage('Update deployment file') {
            when {
                expression { env.GIT_BRANCH == 'develop' }
            }
            steps{
                script {
                    withCredentials([usernamePassword(credentialsId: 'GitHub-encoded', usernameVariable: 'username', passwordVariable: 'password')]){
                        sh "rm -rf $deploymentRepo"
                        sh "git clone https://$username:$password@github.com/$username/${deploymentRepo}.git"
                        dir("$deploymentRepo") {
                            sh "echo \"spec:\n  template:\n    spec:\n      containers:\n        - name: django\n          image: ${registry}:$imageTag\" > patch.yaml"
                            sh "kubectl patch --local -o yaml -f django-deployment.yaml -p \"\$(cat patch.yaml)\" > new-deploy.yaml"
                            sh "mv new-deploy.yaml django-deployment.yaml"
                            sh "rm patch.yaml"
                            sh "git add ."
                            sh "git commit -m\"Patched deployment for $imageTag\""
                            sh "git push https://$username:$password@github.com/$username/${deploymentRepo}.git"
                        }
                    }
                }
            }
        } 
        stage('Deploy to K8s') {
            when {
                expression { env.GIT_BRANCH == 'develop' }
            }
            steps{
                withKubeConfig([credentialsId: minikubeCredential,
                                serverUrl: apiServer,
                                namespace: devNamespace
                               ]) {
                    sh "kubectl apply -f ${deploymentRepo}/django-deployment.yaml"
                }
            }
        }
    }
}