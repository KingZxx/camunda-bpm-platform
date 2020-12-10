import groovy.json.JsonOutput

// https://github.com/camunda/jenkins-global-shared-library
// https://github.com/camunda/cambpm-jenkins-shared-library
@Library(['camunda-ci', 'cambpm-jenkins-shared-library@pipeline-extract-artifact-logic']) _

String getAgent(String dockerImage = 'gcr.io/ci-30-162810/centos:v0.4.6', Integer cpuLimit = 4){
  String mavenForkCount = cpuLimit;
  String mavenMemoryLimit = cpuLimit * 2;
  """
metadata:
  labels:
    agent: ci-cambpm-camunda-cloud-build
spec:
  nodeSelector:
    cloud.google.com/gke-nodepool: agents-n1-standard-32-netssd-preempt
  tolerations:
  - key: "agents-n1-standard-32-netssd-preempt"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: "jnlp"
    image: "${dockerImage}"
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    tty: true
    env:
    - name: LIMITS_CPU
      value: ${mavenForkCount}
    - name: TZ
      value: Europe/Berlin
    resources:
      limits:
        cpu: ${cpuLimit}
        memory: ${mavenMemoryLimit}Gi
      requests:
        cpu: ${cpuLimit}
        memory: ${mavenMemoryLimit}Gi
    workingDir: "/home/work"
    volumeMounts:
      - mountPath: /home/work
        name: workspace-volume
  """
}

def failedStageTypes = []

