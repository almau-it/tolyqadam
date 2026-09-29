pipeline {
    agent { label 'sites2' }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        DEPLOY_PATH = '/var/www/tolyqadam'
    }

    stages {
        stage('Verify') {
            steps {
                sh '''
                    test -f index.html
                    test -f redesign.css
                    test -d assets
                    test -x update.sh
                    echo "Sanity check passed: site files are in place"
                '''
            }
        }

        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                // Агент sites2 стоит на самом веб-сервере: деплой локальный,
                // update.sh делает git pull в /var/www/tolyqadam и перезагружает nginx
                sh '${DEPLOY_PATH}/update.sh'
            }
        }
    }

    post {
        success {
            echo "Deployed ${env.GIT_COMMIT} to ta.commit.kz"
        }
        failure {
            echo 'Deploy failed — check the update.sh output above'
        }
    }
}
