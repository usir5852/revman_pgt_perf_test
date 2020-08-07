pipeline {
    agent {
            kubernetes {
                  yamlFile 'perfPlatform/Jenkins-Slave-Pod.yaml'  // path to the jenkins slave pod definition relative to the root of our project
             }
    }

    environment {
//   Project On-board TASK 1::
//          Preferred project name can be given here as JOBNAME, But make sure to use only lower case letters (a-z) and digits (0-9) on your names. Ex brakes1
//          Restriction comes from Kubernetes side, Kubernetes only allow digits (0-9), lower case letters (a-z), -, and . characters in resource names https://kubernetes.io/docs/concepts/overview/working-with-objects/names/
            JOBNAME = "leansolutiondemo"
//   Project On-board TASK 2::
//          Provide JMeter script name you want to run here. Don't add .jmx extension
            scriptName = "httpCounterDocker"
    }

    stages {
            stage('Deploy JMeter Slaves') {
                  steps {
                        container('kubehelm'){
                              sh 'echo =======================Start deploy JMeter Slaves==============='
                              sh 'helm install --wait custom/jmeter-slave --name distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -f perfPlatform/JMeter-Slave-Pod-Values.yaml'
                              sh 'kubectl wait --for=condition=ready pods -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                              script{
                                env.jmeterSlaveNodesIPList = sh(returnStdout: true, script:'kubectl get pods -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -o jsonpath=\'{.items[*].status.podIP}\' | tr \' \' \',\'')
                                println("IP Details: ${env.jmeterSlaveNodesIPList}")
                              }

                              sh 'echo =======================Finishing deploy JMeter Slaves==============='
                        }
                   }
            }
            stage('Copying test data files to JMeter Slaves') {
                steps {
                    container('kubehelm'){
                        sh 'echo ===============Start copying data files======================='
                        sh 'pwd'
//    Project On-board TASK 3::
//    Following script copy the JMeter test data files to JMeter slaves, so make sure data file locations defined correctly, If you followed standard project structure nothing to change here.
                        sh 'for pod in $(kubectl get pod -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -o custom-columns=:metadata.name); do kubectl cp src/test/data/ $pod:/opt/perf-test-data/;done;'
                        sh 'echo ===============Finishing copying data files======================='
                    }
                }
            }
            stage('Execute Performance Test') {
                steps {
                    container('distributed-jmeter-master'){
                        sh 'echo ===============Start maven build execution======================='
                        sh 'echo ${jmeterSlaveNodesIPList}'
                        sh 'mvn clean install -DjmeterSlaveNodesIPList=${jmeterSlaveNodesIPList} -DscriptName=${scriptName}'
                        sh 'echo ===============Finishing maven build execution======================='
                    }
                }
            }
            stage('Read Performance Test Results') {
                steps {
                    container('distributed-jmeter-master'){
                        sh 'echo ===============Start read Performance Test Results======================='
                        sh 'pwd'
                        perfReport 'target/jmeter/results/*.csv'
                        sh 'echo ===============Finishing Performance Test Results======================='
                    }
                }
            }
            stage('Erase JMeter Slaves') {
                      steps {
                            container('kubehelm'){
//                               sh 'sleep 30m'
                                 sh 'echo ==============Start Erasing JMeter Slaves========================'
                                 sh 'helm delete --purge distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER}'
                                 sh 'kubectl wait --for=delete pods -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                                 sh 'echo ===============Finishing Erasing JMeter Slaves======================='
                            }
                      }
            }
    }

    post {
            unsuccessful {
                sh 'echo ==============Start post failure clearing =============='
                container('kubehelm'){
                    sh 'helm delete --purge distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER}'
                    sh 'kubectl wait --for=delete pods -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                }
                sh 'echo ==============Finishing post failure clearing=============='
            }
    }
}