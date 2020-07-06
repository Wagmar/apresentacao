pipeline {
    agent any

    tools {
        jdk 'OpenJDK 1.8'
    }

    options {
        timestamps()
        gitLabConnection('gl-con1')
        gitlabBuilds(builds: ['build','jacoco','sonarqube','create_control','copy_jar','create_service','create_insts','create_deb'])
    }

    stages {

        stage('Build') {

            steps {
                gitlabCommitStatus(name: 'build') {
                    sh './gradlew --info --stacktrace clean build'
                }
            }

            post {
                always {
                    junit 'build/test-results/test/*.xml'
                }
                success {
                    script {
                        def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                        currentBuild.description = 'v' + props['build.version']
                    }
                }
            }
        }
//
        stage('Code Coverage') {
            steps {
                gitlabCommitStatus(name: 'jacoco') {
                    step([$class: 'JacocoPublisher', execPattern: 'build/jacoco/test.exec',
                            classPattern: 'build/classes/java', sourcePattern: 'src/main/java'])
                }
            }
        }
//
        stage('Static Code Analysis') {
            steps {
                gitlabCommitStatus(name: 'sonarqube') {
                    withSonarQubeEnv('SonarQube Server') {
                        sh './gradlew --info --stacktrace sonarqube -x test'
                    }
                    timeout(time: 5, unit: 'MINUTES') {
                        script {
                            def qualitygate = waitForQualityGate()
//                             if (qualitygate.status != "OK") {
//                                 error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
//                             }
                        }
                    }
                }
            }
        }
//
//         stage('Publish') {
//             steps {
//                 gitlabCommitStatus(name: 'publish') {
//                     withCredentials([usernamePassword(credentialsId: 'rctiReleasesRepo',
//                                                       usernameVariable: 'ORG_GRADLE_PROJECT_deployerUsername',
//                                                       passwordVariable: 'ORG_GRADLE_PROJECT_deployerPassword')]) {
//                         sh './gradlew --info --stacktrace publish -x jar'
//                     }
//                 }
//             }
//         }

        stage('Create Control') {
            steps {
                gitlabCommitStatus(name: 'create_control') {
                script{
                        def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                        def version = props['build.version']
                        def artifactName = props['build.name']
                        def dirDebian = 'build/deploy/DEBIAN'
                        def content = """Package: $artifactName\n"""+"""Version: $version\n"""+"""Architecture: all\n"""+
                        """Maintainer: Riocard TI <desenvolvimento@riocardmais.com.br>\n"""+"""Depends: systemd, ca-certificates\n"""+
                        """Homepage: https://www.riocardmais.com.br\n"""+"""Description: Aplicacao $artifactName\n"""
                        sh """mkdir -p $dirDebian"""
                        writeFile file: "$dirDebian/control", text: """$content"""
                    }
                }
            }
        }

        stage('copyJar') {
            steps {
                gitlabCommitStatus(name: 'copy_jar') {
                script{
                        def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                        def name = props['build.name']
                        def artifactName = name+"-"+props['build.version']
                        def dirJar = """build/deploy/riocard/msa/$name"""
                        def dirLog = """build/deploy/riocard/logs/$name"""
                        sh """mkdir -p $dirJar"""
                        sh """mkdir -p $dirLog"""
                        sh """cp build/libs/$artifactName"""+""".jar build/deploy/riocard/msa/$name"""
                    }
                }
            }
        }

        stage('Create Service') {
            steps {
                gitlabCommitStatus(name: 'create_service') {
                script{
                    def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                    def name = props['build.name']
                    def dirConfig = 'build/deploy/etc/systemd/system'
                    def content = """[Unit]\n"""+
                                  """Description=$name\n"""+
                                  """BindTo=network.target\n"""+
                                  """After=network.target\n"""+
                                  """Requires=network.target\n\n"""+
                                  """[Service]\n"""+
                                  """Type=simple\n"""+
                                  """Environment=LANG=en_US.UTF-8\n"""+
                                  """Environment=JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/\n"""+
                                  """UMask=0002\n"""+
                                  """User=$name\n"""+
                                  """Group=$name\n"""+
                                  """WorkingDirectory=/riocard/msa/$name/\n"""+
                                  """StandardOutput=syslog\n"""+
                                  """StandardError=syslog\n"""+
                                  """SyslogIdentifier=$name\n"""+
                                  """ExecStart=""" +'\$'+ """JAVA_HOME/bin/java -Xms32m -Xmx128m -jar $name"""+""".jar\n"""+
                                  """Restart=on-failure\n\n"""+
                                  """[Install]\n"""+
                                  """WantedBy=multi-user.target\n"""
                    sh """mkdir -p $dirConfig"""
                    sh """echo '$content' > $dirConfig/$name"""+".service"
                    //writeFile file: """$dirConfig/$name"""+""".conf""", text: """$content"""
                    }
                }
            }
        }

        stage('Create Insts Files') {
            steps {
                gitlabCommitStatus(name: 'create_insts') {
                script{
                        def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                        def name = props['build.name']
                        def dirInst = 'build/deploy/DEBIAN'
                        def binBash = "#!/bin/bash\n"
                        def content = binBash+"""systemctl stop $name"""+""".service 2>/dev/null\n"""+
                                      """if [ ! \"\$(id $name 2>/dev/null)\" ]; then \n"""+
                                      """    useradd -s /bin/false -d /riocard/msa/$name --system $name \n"""+
                                      """fi\n"""

                        sh """mkdir -p $dirInst"""
                        sh """echo '$content' > $dirInst/preinst"""
                        sh """chmod 775 $dirInst/preinst"""
                        //create postinst
                        content = binBash+"""chown -R $name:$name /riocard/msa/$name \n"""+
                                          """chown -R $name:$name /riocard/logs/$name \n"""+
                                          """systemctl daemon-reload \n"""+
                                          """systemctl enable $name"""+""".service \n"""+
                                          """systemctl start $name"""+""".service \n"""
                        sh """echo '$content' > $dirInst/postinst"""
                        sh """chmod 775 $dirInst/postinst"""
                        //create postrm
                        content = binBash+"""systemctl stop $name"""+""".service \n"""+
                                          """systemctl disable  $name"""+""".service \n"""+
                                          """userdel $name \n"""
                        sh """echo '$content' > $dirInst/postrm"""
                        sh """chmod 775 $dirInst/postrm"""
                    }
                }
            }
        }

        stage('Create DEB') {
            steps {
                gitlabCommitStatus(name: 'create_deb') {
                script{
                        def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                        def name = props['build.name']
                        def version = props['build.version']
                        sh """dpkg-deb -b build/deploy build/$name-$version"""+""".deb"""
                        //dpkg-deb -b build/deploy build/apresentacao-0.0.1.deb
                        // # sudo dpkg -i msa000.deb
                    }
                }
            }
        }

        stage('Upload') {
            steps {
                gitlabCommitStatus(name: 'upload') {
                script{
                        def props = readProperties file: 'build/resources/main/META-INF/build-info.properties'
                        def name = props['build.name']
                        def version = props['build.version']
                        sh """echo build/$name-$version"""+""".deb jenkins@msas:/riocard/msa/artefatos"""
                        sh """echo jenkins@msas \'sudo apt install /riocar/artefatos/$name-$version"""+""".deb\'"""

                    }
                }
            }
        }
    }
}
// vim: syntax=groovy
