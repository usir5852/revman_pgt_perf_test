
pipeline {
    agent {
            kubernetes {
                  yamlFile 'Jenkins-Slave-Pod.yaml'  // path to the pod definition relative to the root of our project
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
                              sh 'echo =======================Start deploy JMeter Slaves==============='
//                            sh 'helm init --client-only'
//                            sh 'helm repo update'
                              sh 'helm install --wait stable/distributed-jmeter --name distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} --set server.replicaCount=${noOfSlaveNodes},master.replicaCount=0,image.repository=gvasanka/jmeter-plugins,image.tag=5.1.1,image.pullPolicy=Always'
                              sh 'kubectl wait --for=condition=ready pods -l app.kubernetes.io/instance=distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                              sh 'echo =======================Finishing deploy JMeter Slaves==============='
                        }
                    }
            }
            stage('Search Slave IP details') {
                    steps {
                        container('kubehelm'){
                            script{
                                  print "=================Start search for slave IP details====================="
                                  print "Searching for Jmeter Slave IPs"
                                  env.jenkinsSlaveNodes = sh(returnStdout: true, script:'kubectl get pods -l app.kubernetes.io/instance=distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} -o jsonpath=\'{.items[*].status.podIP}\' | tr \' \' \',\'')
                                  println("IP Details: ${env.jenkinsSlaveNodes}")
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
                        sh 'for pod in $(kubectl get pod -l app.kubernetes.io/instance=distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} -o custom-columns=:metadata.name); do kubectl cp src/test/data/ $pod:/opt/perf-test-data;done;'
                        sh 'sleep 10m'
                        sh 'echo ===============Finishing copying data files======================='
                    }
                }
            }
            stage('Execute Performance Test') {
                steps {
                    container('maven'){
                        sh 'echo ===============Start maven build execution======================='
                        sh 'echo ${jenkinsSlaveNodes}'
                        sh 'mvn clean install -DjenkinsSlaveNodes=${jenkinsSlaveNodes}'
                        sh 'echo ===============Finishing maven build execution======================='
                    }
                }
            }
            stage('Read Performance Test Results') {
                steps {
                    sh 'echo ===============Start read Performance Test Results======================='
                    sh 'pwd'
                    perfReport 'target/jmeter/results/httpCounterDocker.csv'
                    sh 'echo ===============Finishing Performance Test Results======================='
                }
            }
            stage('Erase JMeter Slaves') {
                      steps {
                            container('kubehelm'){
                                 sh 'echo ==============Start Erasing JMeter Slaves========================'
                                 sh 'helm delete --purge distributed-jmeter-${JOBNAME}-${BUILD_NUMBER}'
                                 sh 'kubectl wait --for=delete pods -l app.kubernetes.io/instance=distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                                 sh 'echo ===============Finishing Erasing JMeter Slaves======================='
                            }
                      }
            }
    }

    post {
            unsuccessful {
                sh 'echo ==============Start post failure clearing =============='
                container('kubehelm'){
                    sh 'helm delete --purge distributed-jmeter-${JOBNAME}-${BUILD_NUMBER}'
                    sh 'kubectl wait --for=delete pods -l app.kubernetes.io/instance=distributed-jmeter-${JOBNAME}-${BUILD_NUMBER} --timeout=90s'
                }
                sh 'echo ==============Finishing post failure clearing=============='
            }
    }
}