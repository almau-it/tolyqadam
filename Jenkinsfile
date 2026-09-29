pipeline {
    agent { label 'sites2' }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {
        stage('Verify') {
            steps {
                sh '''
                    test -f index.html
                    test -f redesign.css
                    test -d assets
                    echo "Sanity check passed: site files are in place"
                '''
            }
        }

        stage('Publish') {
            steps {
                // nginx раздаёт сайт прямо из этого workspace
                // (root /home/jenkins/agent/workspace/tolyqadam) —
                // достаточно сделать файлы читаемыми для www-data
                sh 'chmod -R a+rX "$WORKSPACE"'
            }
        }
    }

    post {
        success {
            echo "Deployed ${env.GIT_COMMIT} to https://tolyqadam.almau.edu.kz"
        }
        failure {
            echo 'Build failed — nginx keeps serving the previous workspace contents'
        }
    }
}
