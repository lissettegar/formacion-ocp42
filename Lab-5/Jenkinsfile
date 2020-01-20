#!groovy
podTemplate(
  label: "skopeo-pod",
  cloud: "openshift",
  inheritFrom: "maven",
  containers: [
    containerTemplate(
      name: "jnlp",
      image: "image-registry.openshift-image-registry.svc:5000/${GUID}-jenkins/jenkins-agent-appdev",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi",
      resourceRequestCpu: "1",
      resourceLimitCpu: "2"
    )
  ]
) {
  node('skopeo-pod') {
    // Define Maven Command to point to the correct
    // settings for our Nexus installation
    def mvnCmd = "mvn -s ./nexus_openshift_settings.xml"

    // Set variable globally to be available in all stages
    // Set Development and Production Project Names
    def devProject  = "${GUID}-tasks-dev"
    def prodProject = "${GUID}-tasks-prod"


    // Checkout Source Code
    stage('Checkout Source') {
      checkout scm
    }

    def version = getVersionFromPom("pom.xml")
    // Set the tag for the development image: version + build number
    devTag  = "${version}-" + currentBuild.number
    // Set the tag for the production image: version
    prodTag = "${version}"


    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build App') {
      echo "Building version ${devTag}"
      sh "${mvnCmd} clean package -DskipTests=true"
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      echo "Running Unit Tests"
      sh "${mvnCmd} test"

      // This next step is optional.
      // It displays the results of tests in the Jenkins Task Overview
      step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
    }

    stage('Code Analysis') {
      echo "Running Code Analysis"
      sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-${GUID}-sonarqube.apps.ocpdevmad01.tic1.intranet -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
    }


    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      echo "Publish to Nexus"
      sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus3.${GUID}-nexus.svc.cluster.local:8081/repository/releases"
    }


    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      echo "Building OpenShift container image tasks:${devTag}"

      // Start Binary Build in OpenShift using the file we just published
      // The filename is openshift-tasks.war in the 'target' directory of your current
      // Jenkins workspace
      script {
        openshift.withCluster() {
          openshift.withProject("${devProject}") {
            openshift.selector("bc", "tasks").startBuild("--from-file=./target/openshift-tasks.war", "--wait=true")
            // openshift.selector("bc", "tasks").startBuild("--from-file=http://nexus3-${GUID}-nexus.apps.ocpdevmad01.tic1.intranet/repository/releases/org/jboss/quickstarts/eap/openshift-tasks/${version}/openshift-tasks-${version}.war", "--wait=true")
            openshift.tag("tasks:latest", "tasks:${devTag}")
          }
        }
      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      echo "Deploying container image to Development Project"
      script {
        // Update the Image on the Development Deployment Config
        openshift.withCluster() {
          openshift.withProject("${devProject}") {
            // OpenShift 4
            openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${devTag}")

            // Update the Config Map which contains the users for the Tasks application
            // (just in case the properties files changed in the latest commit)
            openshift.selector('configmap', 'tasks-config').delete()
            def configmap = openshift.create('configmap', 'tasks-config', '--from-file=./configuration/application-users.properties', '--from-file=./configuration/application-roles.properties' )

            // Deploy the development application.
            openshift.selector("dc", "tasks").rollout().latest();

            // Wait for application to be deployed
            def dc = openshift.selector("dc", "tasks").object()
            def dc_version = dc.status.latestVersion
            def rc = openshift.selector("rc", "tasks-${dc_version}").object()

            echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
            while (rc.spec.replicas != rc.status.readyReplicas) {
              sleep 5
              rc = openshift.selector("rc", "tasks-${dc_version}").object()
            }
          }
        }
      }
    }

    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      echo "Running Integration Tests"
      script {
        def status = "000"

        // Create a new task called "integration_test_1"
        echo "Creating task"
        status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${GUID}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
        echo "Status: " + status
        if (status != "201") {
            error 'Integration Create Test Failed!'
        }

        echo "Retrieving tasks"
        status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Accept: application/json' -X GET http://tasks.${GUID}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
        if (status != "200") {
            error 'Integration Get Test Failed!'
        }

        echo "Deleting tasks"
        status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -X DELETE http://tasks.${GUID}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
        if (status != "204") {
            error 'Integration Create Test Failed!'
        }
      }
    }


    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      echo "Copy image to Nexus Docker Registry"
      script {
        // OpenShift 4
        sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin docker://image-registry.openshift-image-registry.svc.cluster.local:5000/${devProject}/tasks:${devTag} docker://nexus-registry.${GUID}-nexus.svc.cluster.local:5000/tasks:${devTag}"
      }
    }

    // Promote to Production
    stage('PROMOTE-to-PRO - Stage') {
      input message: "Promote new code?", ok: "Promote"
      }

    stage('Promote to Production') {
      echo "Promote to Production"
      script {
        openshift.withCluster() {
          openshift.withProject("${prodProject}") {
            openshift.tag("${devProject}/tasks:${devTag}", "${prodProject}/tasks:${prodTag}")
            openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${prodProject}/tasks:${prodTag}")

            // Update Config Map in change config files changed in the source
            openshift.selector("configmap", "tasks-config").delete()
            def configmap = openshift.create("configmap", "tasks-config", "--from-file=./configuration/application-users.properties", "--from-file=./configuration/application-roles.properties" )

            // Deploy the production application.
            openshift.selector("dc", "tasks").rollout().latest();

            // Wait for application to be deployed
            def dc = openshift.selector("dc", "tasks").object()
            def dc_version = dc.status.latestVersion
            def rc = openshift.selector("rc", "tasks-${dc_version}").object()

            echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
            while (rc.spec.replicas != rc.status.readyReplicas) {
              sleep 5
              rc = openshift.selector("rc", "tasks-${dc_version}").object()
            }
          }
        }
      }
    }
  }
 }
// Convenience Functions to read version from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
