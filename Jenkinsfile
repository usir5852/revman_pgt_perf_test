
pipeline {
    agent {
            kubernetes {
                  yamlFile 'Jenkins-Slave-Pod.yaml'  // path to the pod definition relative to the root of our project
             }
    }

    environment {
//    Project On-board TASK 1::
//          Preferred project name can be given here as JOBNAME, But make sure to use only lower case letters (a-z) and digits (0-9) on your names. Ex brakes1
//          Restriction comes from Kubernetes side, Kubernetes only allow digits (0-9), lower case letters (a-z), -, and . characters in resource names https://kubernetes.io/docs/concepts/overview/working-with-objects/names/
            JOBNAME = "sprintdemo"
            scriptName = "httpCounterDocker"
    }

    stages {
            stage('Get properties file from user and send to workspace') {
                steps {
                      node('master') {
                            script{
                                    // Get file using input step, will put it in build directory
                                    print "=================Please upload your property files ====================="
                                    def inputFile = input message: 'Upload file', parameters: [file(name: 'global.properties')]
                                    // Read contents and write to workspace
                                    writeFile(file: 'global.properties', text: inputFile.readToString())
                                    // Stash it for use in a different part of the pipeline
                                    stash name: 'data', includes: 'global.properties'
                            }
                      }
                      container('kubehelm'){
                             script{
                                  print "=================Get Jenkins master Name====================="
                                  env.jenkinsMasterPodName = sh(returnStdout: true, script:'kubectl get pods -l app.kubernetes.io/component=jenkins-master -o jsonpath=\'{.items[*]..metadata.name}\'')
                                  println("Jenkins Pod Name Details: ${env.jenkinsMasterPodName}")


                                  print "===================Start copy property file==================="
                                  sh 'kubectl cp ${jenkinsMasterPodName}:/var/jenkins_home/workspace/${JOB_NAME}/global.properties src/test/property/global.properties'
                                  print "===================Finish copy property file==================="
                              }

                      }
                }
            }

            stage('Deploy JMeter Slaves') {
                  steps {
                        container('kubehelm'){
                              sh 'echo =======================Start deploy JMeter Slaves==============='
                              sh 'helm install --wait custom/jmeter-slave --name distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -f JMeter-Slave-Pod-Values.yaml'
                              sh 'kubectl wait --for=condition=ready pods -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                              sh 'echo =======================Finishing deploy JMeter Slaves==============='
                        }
                   }
            }
            stage('Get JMeter SlaveNodes IP details') {
                    steps {
                        container('kubehelm'){
                            script{
                                  print "=================Start search for slave IP details====================="
                                  print "Searching for Jmeter Slave IPs"
                                  env.jmeterSlaveNodes = sh(returnStdout: true, script:'kubectl get pods -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -o jsonpath=\'{.items[*].status.podIP}\' | tr \' \' \',\'')
                                  println("IP Details: ${env.jmeterSlaveNodes}")
                                  print "===================Finishing search for slave IP details==================="
                            }
                        }
                    }
            }
            stage('Copying data files to JMeter Slaves') {
                steps {
                    container('kubehelm'){
                        sh 'echo ===============Start copying data files======================='
                        sh 'pwd'
//    Project On-board TASK 5::
//    Following script copy the JMeter test data files to JMeter slaves, so make sure data file locations defined correctly, If you followed standard project structure nothing to change here.
                        sh 'for pod in $(kubectl get pod -l app.kubernetes.io/instance=distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -o custom-columns=:metadata.name); do kubectl cp src/test/data/ $pod:/opt/perf-test-data;done;'
                        sh 'echo ===============Finishing copying data files======================='
                    }
                }
            }
            stage('Execute Performance Test') {
                steps {
                    container('distributed-jmeter-master'){
                        sh 'echo ===============Start maven build execution======================='
                        sh 'echo ${jmeterSlaveNodes}'
                        sh '''mvn clean install -DjmeterSlaveNodes=${jmeterSlaveNodes} -DscriptName=${scriptName}'''
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
                                 sh 'sleep 30m'
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