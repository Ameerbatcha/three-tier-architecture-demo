pipeline {
    agent any

    tools {
        maven 'maven'
        nodejs 'nodejs'
        git 'Default'
    }

    parameters {
        choice(
            name: 'SERVICE_NAME',
            choices: ['catalogue', 'user', 'cart', 'shipping', 'ratings', 'payment', 'dispatch', 'mongo', 'mysql', 'web', 'ALL'],
            description: 'Select the service to deploy'
        )
    }

    environment {
        DOCKER_TAG = getVersion()
        SONAR_TOKEN = credentials('sonarqube-auth')
    }

    stages {
        stage('SCM') {
            steps {
                 // script{
                 //    echo "${DOCKER_TAG} SAMPLEWORDING "    
                 //    }
                deleteDir()
                git 'https://github.com/Ameerbatcha/three-tier-architecture-demo.git'
            }
        }

        stage('SAST') {
            steps {
                script {
                    def services = []
                    if (params.SERVICE_NAME == 'ALL') {
                        services = ['catalogue', 'user', 'cart', 'shipping', 'ratings', 'payment', 'dispatch', 'mongo', 'mysql', 'web']
                    } else {
                        services = [params.SERVICE_NAME]
                    }

                    for (String svc : services) {
                        dir("${svc}") {
                            if (fileExists('pom.xml')) {
                                // Java service
                                sh 'mvn clean compile'
                                withSonarQubeEnv('sonar') {
                                    sh 'mvn sonar:sonar'
                                }
                            } else if (fileExists('package.json')) {
                                // Node.js service
                                // Copy files to the remote server
                                sshPublisher(publishers: [
                                    sshPublisherDesc(
                                        configName: 'worker1',
                                        transfers: [
                                            sshTransfer(
                                                sourceFiles: '**/*',
                                                removePrefix: '',
                                                remoteDirectory: "/opt/docker/${svc}",
                                                execCommand: """
                                                    cd /opt/docker/${svc}
                                                    npm install
                                                    echo 'sonar.projectKey=${svc}' > sonar-scanner.properties
                                                    echo 'sonar.sources=.' >> sonar-scanner.properties
                                                    WORKER_IP=\$(hostname -I | awk '{print \$1}')
                                                    echo "sonar.host.url=http://65.2.182.153:9000" >> sonar-scanner.properties
                                                    echo 'sonar.login=${SONAR_TOKEN}' >> sonar-scanner.properties
                                                    sonar-scanner -Dsonar.projectKey=${svc} -Dsonar.login=${SONAR_TOKEN} -Dproject.settings=sonar-scanner.properties
                                                """
                                            )
                                        ],
                                        usePromotionTimestamp: false,
                                        verbose: true
                                    )
                                ])
                            } else if (fileExists('requirements.txt')) {
                                // Python service
                                sh 'pip install -r requirements.txt'
                                sh 'pip install pylint'  // Ensure pylint is installed

                                withSonarQubeEnv('sonar') {
                                    writeFile file: 'sonar-project.properties', text: """
                                    sonar.projectKey=${svc}
                                    sonar.sources=.
                                    sonar.python.pylint.reportPaths=pylint-report.txt
                                    """
                                    sh 'pylint $(find . -type f -name "*.py") > pylint-report.txt || true'
                                    withEnv(["PATH+EXTRA=/usr/local/bin/sonar-scanner-6.0.0.4432-linux/bin"]) {
                                        sh 'sonar-scanner'
                                    }
                                }
                            } else {
                                echo "No known project type found for ${svc}"
                            }
                        }
                    }
                }
            }
        }

        stage('Bundling') {
            steps {
                sh 'tar czf Bundle.tar.gz *'
            }
        }

        stage('Build and push Image') {
            steps {
                script {
                    sh "echo ${DOCKER_TAG}"
                    withCredentials([usernamePassword(credentialsId: 'docker_hub_DSO', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        def services = []
                        if (params.SERVICE_NAME == 'ALL') {
                            services = ['cart', 'catalogue', 'dispatch', 'mongo', 'mysql', 'payment', 'user', 'shipping', 'ratings', 'web']
                        } else {
                            services = [params.SERVICE_NAME]
                        }

                        for (String svc : services) {
                            sshPublisher(publishers: [
                                sshPublisherDesc(
                                    configName: 'worker1',
                                    transfers: [
                                        sshTransfer(
                                            cleanRemote: false,
                                            excludes: '',
                                            execCommand: """
                                                cd /opt/docker;
                                                tar -xf Bundle.tar.gz ${svc};
                                                cd ${svc};
                                                docker build . -t securityanddevops/rs-${svc}:${BUILD_NUMBER}-${DOCKER_TAG}
                                                trivy image --severity CRITICAL --exit-code 1 securityanddevops/rs-${svc}:${BUILD_NUMBER}-${DOCKER_TAG}
                                                docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                                                docker tag securityanddevops/rs-${svc}:${BUILD_NUMBER}-${DOCKER_TAG} securityanddevops/rs-${svc}:latest
                                                docker push securityanddevops/rs-${svc}:${BUILD_NUMBER}-${DOCKER_TAG}
                                                docker push securityanddevops/rs-${svc}:latest
                                                docker rmi securityanddevops/rs-${svc}:${BUILD_NUMBER}-${DOCKER_TAG}
                                            """,
                                            execTimeout: 2000000,
                                            flatten: false,
                                            makeEmptyDirs: false,
                                            noDefaultExcludes: false,
                                            patternSeparator: '[, ]+$',
                                            remoteDirectory: '//opt//docker',
                                            remoteDirectorySDF: false,
                                            removePrefix: '',
                                            sourceFiles: '*.gz'
                                        )
                                    ],
                                    usePromotionTimestamp: false,
                                    useWorkspaceInPromotion: false,
                                    verbose: true
                                )
                            ])
                        }
                    }
                }
            }
        }
    stage('Deploy to Testing Namespace') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'worker1',
                        transfers: [
                            sshTransfer(
                                execCommand: """
                                    cd /opt/three-tier-architecture-demo-instana
                                    git config --global --add safe.directory /opt/three-tier-architecture-demo-instana
                                    git stash
                                    git pull -f https://github.com/Thoshinny-cyber/three-tier-architecture-demo-instana.git
                                    cd EKS/helm
                                    helm upgrade --install robot-shop --namespace testing .
                                    kubectl apply -f ingress-testing.yaml

                                """,
                                execTimeout: 2000000,
                                removePrefix: '',
                                remoteDirectory: '',
                                sourceFiles: ''
                            )
                        ],
                        usePromotionTimestamp: false,
                        verbose: true
                    )
                ])
            }
        }
                stage('Wait for Pods to be Ready') {
    steps {
        sshPublisher(publishers: [
            sshPublisherDesc(
                configName: 'worker1',
                transfers: [
                    sshTransfer(
                        execCommand: """
                            timeout=180
                            interval=30
                            elapsed=0
                            while [[ \$elapsed -lt \$timeout ]]; do
                                total_pods=\$(kubectl get pods -n testing --no-headers | wc -l)
                                running_pods=\$(kubectl get pods -n testing --field-selector=status.phase=Running --no-headers | wc -l)
                                if [[ "\$total_pods" -eq "\$running_pods" && "\$total_pods" -ne 0 ]]; then
                                    echo "All pods are ready"
                                    exit 0
                                fi
                                sleep \$interval
                                elapsed=\$((elapsed + interval))
                            done
                            echo "Pods did not become ready in time"
                            exit 1
                        """,
                        execTimeout: 20000000,
                        removePrefix: '',
                        remoteDirectory: '',
                        sourceFiles: ''
                    )
                ],
                usePromotionTimestamp: false,
                verbose: true
            )
        ])
    }
}
        //  stage('Wait for Pods to be Ready') {
        //     steps {
        //         sshPublisher(publishers: [
        //             sshPublisherDesc(
        //                 configName: 'Bootstrap_Server',
        //                 transfers: [
        //                     sshTransfer(
        //                         execCommand: """
        //                             timeout=180
        //                             interval=30
        //                             elapsed=0
        //                             while [[ \$elapsed -lt \$timeout ]]; do
        //                                 if kubectl get pods -n testing --field-selector=status.phase=Running | grep -q 'Running'; then
        //                                     echo "Pods are ready"
        //                                     exit 0
        //                                 fi
        //                                 sleep \$interval
        //                                 elapsed=\$((elapsed + interval))
        //                             done
        //                             echo "Pods did not become ready in time"
        //                             exit 1
        //                         """,
        //                         execTimeout: 2000000,
        //                         removePrefix: '',
        //                         remoteDirectory: '',
        //                         sourceFiles: ''
        //                     )
        //                 ],
        //                 usePromotionTimestamp: false,
        //                 verbose: true
        //             )
        //         ])
        //     }
        // }

        stage('Run Automation Tests') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'worker1',
                        transfers: [
                            sshTransfer(
                                execCommand: """
                                    cd /opt/tests
                                    ./run_tests.sh
                                """,
                                execTimeout: 2000000,
                                removePrefix: '',
                                remoteDirectory: '',
                                sourceFiles: ''
                            )
                        ],
                        usePromotionTimestamp: false,
                        verbose: true
                    )
                ])
            }
        }

        stage('Deploy to UAT Namespace') {
            // when {
            //     expression {
            //         return currentBuild.result == null || currentBuild.result == 'SUCCESS'
            //     }
            // }
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'worker1',
                        transfers: [
                            sshTransfer(
                                execCommand: """
                                    cd /opt/three-tier-architecture-demo-instana
                                    cd EKS/helm
                                    helm upgrade --install robot-shop --namespace uat .
                                    kubectl apply -f ingress-uat.yaml
                            
                                """,
                                execTimeout: 2000000,
                                removePrefix: '',
                                remoteDirectory: '',
                                sourceFiles: ''
                            )
                        ],
                        usePromotionTimestamp: false,
                        verbose: true
                    )
                ])
            }
        }
    }
}
def getVersion() {
    def commitHash = sh(label: '', returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    return commitHash
}


