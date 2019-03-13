def templateName = "nucleus"
def templatePath = "https://raw.githubusercontent.com/adri8n/openshift-templates/master/templates/nucleus.json"
def appName = "nucleus-web"
def devProjectName = "dev"
def stageProjectName = "stage"

pipeline {
    agent none
     options {
        // set a timeout of 20 minutes for this pipeline
        timeout(time: 20, unit: 'MINUTES')
    }
    stages {
        stage('Cleanup') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(devProjectName) {
                            // change this to a groovy for loop using dict/hash:
                            openshift.selector("all", [ template : templateName]).delete()
                            if (openshift.selector("secrets", appName).exists()) {
                                openshift.selector("secrets", appName).delete()
                            }
                            if (openshift.selector("secrets", appName).exists()) {
                                openshift.selector("secrets", appName).delete()
                                openshift.selector("secrets", appName).watch {
                                    echo "Waiting for secret ${appName} deletion..."
                                    sleep(5)
                                    return it.count() == 0
                                }
                            }
                            if (openshift.selector("pvc", "nucleus-web-staticfiles-claim").exists()) {
                                openshift.selector("pvc", "nucleus-web-staticfiles-claim").delete()
                                openshift.selector("pvc", "nucleus-web-staticfiles-claim").watch {
                                    echo "Waiting for PVC nucleus-web-staticfiles-claim deletion..."
                                    sleep(5)
                                    return it.count() == 0
                                }
                            }
                            if (openshift.selector("pvc", "postgresql").exists()) {
                                openshift.selector("pvc", "postgresql").delete()
                                openshift.selector("pvc", "postgresql").watch {
                                    echo "Waiting for PVC postgresql deletion..."
                                    sleep(5)
                                    return it.count() == 0
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('initiate template') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject(devProjectName) {
                            // create a new application from the templatePath
                            openshift.newApp(templatePath)
                        }
                    }
                } // script
            }
        }
        stage('build') {
            steps {
                script {
                    openshift.verbose()
                    openshift.withCluster() {
                        openshift.withProject(devProjectName) {
                            def builds = openshift.selector("bc", 'nucleus-web').related('builds')
                            builds.untilEach(1) {
                                echo "${it.object().name}: status ${it.object().status.phase}"
                                return (it.object().status.phase == "Complete")
                            }
                        }
                    }
                } // script
            } // steps
        } // stage
        stage("deploy") {
            steps {
                script {
                    openshift.verbose()
                    openshift.withCluster() {
                        openshift.withProject(devProjectName) {
                            echo "monitor dc deployment"
                            openshift.selector("dc", appName).related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                }
            }
        }
        stage('Promote to Stage?') {
            steps {
                timeout(time:15, unit:'MINUTES') {
                    input message: "Promote to STAGE?", ok: "Promote"
                }
                script {
                    openshift.withCluster() {
                        openshift.tag("${devProjectName}/${templateName}-web-nginx:latest", "${stageProjectName}/${templateName}-web-nginx:latest")
                        openshift.tag("${devProjectName}/${templateName}-web:latest", "${stageProjectName}/${templateName}-web:latest")
                    }
                }
            }
        }
        stage('Rollout to Stage') {
            steps {
                script {
                    def nucleus
                    def objs
                    def timestamp
                    openshift.verbose()
                    openshift.withCluster() {
                        openshift.withProject(devProjectName) {
                            nucleus = openshift.selector('dc', [ app : 'nucleus' ])
                            objs = nucleus.objects (exportable:true)
                            secrets = openshift.selector('secrets', [ app : 'nucleus' ]).objects(exportable:true)
                            services = openshift.selector('services', [ app : 'nucleus' ]).objects(exportable:true)
                            timestamp = "${System.currentTimeMillis()}"
                            for ( obj in objs ) {
                                obj.metadata.labels[ "promoted-on" ] = timestamp
                            }
                        }
                        openshift.withProject(stageProjectName) {
                            nucleus.delete('--ignore-not-found')
                            try {
                                openshift.selector('secrets', 'nucleus-web').delete()
                            } catch(Exception ex) {
                                echo "error, probably didn't exist ${ex}"
                            }
                            openshift.create(secrets)
                            try {
                                openshift.create(services)
                            } catch(Exception ex) {
                                echo "error, probably already exists ${ex}"
                            }
                            try {
                                openshift.create('https://raw.githubusercontent.com/adri8n/openshift-templates/master/templates/nucleus-pvcs.json')
                            } catch(Exception ex) {
                                echo "error, probably already exists ${ex}"
                            }
                            openshift.create(objs)
                            try {
                                openshift.selector("service", "nucleus-web-nginx").expose()
                            } catch(Exception ex) {
                                echo "Expose already exists: ${ex}"
                            }
                            openshift.selector("dc", appName).related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                } 
            }
        }
    }
}