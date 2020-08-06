
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
    }

    parameters {
//    Project On-board TASK 2::   define how many JMeter slave nodes you required for the test
//             file(description: 'Upload your JMeter Parameter file', name: 'parameterFile.properties')

//    Project On-board TASK 3::    define JMeter performance script name you want to execute
            string(defaultValue: "httpCounterDocker", description: 'which JMeter script you want to execute?', name: 'scriptName')

//    Project On-board TASK 4::
//          All the project related custom parameters should be define under here
    }

    stages {
             stage('Get Jenkins Slave Name') {
                   steps {
                        container('kubehelm'){
                             script{
                                  print "=================Get Jenkins Slave Name====================="
                                  println("IP Details: ${env.NODE_NAME}")
                                  print "===================Finishing Get Jenkins Slave Name==================="
                              }
                        }
                   }
             }

            stage('file input') {
                steps {
                      node('master') {
                            script{
                                    // Get file using input step, will put it in build directory
                                    def inputFile = input message: 'Upload file', parameters: [file(name: 'data.txt')]
                                    // Read contents and write to workspace
                                    writeFile(file: 'data.txt', text: inputFile.readToString())
                                    // Stash it for use in a different part of the pipeline
                                    stash name: 'data', includes: 'data.txt'
                            }
                      }
                }
            }

            stage('Deploy JMeter Slaves') {
                  steps {
                        container('kubehelm'){
                              sh 'echo =======================Start deploy JMeter Slaves==============='
//                            sh 'helm init --client-only'
//                            sh 'helm repo add custom https://gvasanka.github.io/jmeter-helm-chart'
//                            sh 'helm repo update'
                              sh 'helm install --wait custom/jmeter-slave --name distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} -f JMeter-Slave-Pod-Values.yaml'
//                               sh 'helm install --wait stable/distributed-jmeter --name distributed-jmeter-slave-${JOBNAME}-${BUILD_NUMBER} --set server.replicaCount=${noOfSlaveNodes},master.replicaCount=0,image.repository=gvasanka/jmeter-plugins,image.tag=5.1.1'
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
//    Project On-board TASK 6::
//                         Project maven build command have to define like below passing all the required custom parameters
//                         sh '''mvn clean install -DjenkinsSlaveNodes=${jmeterSlaveNodes} -DscriptName=${scriptName} -Dprotocol=${protocol} -DserverIP=${serverIP} \
//                                                      -DpUserData=${pUserData} -DpICThreadCount=${pICThreadCount} -DpICRampupTime=${pICRampupTime} -DpICStepCount=${pICStepCount} \
//                                                      -DpICDuration=${pICDuration} -DpVCThreadCount=${pVCThreadCount} -DpVCRampupTime=${pVCRampupTime} -DpVCStepCount=${pVCStepCount} \
//                                                      -DpVCDuration=${pVCDuration} -DpThinktime=${pThinktime} -Dsyy_itm_vnd_ui_master=${syy_itm_vnd_ui_master} -DloginWebUI=${loginWebUI} \
//                                                      -Dcframeworkservice=${cframeworkservice} -DpPacing=${pPacing} -Dsyy_itm_vnd_ui_master_approve=${syy_itm_vnd_ui_master_approve}  \
//                                                      -Dhost=${host} -DGenerated_Vendor_Namep=${Generated_Vendor_Namep} -DSTEP_ID=${STEP_ID} \
//                                                      -Dprojectbuild=${projectbuild} -Dprojectbuildversion=${projectbuildversion}'''
                        sh '''mvn clean install -DjmeterSlaveNodes=${jmeterSlaveNodes} -DscriptName=${scriptName}'''
//                         sh 'sleep 20m'
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