pipeline {
  agent none
  options {
    buildDiscarder(logRotator(numToKeepStr: '5')) //, artifactNumToKeepStr: '30'
    copyArtifactPermission('*');
  }
  parameters {
      string defaultValue: 'cambpm-ee-main-pr/pipeline-master', description: 'The name of the EE branch to run the EE pipeline on, e.g. cambpm-ee/cambpm-ee-main/PR-333', name: 'EE_BRANCH_NAME'
  }
  stages {
    stage('ASSEMBLY') {
      when {
        expression {
          env.BRANCH_NAME == cambpmDefaultBranch() || !pullRequest.labels.contains('no-build-TODO') //TODO
        }
        beforeAgent true
      }
      agent {
        kubernetes {
          yaml getAgent('gcr.io/ci-30-162810/centos:v0.4.6', 16)
        }
      }
      steps {
        withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
          nodejs('nodejs-14.6.0'){
             configFileProvider([configFile(fileId: 'maven-nexus-settings', variable: 'MAVEN_SETTINGS_XML')]) {
               sh """
                 mvn -s \$MAVEN_SETTINGS_XML clean install source:jar -Pdistro,distro-ce,distro-wildfly,distro-webjar -DskipTests -Dmaven.repo.local=\${WORKSPACE}/.m2 com.mycila:license-maven-plugin:check -B
               """
             }
          }

          // archive all .jar, .pom, .xml, .txt runtime artifacts + required .war/.zip/.tar.gz for EE pipeline
          // add a new line for each group of artifacts
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/*.jar,.m2/org/camunda/**/*-SNAPSHOT/**/*.pom,.m2/org/camunda/**/*-SNAPSHOT/**/*.xml,.m2/org/camunda/**/*-SNAPSHOT/**/*.txt')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-webapp*frontend-sources.zip')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/license-book*.zip')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-jboss-modules*.zip')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-*-assembly*.tar.gz')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-webapp*.war')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-engine-rest*.war')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-example-invoice*.war')
          cambpmArchiveArtifacts('.m2/org/camunda/**/*-SNAPSHOT/**/camunda-h2-webapp*.war')

          cambpmStash("platform-stash-runtime",
                      ".m2/org/camunda/**/*-SNAPSHOT/**",
                      "**/qa/**,**/*qa*/**,**/*.zip,**/*.tar.gz")
          cambpmStash("platform-stash-archives",
                      ".m2/org/camunda/bpm/**/*-SNAPSHOT/**/*.zip,.m2/org/camunda/bpm/**/*-SNAPSHOT/**/*.tar.gz")
          cambpmStash("platform-stash-qa",
                      ".m2/org/camunda/bpm/**/qa/**/*-SNAPSHOT/**,.m2/org/camunda/bpm/**/*qa*/**/*-SNAPSHOT/**",
                      "**/*.zip,**/*.tar.gz")

        }
        

        script {
          String labels = '';
          if (env.BRANCH_NAME == cambpmDefaultBranch()) {
            params.EE_BRANCH_NAME = 'cambpm-ee-main/pipeline-master'
          } else {
            labels = JsonOutput.toJson(pullRequest.labels)
            shaCE = sh(script:"git rev-parse HEAD", returnStdout: true).trim()
          }

          if (cambpmWithLabels('webapp-integration','all-as','h2','websphere','weblogic','jbosseap','run','spring-boot','authorizations')) {
            cambpmTriggerDownstream("cambpm-ee/${params.EE_BRANCH_NAME}", labels, shaCE, true, true)
          }

          if (cambpmWithLabels('all-db','cockroachdb','authorizations')) {
            cambpmTriggerDownstream("cambpm-ce/cambpm-sidetrack/${env.BRANCH_NAME}", labels, true, false)
          }

          if (cambpmWithLabels('default-build','rolling-update','migration','all-db','h2','db2','mysql','oracle','mariadb','sqlserver','postgresql','cockroachdb','daily')) {
            cambpmTriggerDownstream("cambpm-ce/cambpm-daily/${env.BRANCH_NAME}", labels, true, false)
          }

          if (cambpmWithLabels('master')) {
            withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
              configFileProvider([configFile(fileId: 'maven-nexus-settings', variable: 'MAVEN_SETTINGS_XML')]) {
               sh 'mvn -s \$MAVEN_SETTINGS_XML org.sonatype.plugins:nexus-staging-maven-plugin:deploy-staged -DaltStagingDirectory=${WORKSPACE}/staging -Dmaven.repo.local=${WORKSPACE}/.m2 -DskipStaging=true -B'
              }
            }
          }
        }

      }
    }
    stage('h2 tests') {
      parallel {
        stage('engine-UNIT-h2') {
          when {
            expression {
              cambpmWithLabels('h2', 'rolling-update', 'migration')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/centos:v0.4.6', 16)
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, getMavenProfileDir('engine-unit'), getMavenProfileCmd('engine-unit') + cambpmGetDbProfiles('h2'))
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
            failure {
              cambpmAddFailedStageType(failedStageTypes, 'engine-unit')
            }
          }
        }
        stage('engine-UNIT-authorizations-h2') {
          when {
            expression {
              cambpmWithLabels('h2','authorizations')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/centos:v0.4.6', 16)
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, getMavenProfileDir('engine-unit-authorizations'), getMavenProfileCmd('engine-unit-authorizations') + cambpmGetDbProfiles('h2'))
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
            failure {
              cambpmAddFailedStageType(failedStageTypes, 'engine-unit-authorizations')
            }
          }
        }
        stage('engine-rest-UNIT-jersey-2') {
          when {
            expression {
              cambpmWithLabels('rest-api')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, 'engine-rest/engine-rest/', 'clean install -Pjersey2')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('engine-rest-UNIT-resteasy3') {
          when {
            expression {
              cambpmWithLabels('rest-api')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, 'engine-rest/engine-rest/', 'clean install -Presteasy3')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('webapp-UNIT-h2') {
          when {
            expression {
              cambpmWithLabels('default-build')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, getMavenProfileDir('webapps-unit'), getMavenProfileCmd('webapps-unit') + cambpmGetDbProfiles('h2'))
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
            failure {
              cambpmAddFailedStageType(failedStageTypes, 'webapps-unit')
            }
          }
        }
        stage('webapp-UNIT-authorizations-h2') {
          when {
            expression {
              cambpmWithLabels('default-build')
            }
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, getMavenProfileDir('webapps-unit-authorizations'), getMavenProfileCmd('webapps-unit-authorizations') + cambpmGetDbProfiles('h2'))
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
            failure {
              cambpmAddFailedStageType(failedStageTypes, 'webapps-unit-authorizations')
            }
          }
        }
        stage('engine-IT-tomcat-9-postgresql-96') {
          when {
            expression {
              cambpmWithLabels('all-as','tomcat')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getDbAgent('postgresql_96')
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, false, 'qa/', 'clean install -Ptomcat,postgresql,engine-integration')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('engine-IT-wildfly-postgresql-96') {
          when {
            expression {
              cambpmWithLabels('all-as','wildfly')
            }
          }
          agent {
            kubernetes {
              yaml getDbAgent('postgresql_96')
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, false, 'qa/', 'clean install -Pwildfly,postgresql,engine-integration')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
            failure {
              cambpmAddFailedStageType(failedStageTypes, 'engine-IT-wildfly')
            }
          }
        }
        stage('webapp-IT-tomcat-9-h2') {
          when {
            expression {
              cambpmWithLabels('webapp-integration', 'h2')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/chrome:78v0.1.2')
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, false, 'qa/', 'clean install -Ptomcat,h2,webapps-integration')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('webapp-IT-standalone-wildfly') {
          when {
            branch cambpmDefaultBranch();
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/chrome:78v0.1.2')
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, false, 'qa/', 'clean install -Pwildfly-vanilla,webapps-integration-sa')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('camunda-run-IT') {
          when {
            expression {
              cambpmWithLabels('run')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/chrome:78v0.1.2', 16)
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, true, 'distro/run/', 'clean install -Pintegration-test-camunda-run')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('spring-boot-starter-IT') {
          when {
            expression {
              cambpmWithLabels('spring-boot')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/chrome:78v0.1.2', 16)
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, true, 'spring-boot-starter/', 'clean install -Pintegration-test-spring-boot-starter')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
      }
    }
    stage('UNIT DB tests') {
      matrix {
        axes {
          axis {
            name 'DB'
            values 'postgresql_96', 'mariadb_103'
          }
          axis {
            name 'PROFILE'
            values 'engine-unit', 'engine-unit-authorizations', 'webapps-unit', 'webapps-unit-authorizations'
          }
        }
        when {
          expression {
            cambpmIsNotFailedStageType(failedStageTypes, env.PROFILE) && (cambpmWithLabelsList(getLabels(env.PROFILE)) || cambpmWithDbLabels(env.DB))
          }
          beforeAgent true
        }
        agent {
          kubernetes {
            yaml getDbAgent(env.DB)
          }
        }
        stages {
          stage('UNIT test') {
            steps {
              echo("UNIT DB Test Stage: ${env.PROFILE}-${env.DB}")
              catchError(stageResult: 'FAILURE') {
                withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                  runMaven(true, false, false, getMavenProfileDir(env.PROFILE), getMavenProfileCmd(env.PROFILE) + cambpmGetDbProfiles(env.DB) + " " + cambpmGetDbExtras(env.DB), true)
                }
              }
            }
            post {
              always {
                cambpmPublishTestResult();
              }
            }
          }
        }
      }
    }
    stage('db tests + CE webapps IT') {
      parallel {
        stage('engine-api-compatibility') {
          when {
            allOf {
              expression {
                cambpmIsNotFailedStageType(failedStageTypes, 'engine-unit')
              }
              branch cambpmDefaultBranch();
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/centos:v0.4.6', 16)
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, 'engine/', 'clean verify -Pcheck-api-compatibility')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('engine-UNIT-plugins') {
          when {
            allOf {
              expression {
                cambpmIsNotFailedStageType(failedStageTypes, 'engine-unit')
              }
              branch cambpmDefaultBranch();
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent('gcr.io/ci-30-162810/centos:v0.4.6', 16)
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, 'engine/', 'clean test -Pcheck-plugins')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('engine-UNIT-database-table-prefix') {
          when {
            expression {
              cambpmIsNotFailedStageType(failedStageTypes, 'engine-unit') && cambpmWithLabels('all-db','h2','db2','mysql','oracle','mariadb','sqlserver','postgresql','cockroachdb') // TODO store as param
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, 'engine/', 'clean test -Pdb-table-prefix')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('webapp-UNIT-database-table-prefix') {
          when {
            allOf {
              expression {
                cambpmIsNotFailedStageType(failedStageTypes, 'webapps-unit')
              }
              branch cambpmDefaultBranch();
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                nodejs('nodejs-14.6.0') {
                  runMaven(true, false, false, 'webapps/', 'clean test -Pdb-table-prefix')
                }
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('engine-UNIT-wls-compatibility') {
          when {
            allOf {
              expression {
                cambpmIsNotFailedStageType(failedStageTypes, 'engine-unit')
              }
              branch cambpmDefaultBranch();
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, false, false, '.', 'clean verify -Pcheck-engine,wls-compatibility,jersey')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('IT-wildfly-domain') {
          when {
            expression {
              cambpmIsNotFailedStageType(failedStageTypes, 'engine-IT-wildfly') && cambpmWithLabels('wildfly')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, false, 'qa/', 'clean install -Pwildfly-domain,h2,engine-integration')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
        stage('IT-wildfly-servlet') {
          when {
            expression {
              cambpmIsNotFailedStageType(failedStageTypes, 'engine-IT-wildfly') && cambpmWithLabels('wildfly')
            }
            beforeAgent true
          }
          agent {
            kubernetes {
              yaml getAgent()
            }
          }
          steps {
            catchError(stageResult: 'FAILURE') {
              withMaven(jdk: 'jdk-8-latest', maven: 'maven-3.2-latest', mavenSettingsConfig: 'camunda-maven-settings', options: [artifactsPublisher(disabled: true), junitPublisher(disabled: true)]) {
                runMaven(true, true, true, 'qa/', 'clean install -Pwildfly,wildfly-servlet,h2,engine-integration')
              }
            }
          }
          post {
            always {
              cambpmPublishTestResult();
            }
          }
        }
      }
    }
  }
  post {
    changed {
      script {
        if (!agentDisconnected()){ 
          // send email if the slave disconnected
        }
      }
    }
    always {
      script {
        if (agentDisconnected()) {// Retrigger the build if the slave disconnected
          //currentBuild.result = 'ABORTED'
          //currentBuild.description = "Aborted due to connection error"
          build job: currentBuild.projectName, propagate: false, quietPeriod: 60, wait: false
        }
      }
    }
  }
}

void runMaven(boolean runtimeStash, boolean archivesStash, boolean qaStash, String directory, String cmd, boolean singleThreaded = false) {
  if (runtimeStash) cambpmUnstash("platform-stash-runtime")
  if (archivesStash) cambpmUnstash("platform-stash-archives")
  if (qaStash) cambpmUnstash("platform-stash-qa")
  String forkCount = singleThreaded? "-DforkCount=1" : '';
  configFileProvider([configFile(fileId: 'maven-nexus-settings', variable: 'MAVEN_SETTINGS_XML')]) {
    sh("mvn -s \$MAVEN_SETTINGS_XML ${forkCount} ${cmd} -nsu -Dmaven.repo.local=\${WORKSPACE}/.m2  -f ${directory}/pom.xml -B")
  }
}

String getDbAgent(String dbLabel, Integer cpuLimit = 4, Integer mavenForkCount = 1){
  Map dbInfo = cambpmGetDbInfo(dbLabel)
  String mavenMemoryLimit = cpuLimit * 4;
  """
metadata:
  labels:
    name: "${dbLabel}"
    jenkins: "slave"
    jenkins/label: "jenkins-slave-${dbInfo.type}"
spec:
  containers:
  - name: "jnlp"
    image: "gcr.io/ci-30-162810/${dbInfo.type}:${dbInfo.version}"
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
    tty: true
    env:
    - name: LIMITS_CPU
      value: ${mavenForkCount}
    - name: TZ
      value: Europe/Berlin
    resources:
      limits:
        memory: ${mavenMemoryLimit}Gi
      requests:
        cpu: ${cpuLimit}
        memory: ${mavenMemoryLimit}Gi
    volumeMounts:
    - mountPath: "/home/work"
      name: "workspace-volume"
    workingDir: "/home/work"
    nodeSelector:
      cloud.google.com/gke-nodepool: "agents-n1-standard-4-netssd-preempt"
    restartPolicy: "Never"
    tolerations:
    - effect: "NoSchedule"
      key: "agents-n1-standard-4-netssd-preempt"
      operator: "Exists"
    volumes:
    - emptyDir:
        medium: ""
      name: "workspace-volume"
  """
}

String resolveMavenProfileInfo(String profile) {
  Map PROFILE_PATHS = [
      'engine-unit': [
          directory: 'engine/',
          command: 'clean test -P',
          labels: ['authorizations']],
      'engine-unit-authorizations': [
          directory: 'engine/',
          command: 'clean test -PcfgAuthorizationCheckRevokesAlways,',
          labels: ['authorizations']],
      'webapps-unit': [
          directory: 'webapps/',
          command: 'clean test -Dskip.frontend.build=true -P',
          labels: ['default-build']],
      'webapps-unit-authorizations': [
          directory: 'webapps/',
          command: 'clean test -Dskip.frontend.build=true -PcfgAuthorizationCheckRevokesAlways,',
          labels: ['default-build']]
  ]

  return PROFILE_PATHS[profile]
}

String getMavenProfileCmd(String profile) {
  return resolveMavenProfileInfo(profile).command
}

String getMavenProfileDir(String profile) {
  return resolveMavenProfileInfo(profile).directory
}

String[] getLabels(String profile) {
  return resolveMavenProfileInfo(profile).labels
}