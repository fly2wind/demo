#!/usr/bin/env groovy

node {
    stage('checkout') {
        checkout scm
    }

    docker.image('openjdk:8').inside('-u root -e GRADLE_USER_HOME=.gradle') {
        stage('check java') {
            sh "java -version"
        }

        stage('clean') {
            sh "chmod +x gradlew"
            sh "./gradlew clean --no-daemon"
        }

        stage('prepare environment') {
            sh "./gradlew npmInstall -PnodeInstall"
        }

        stage('backend tests') {
            try {
                sh "./gradlew test -PnodeInstall"
            } catch(err) {
                throw err
            } finally {
                junit '**/build/**/TEST-*.xml'
            }
        }

        stage('frontend tests') {
            try {
                sh "./gradlew npm_test -PnodeInstall"
            } catch(err) {
                throw err
            } finally {
                junit '**/build/test-results/karma/TESTS-*.xml'
            }
        }

        stage('packaging') {
            sh "./gradlew bootRepackage -x test -Pprod -PnodeInstall"
            archiveArtifacts artifacts: '**/build/libs/*.war', fingerprint: true
        }

        stage('quality analysis') {
            withSonarQubeEnv('Sonar') {
                sh "./gradlew sonarqube"
            }
        }
    }

    def dockerImage
    stage('build docker') {
        sh "cp -R src/main/docker build/"
        sh "cp build/libs/*.war build/docker/"
        dockerImage = docker.build('demo', 'build/docker')
    }

    stage('publish docker') {
        docker.withRegistry('http://thoughtworks.io:5001', 'registry-login') {
            dockerImage.push 'latest'
        }
    }


    docker.image('identt/rancher-compose').inside("-e COMPOSE_PROJECT_NAME=demo") {
        stage('deploy uat') {
          sh "rancher-compose --url http://10.202.128.107:8080/v2-beta/projects/1a118 --access-key E4ADDAB2FB34352E015C --secret-key 4ErqRBJdJQVs62VDJ43MwMQV8iYp9xoJupBJ29YU up -p -d -c --upgrade"
        }
    }


}
