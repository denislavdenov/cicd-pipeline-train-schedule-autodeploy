pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "denislavd/train-schedule"
        CANARY_REPLICAS = 0
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('CanaryDeploy') {
            when {
                branch 'master'
            }
            environment { 
                CANARY_REPLICAS = 1
            }
            steps {
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube-canary.yml',
                    enableConfigSubstitution: true
                )
            }
        }
        stage('SmokeTest') {
            when {
                branch 'master'
            }
            steps {
                script {
                    sleep 5
                    
                    sh '''
                    response=$(curl -I http://$KUBE_MASTER_IP:30001 | grep '200 OK' | cut -c 10-15)
                    echo "response = ${response}"

                    if [ "$response" == '200 OK' ]; then 
                        echo "Smoke test against canary deployment passed - ${response}."
                    else
                        echo "Smoke test against canary deployment failed - ${response}."
                        exit 1
                    fi
                    '''
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
    post {
        cleanup {
            kubernetesDeploy (
                kubeconfigId: 'kubeconfig',
                configs: 'train-schedule-kube-canary.yml',
                enableConfigSubstitution: true
            )
        }
    }
}
