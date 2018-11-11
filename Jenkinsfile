#!groovy

// Run this pipeline on the custom Maven Slave ('maven-appdev')
// Maven Slaves have JDK and Maven already installed
// 'maven-appdev' has skopeo installed as well.
node('maven-appdev') {
  // Define Maven Command. Make sure it points to the correct
  // settings for our Nexus installation (use the service to
  // bypass the router). The file nexus_openshift_settings.xml
  // needs to be in the Source Code repository.
  def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

  // Checkout Source Code
  stage('Checkout Source') {
    git credentialsId: '3a55338d-550a-4b0b-8a2f-c7348ee48d3f', url: 'http://gogs-vino-gogs.apps.6209.openshift.opentlc.com/CICDLabs/openshift-tasks.git'
  }
  // The following variables need to be defined at the top level
  // and not inside the scope of a stage - otherwise they would not
  // be accessible from other stages.
  // Extract version and other properties from the pom.xml
  def groupId    = getGroupIdFromPom("pom.xml")
  def artifactId = getArtifactIdFromPom("pom.xml")
  def version    = getVersionFromPom("pom.xml")

  // Set the tag for the development image: version + build number
  def devTag  = "0.0-0"
  // Set the tag for the production image: version
  def prodTag = "0.0"

  // Using Maven build the war file
  // Do not run tests in this step
  stage('Build war') {
    echo "Building version ${version}"
    sh "${mvnCmd} clean package -DskipTests"
  }

  // Using Maven run the unit tests
  stage('Unit Tests') {
    echo "Running Unit Tests"
    sh "${mvnCmd} test"
  }

  // Using Maven call SonarQube for Code Analysis
  stage('Code Analysis') {
    echo "Running Code Analysis"
    sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-vino-sonarqube.apps.6209.openshift.opentlc.com/ -Dsonar.projectName=${JOB_BASE_NAME}-${devTag}"
    
  }

  // Publish the built war file to Nexus
  stage('Publish to Nexus') {
    echo "Publish to Nexus"
    sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.vino-nexus.svc.cluster.local:8081/repository/releases" 
  }

  // Build the OpenShift Image in OpenShift and tag it.
  stage('Build and Tag OpenShift Image') {
    echo "Building OpenShift container image tasks:${devTag}"
    sh "oc start-build tasks --follow --from-file=http://nexus3.vino-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${version}/tasks-${version}.war -n vino-tasks-dev"
    openshiftTag alias: 'false', destStream: 'tasks', destTag: devTag, destinationNamespace: 'vino-tasks-dev', namespace: 'vino-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
    
  }

  // Deploy the built image to the Development Environment.
  stage('Deploy to Dev') {
    echo "Deploying container image to Development Project"
    sh "oc set image dc/tasks tasks=docker-registry.default.svc:5000/vino-tasks-dev/tasks:${devTag} -n vino-tasks-dev"
    // Update the Config Map which contains the users for the Tasks application
    sh "oc delete configmap tasks-config -n vino-tasks-dev --ignore-not-found=true"
    sh "oc create configmap tasks-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n vino-tasks-dev"

  // Deploy the development application.
  // Replace xyz-tasks-dev with the name of your production project
  openshiftDeploy depCfg: 'tasks', namespace: 'vino-tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
  openshiftVerifyDeployment depCfg: 'tasks', namespace: 'vino-tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
  openshiftVerifyService namespace: 'vino-tasks-dev', svcName: 'tasks', verbose: 'false'

  }

  // Run Integration Tests in the Development Environment.
  stage('Integration Tests') {
    echo "Running Integration Tests"
    sleep 15

  // Create a new task called "integration_test_1"
  echo "Creating task"
  sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.vino-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1"

  // Retrieve task with id "1"
  echo "Retrieving tasks"
  sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X GET http://tasks.vino-tasks-dev.svc.cluster.local:8080/ws/tasks/1"

  // Delete task with id "1"
  echo "Deleting tasks"
  sh "curl -i -u 'tasks:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.vino-tasks-dev.svc.cluster.local:8080/ws/tasks/1"
  }

  // Copy Image to Nexus Docker Registry
  stage('Copy Image to Nexus Docker Registry') {
      echo "Copy image to Nexus Docker Registry"
      sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry.default.svc.cluster.local:5000/vino-tasks-dev/tasks:${devTag} docker://nexus-registry.vino-nexus.svc.cluster.local:5000/tasks:${devTag}"

  // Tag the built image with the production tag.
  // Replace xyz-tasks-dev with the name of your dev project
  openshiftTag alias: 'false', destStream: 'tasks', destTag: prodTag, destinationNamespace: 'vino-tasks-dev', namespace: 'vino-tasks-dev', srcStream: 'tasks', srcTag: devTag, verbose: 'false'
  }

  // Blue/Green Deployment into Production
  // -------------------------------------
  // Do not activate the new version yet.
  def destApp   = "tasks-green"
  def activeApp = ""

  stage('Blue/Green Production Deployment') {
    // Replace vino-tasks-dev and vino-tasks-prod with
  // your project names
  activeApp = sh(returnStdout: true, script: "oc get route tasks -n vino-tasks-prod -o jsonpath='{ .spec.to.name }'").trim()
  if (activeApp == "tasks-green") {
    destApp = "tasks-blue"
  }
  echo "Active Application:      " + activeApp
  echo "Destination Application: " + destApp

  // Update the Image on the Production Deployment Config
  sh "oc set image dc/${destApp} ${destApp}=docker-registry.default.svc:5000/vino-tasks-dev/tasks:${prodTag} -n vino-tasks-prod"

  // Update the Config Map which contains the users for the Tasks application
  sh "oc delete configmap ${destApp}-config -n vino-tasks-prod --ignore-not-found=true"
  sh "oc create configmap ${destApp}-config --from-file=./configuration/application-users.properties --from-file=./configuration/application-roles.properties -n vino-tasks-prod"

  // Deploy the inactive application.
  // Replace xyz-tasks-prod with the name of your production project
  openshiftDeploy depCfg: destApp, namespace: 'vino-tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
  openshiftVerifyDeployment depCfg: destApp, namespace: 'vino-tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
  openshiftVerifyService namespace: 'vino-tasks-prod', svcName: destApp, verbose: 'false'
  }

  stage('Switch over to new Version') {
    input "Switch Production?"
    echo "Switching Production application to ${destApp}."
    // Replace xyz-tasks-prod with the name of your production project
  sh 'oc patch route tasks -n vino-tasks-prod -p \'{"spec":{"to":{"name":"' + destApp + '"}}}\''
  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}
