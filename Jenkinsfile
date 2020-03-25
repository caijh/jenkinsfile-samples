pipeline {
    agent any

    parameters {
        gitParameter(name: 'BRANCH_TAG', type: 'PT_BRANCH_TAG', defaultValue: 'master', description: '分支')  
        choice(name: 'project', choices: ['hjmos-registercenter', 'hjmos-schedulecenter', 'hjmos-gateway', 'hjmos-authcenter', 'hjmos-webim'], description: '选择编译项目')
        string(name: 'version', defaultValue: '1.0.00', description: 'app版本，生成镜像的tag')
        string(name: 'docker_registry', defaultValue: '10.38.2.12:1000', description: '私有仓库')
    }


    tools {
        maven 'Default'
        git 'Default'
    }

    environment {
        app_dir = "${params.project}"
        app_name = "${params.project}"
        app_version = "${params.version}"
        docker_registry = "${docker_registry}"
    }

    stages {
        stage("Pull") {
            steps {
                checkout([$class: 'GitSCM', branches: [[name: "${params.BRANCH_TAG}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'caijunhui-gerrit', url: 'ssh://caijunhui@172.25.10.128:29418/hjmos_FramePlat']]])
            }
        }

        stage('Build App') {
            steps {
                withMaven(maven: 'Default') {
                    sh "mvn -U clean package -f $app_dir/pom.xml"
                }
            }
        }

        stage('Build Image') {
            steps {
                dir("$app_dir") {
                    script {
                        def dockerfile_exists = fileExists 'Dockerfile'
                        if (!dockerfile_exists) {
                            writeFile encoding: 'utf-8', file: 'Dockerfile', text: '''FROM 10.38.2.12:1000/library/openjdk:8-jdk-alpine
                                RUN echo "Asia/Shanghai" > /etc/timezone
                                ARG JAR_FILE
                                COPY target/${JAR_FILE} app.jar
                                ENV JAVA_OPS="-Djava.security.egd=file:/dev/./urandom"
                                ENV APP_OPS=""
                                ENTRYPOINT ["sh","-c","java -jar $JAVA_OPS /app.jar  $APP_OPS"]'''
                        }
                    }
                    sh "docker build -t $docker_registry/library/$app_name:$app_version --build-arg JAR_FILE=${app_name}.jar ."
                }
            }
        }

        stage('Publish Image') {
            steps {
                script {
                    def new_image_name = "$docker_registry/library/$app_name:$app_version"
                    withCredentials([usernamePassword(credentialsId: 'docker-registry', passwordVariable: 'dockerPassword', usernameVariable: 'dockerUser')]) {
                        sh "docker login -u ${dockerUser} -p ${dockerPassword} ${docker_registry}"
                        sh "docker push $docker_registry/library/$app_name:$app_version"
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                dir("$app_dir") {
                    script {
                        def dockerfile_exists = fileExists 'Dockerfile'
                        if (dockerfile_exists) {
                            sh 'rm -rf Dockerfile'
                        }
                    }
                }
            }
        }
        success {
            sh "docker ps -a|grep Exited|awk '{print \$1}'|xargs -I {} docker rm {}"
            sh "docker images|grep '<none>'|awk '{print \$3}'|xargs -I {} docker image rm {} > /dev/null 2>&1 || true"
        }
    }

}