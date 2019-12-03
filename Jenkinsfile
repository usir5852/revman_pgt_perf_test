pipeline {
    agent {
            kubernetes {
                  yamlFile 'build-pod.yaml'  // path to the pod definition relative to the root of our project
             }
    }

    environment {
            JOBNAME = "jobname"
    }

    parameters {
            string(defaultValue: "1", description: 'How many slave required ?', name: 'noOfSlaveNodes')
    }

    stages {
            stage('Deploy JMeter Slaves') {
                   steps {
                        container('kubehelm'){
                              sh 'echo ======================================'
                              sh 'helm install stable/distributed-jmeter --name distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} --set server.replicaCount=${noOfSlaveNodes},master.replicaCount=0'
                              sh 'sleep 5'
                              sh 'echo ======================================'
                        }
                    }
            }
            stage('Search Slave IP details') {
                    steps {
                        container('kubehelm'){
                            script{
                                  print "======================================"
                                  print "Searching for Jmeter Slave IPs"
                                  env.jenkinsSlaveNodes = sh(returnStdout: true, script:'kubectl get pods -l app.kubernetes.io/instance=distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} -o jsonpath=\'{.items[*].status.podIP}\' | tr \' \' \',\'')
                                  println("IP Details: ${env.jenkinsSlaveNodes}")
                                  print "======================================"
                            }
                        }
                    }
             }
            stage('Execute Performance Test') {
                steps {
                    container('maven'){
                        sh 'echo ======================================'
                        sh 'echo ${jenkinsSlaveNodes}'
                        sh 'mvn clean install \"-DjenkinsSlaveNodes=${jenkinsSlaveNodes}\"'
                        sh 'echo ======================================'
                    }
                }
            }
            stage('Read Performance Test Results') {
                steps {
                    sh 'pwd'
                    perfReport 'target/jmeter/results/httpCounterDocker.csv'
                }
            }
            stage('Erase JMeter Slaves') {
                      steps {
                            container('kubehelm'){
                                 sh 'echo ======================================'
                                 sh 'helm delete --purge distributed-jmeter-${JOBNAME}-${BUILD_NUMBER}'
                                 sh 'sleep 5'
                                 sh 'echo ======================================'
                            }
                      }
            }
    }
}