pipeline {
    agent any
    stages {
        // DESCARGA CÓDIGO FUENTE
        stage('Get Code') {
            steps {
                sh 'whoami ; hostname ; hostname -I; uname -a'
                git branch: 'develop', url:'https://github.com/beetlebum97/todo-list-aws.git'
                echo "WORKSPACE: ${env.WORKSPACE}"
                sh 'git rev-parse --abbrev-ref HEAD'
                sh 'ls -la'
            }
        }
        // PRUEBAS ESTÁTICAS
        stage('Static Test') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    // FLAKE8
                    sh 'flake8 --format=pylint --exit-zero src > flake8.out'
                    recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')]

                    // BANDIT
                    sh 'bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
                }
            }
        }
        // DESPLIEGUE SAM
        stage('Deploy') {
            steps {
                sh '''
                sam delete --stack-name todo-list-aws-staging --no-prompts
                sam deploy \
                    --config-file samconfig.toml \
                    --config-env staging \
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
                    pytest --junitxml=result-unit.xml test/integration/todoApiTest.py
                '''
                junit 'result-unit.xml'
            }
        }
        // FUSIÓN CON RAMA MASTER
        stage('Promote') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github-pass', variable: 'TOKEN'),
                                    string(credentialsId: 'github-usuario', variable: 'USUARIO'),
                                    string(credentialsId: 'github-email', variable: 'EMAIL')]) {
                        sh '''
                            # Limpiar outputs, cargar rama master y fusionar con develop
                            rm bandit.out flake8.out deploy_out.txt result-unit.xml
                            git fetch --all
                            git checkout master 2>/dev/null || git checkout -b master origin/master
                            git merge develop --no-commit --no-ff || true

                            # Preservar archivos Jenkinsfile de la rama master
                            git restore --source=HEAD --staged --worktree -- Jenkinsfile*
                            git add .
			    git status
			    git commit -m "CP1-D R5"

                            # Subir cambios al repositorio remoto
                            git config user.name "${USUARIO}"
                            git config user.email "${EMAIL}"
                            git push https://${TOKEN}@github.com/beetlebum97/todo-list-aws.git master
                        '''
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
