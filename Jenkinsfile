pipeline {
    agent any


  post {
    always {
      deleteDir()
    }
  }

    stages {
//Clean Workspace se sustituye por lo anterior. Y el getCode con la configuración de checkout scm propia de jenkins
//        stage('Clean workspace') {
//            steps { deleteDir() }
//        }
//
//        stage('Get Code') {
//            steps {
//                git branch: 'develop',
//                    url: 'https://github.com/JesusCanor/jenkins-unir.git'
//           }
//        }

        stage('Tests') {
            parallel {
                stage('Unit') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') { //Siempre verde
                            sh '''
                                export PYTHONPATH=$WORKSPACE
                                pytest test/unit --junitxml=result-unit.xml --cov=. --cov-branch --cov-report=xml:result-coverage.xml
                            '''
                            junit testResults: 'result-unit.xml', allowEmptyResults: true
                        }
                   }
                }
                stage('Rest') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') { //Siempre verde
                            sh '''
                                export FLASK_APP=app/api.py
                                flask run &
                                FLASK_PID=$!

                                java -jar /opt/wiremock/wiremock-standalone.jar \
                                --port 9090 \
                                --root-dir test/wiremock &

                                sleep 1
                                # Esperar que Wiremock esté levantado
                                for i in $(seq 1 5); do
                                if curl -sSf http://127.0.0.1:9090/__admin >/dev/null; then
                                    echo "Wiremock listo"
                                    break
                                fi
                                sleep 1
                                done

                                # Wiremock falla al arrancar
                                curl -sSf http://127.0.0.1:9090/__admin >/dev/null || {
                                echo "WireMock NO arrancó. Log:";
                                tail -n 200 wiremock.log || "Tail problem"
                                kill "$WIREMOCK_PID" || "Wiremock PID doesn't exist"
                                exit 1
                                }

                                #export PYTHONPATH="$WORKSPACE" NO NECESARIO
                                pytest --junitxml=result-rest.xml test/rest
                                kill $FLASK_PID
                            '''
                        }
                    }
                }
            }
        }

        stage('Static') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                //Evita que el pipeline se pare si ocurre un error o FAILURE
                //No utilizo UNSTABLE, para no "tapar" el FAILURE real de los qualityGates
                    sh '''
                        flake8 --exit-zero --format=pylint app > flake8.out
                    '''
                    recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')],
                        qualityGates: [
                            [threshold: 8, type: 'TOTAL', unstable: true],
                            [threshold: 10, type: 'TOTAL', unstable: false]
                        ]
                }
            }
        }

        stage('Security') {
            steps{
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                //Evita que el pipeline se pare si ocurre un error o FAILURE
                //No utilizo UNSTABLE, para no "tapar" el FAILUREreal de los qualityGates
                    sh '''
                        bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                    '''
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')], 
                        qualityGates: [
                            [threshold: 2, type: 'TOTAL', unstable: true],
                            [threshold: 4, type: 'TOTAL', unstable: false]
                        ]
                }
            }
        }

        stage('Coverage') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') { 
                //Evita que el pipeline se pare si ocurre un error o FAILURE
                //No utilizo UNSTABLE, para no "tapar" el FAILUREreal de los qualityGates
                    recordCoverage qualityGates: [[criticality: 'ERROR', integerThreshold: 85, metric: 'LINE', threshold: 85.0],
                                                [integerThreshold: 95, metric: 'LINE', threshold: 95.0],
                                                [criticality: 'ERROR', integerThreshold: 80, metric: 'BRANCH', threshold: 80.0],
                                                [integerThreshold: 90, metric: 'BRANCH', threshold: 90.0]], 
                                                tools: [[parser: 'COBERTURA', pattern: 'result-coverage.xml']]
                }
            }
        }

        stage('Performance') {
            steps {
                //catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') 
                // En este caso, al no existir qualityGates, se podría utilizar buildResult UNSTABLE, para dejar esa "trazabilidad" de que algo ha ido mal, sin dejar un FAILURE. permitiendo que continúe el pipeline
                    sh '''
                        export FLASK_APP=app/api.py
                        flask run &
                        FLASK_PID=$!
                        
                        sleep 1
                        # Esperar que FLASK esté levantado
                        for i in $(seq 1 5); do
                          if curl -sSf http://127.0.0.1:5000/health-check >/dev/null; then
                            echo "Flask listo"
                            break
                          fi
                          sleep 1
                        done
                  
                        # Flask falla al arrancar
                        curl -sSf http://127.0.0.1:5000/health-check >/dev/null || {
                          echo "Flask NO arrancó";
                          kill "$FLASK_PID" || "FLASK PID doesn't exist"
                          exit 1
                        }

                        OUT="$WORKSPACE/test/jmeter/flask-$BUILD_NUMBER.jtl"

                        # Diagnóstico y blindaje
                        rm -f "$WORKSPACE/test/jmeter/flask-"*.jtl
                        jmeter -n -t "$WORKSPACE/test/jmeter/Performance Test.jmx" -l "$OUT"
                        kill "$FLASK_PID" || true
                    '''
                perfReport sourceDataFiles: 'test/jmeter/flask-*.jtl'
            }
        }
    }
}