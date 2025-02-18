pipeline {
    agent any
    stages {
        // DESCARGA CÓDIGO FUENTE
	stage('Get Code') {
            steps {
                sh 'whoami ; hostname ; hostname -I; uname -a'      
                git branch: 'master', url:'https://github.com/beetlebum97/todo-list-aws.git'   
                echo "WORKSPACE: ${env.WORKSPACE}"
                sh 'git rev-parse --abbrev-ref HEAD'
                sh 'ls -la'
            }
        }
        // DESPLIEGUE SAM
        stage('Deploy') {
            steps {     
                sh '''
                sam delete --stack-name todo-list-aws-production --no-prompts  
                sam deploy \
                    --config-file samconfig.toml \
                    --config-env production \
                    --no-confirm-changeset \
                    --no-fail-on-empty-changeset | tee deploy_output.txt
                '''
            }
        }
        // PRUEBAS DE INTEGRACIÓN
        stage('Rest Test') {
            steps {
                sh '''
                    BASE_URL=$(awk '/Key *BaseUrlApi/{getline; getline; print $2}' deploy_output.txt)
                    export BASE_URL=${BASE_URL}
                    pytest --junitxml=result-unit.xml -k "test_api_listtodos or test_api_gettodo" test/integration/todoApiTest.py
                '''
                junit 'result-unit.xml'
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
