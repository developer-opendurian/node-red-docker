@Library('opendurian') _
def APP_VERSION = null
def COMMIT_HASH = null
def IMAGE_TAG = null
def GIT_TAG = null
def commitSha1() {
  sh 'git rev-parse HEAD > commitSha1'
  def commit = readFile('commitSha1').trim()
  sh 'rm commitSha1'
  commit.substring(0, 7)
}
pipeline {
    options {
        disableConcurrentBuilds()
    }
    environment {
        DOCKER_REGISTRY = '609207776791.dkr.ecr.ap-southeast-1.amazonaws.com'
        DOCKER_REPO_NAME = 'node-red'
        PROJECT_NAME = 'node-red'
        IMAGE_NAME = "${env.DOCKER_REGISTRY}/${env.DOCKER_REPO_NAME}"
        TAG_PREFIX = "${env.BRANCH_NAME}"
        COMMIT_HASH = commitSha1()
        IMAGE_TAG = "${env.PROJECT_NAME}-${env.TAG_PREFIX}-${env.COMMIT_HASH}"
        IMAGE_TAG_LATEST = "${env.PROJECT_NAME}-${env.TAG_PREFIX}-latest"

    }
    post {
        always {
            cleanWs()
        }
        success {
            node('master') {
                opdSlackNotify2('good', 'Success',COMMIT_HASH)
            }
        }
        failure {
            node('master') { opdSlackNotify2('danger', 'Failed',COMMIT_HASH) }
        }
        aborted {
            node('master') { opdSlackNotify2('warning', 'Aborted',COMMIT_HASH) }
        }
    }
  
    agent {
        dockerfile {
            additionalBuildArgs '--pull --build-arg jenkins_uid=$(id -u jenkins) --build-arg docker_uid=$(getent group docker | awk -F: \'{printf "%d",$3}\')'
            args "-u jenkins --privileged -v /var/run/docker.sock:/var/run/docker.sock:rw,z"
            dir './build/pipeline'
            filename 'Dockerfile'
        }
    }

  
    //stages {
       // stage('Pre') {
            //steps {
                //echo "Pulling scripts repository..."
               // opdPullScript('AuthenGit','github.com/OpenDurian/opendurian-pipeline.git','~/script')
            //}
        //}
    stages {
        stage('Pre') {
            steps {
                echo "Pulling scripts repository..."
                dir("~/script") {
                checkout([$class: 'GitSCM',
                branches: [[name: 'master']],
                doGenerateSubmoduleConfigurations: false,
                extensions: [[$class: 'CloneOption', timeout: 120]],
                submoduleCfg: [],
                userRemoteConfigs: [[credentialsId: "${GIT_CREDENTIAL}", url: "${LIBRARY_REPO}"]]
            ])}}}

        stage('Build') {
            steps {
                opdSlackNotify2('#008FFF', 'Building', COMMIT_HASH)
                echo "Building docker image..."
                script {
                   withAWS(credentials: 'opd2-aws', region: 'ap-southeast-1') {
                       echo "Start building Docker image"
                       sh "aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin ${env.DOCKER_REGISTRY}"
                       def img = docker.build("${env.IMAGE_NAME}:${env.IMAGE_TAG}", "-f ./compose/production/django/Dockerfile .")
                       img.push()
                       img.push("${env.IMAGE_TAG_LATEST}")
                       echo "Removing docker image ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                       sh "docker rmi ${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                    }
                }
            }
        }
        stage('Deploy') {
            parallel {
                stage('develop') {
                    when {
                         branch 'develop'
                    }
                    steps {
                        opdSlackNotify2('#008FFF', 'Deploy', COMMIT_HASH)  
                        echo "Deployment Application..."
                            withAWS(credentials: 'opd2-aws', region: 'ap-southeast-1') {
                            echo 'authen with aws credentials successfully'
                            withKubeConfig(credentialsId: 'kube-config-develop') {
                               sh "kubectl set image deployment/${env.PROJECT_NAME} ${env.PROJECT_NAME}=${env.IMAGE_NAME}:${env.IMAGE_TAG} --record -n develop"
                            }
                        }
                    }
                }
                stage('UAT') {
                    when {
                        tag "uat-v.*"
                    }
                    steps {
                        opdSlackNotify2('#008FFF', 'Deploy', COMMIT_HASH)  
                        echo "Deployment Application..."
                            withAWS(credentials: 'opd2-aws', region: 'ap-southeast-1') {
                            echo 'authen with aws credentials successfully'
                            withKubeConfig(credentialsId: 'kube-config-develop') {
                               sh "kubectl set image deployment/${env.PROJECT_NAME} ${env.PROJECT_NAME}=${env.IMAGE_NAME}:${env.IMAGE_TAG} --record -n uat"
                            }
                        }    
                    }
                }
            }
        }
        stage('Deliver') {
            when {
                tag "release-v.*"
            }
            steps {
                opdSlackNotify2('#008FFF', 'Waiting for input...', COMMIT_HASH)
                timeout(time: 2, unit: 'HOURS') {
                    input(message:"Do you want to deploy tag ${env.BRANCH_NAME} into production?",ok:'I know what I am doing!',submitter:'Eittipat,Dew,Big')
                }
                withAWS(credentials: 'opd2-aws', region: 'ap-southeast-1') {
                    echo 'authen with aws credentials successfully'
                    withKubeConfig(credentialsId: 'kube-config-prod') {
                       sh "kubectl set image deployment/${env.PROJECT_NAME} ${env.PROJECT_NAME}=${env.IMAGE_NAME}:${env.IMAGE_TAG} --record -n production"
                    }
                }
            }
        }
    }
}
