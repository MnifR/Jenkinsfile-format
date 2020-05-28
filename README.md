# Learning Jenkins Pipeline

## Jenkinsfile Samples

### Declarative Pipeline

```groovy
pipeline {
    agent any
    stages ('Example') {
        steps {
            echo "Hello World"
        }
    }
}
```

### Scripted Pipeline

```groovy

node {
    stage('Example') {
        if (env.BRANCH_NAME == 'master') {
            echo 'I only execute on the master branch'
        } else {
            echo 'I execute elsewhere'
        }
    }
}
```

## Declarative Pipeline 

```groovy
pipeline {
    agent {}
    triggers {}
    tools {}
    environment {}
    options {}
    parameters {}
    stages {
        stage('stage1') {}
        stage('stage2') {}
        
        parallel { 
            stage('parallel_1') {}
            stage('parallel_2') {}
        }
    }
    
    // stages 
    post {
      always {}
      changed {}
      fixed {}
      regression {}
      aborted {}
      failure {}
      success {}
      unstable {}
      cleanup {}
    }
}
```


## Declarative Pipeline Sample

```groovy
pipeline {
    // ************
    // agent
    // *************
    agent any
    //agent none 
    /*
    agent {
      docker {
        image 'maven:3-alpine',
        label 'my-defined-label',
        args '-v /tmp:/tmp'
      }
    }
    */
    /*
    agent {
        dockerfile {
            filename 'Dockerfile.build'
            dir 'build'
            label 'my-defined-label'
            additionalBuildArgs  '--build-arg version=1.0.2'
    	}
    }
    */
    
    // ************
    // triggers
    // *************
    triggers {
        cron('H */4 * * 1-5')
        pollSCM('H */4 * * 1-5')
        upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS)
    }
    
    // ************
    // tools
    // *************
    tools {
      // Jenkins 'Global Tool Configuration'
      nodejs "node-latest"
      maven 'apache-maven-3.0.1' 
      gradle 'gradle-latest'
    }
    
    // ************
    // environment
    // *************
    environment {
        PROJECT = 'webapp'
        PHASE = params.PHASE 
        JOB_NAME = env.JOB_NAME // (https://wiki.jenkins.io/display/JENKINS/Building+a+software+project)
    }
    
    // ************
    // options
    // *************
    options {
        buildDiscarder(logRotator(numToKeepStr: '3', artifactNumToKeepStr: '3'))
      	checkoutToSubdirectory('foo')
      	disableConcurrentBuilds()
        newContainerPerStage
        overrideIndexTriggers(true)
        preserveStashes(5)
        retry(3)
        skipDefaultCheckout()
        skipStagesAfterUnstable()
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }
    
    // ************
    // parameters
    // *************
    parameters {
        string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: '')
        text(name: 'DEPLOY_TEXT', defaultValue: 'One\nTwo\nThree\n', description: '')
        booleanParam(name: 'DEBUG_BUILD', defaultValue: true, description: '')
        choice(name: 'CHOICES', choices: 'one\ntwo\nthree', description: '')
        file(name: 'FILE', description: 'Some file to upload')
        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'A secret password')
    }
    
    stages {
        stage('stage 1') {
            // ************
		    // when
    		// *************
            when {
                branch 'master'
                buildingTag()
                changelog '.*^\\[DEPENDENCY\\] .+$'
                changeset "**/*.js"
                changeRequest()
                environment name: 'DEPLOY_TO', value: 'production'
                equals expected: 2, actual: currentBuild.number
                expression { return params.DEBUG_BUILD }
                tag "release-*"
                tag pattern: "release-\\d+", comparator: "REGEXP"
                not { branch 'master' }
                allOf { branch 'master'; environment name: 'DEPLOY_TO', value: 'production' }
                anyOf { branch 'master'; branch 'staging' }
                anyOf {
                    environment name: 'DEPLOY_TO', value: 'production'
                    environment name: 'DEPLOY_TO', value: 'staging'
                }
        	}
            
            // ************
		    // step
    		// *************
            step {
                echo "Hello ${params.PERSON}"
                echo "Biography: ${params.BIOGRAPHY}"
                echo "Toggle: ${params.TOGGLE}"
                echo "Choice: ${params.CHOICE}"
                echo "Password: ${params.PASSWORD}"
                sh 'echo test..'
            }
        }
        
        parallel {
            stage('NPM Build') {
                steps {
                    sh """
                    	cd ${WORKSPACE}
                    	npm set progress=false
                    	yarn install
                    	yarn run stage
                    """
                }
            }
            stage('Gradle Build') {
                steps {
                    sh "gradle clean build"
                }
            }
        }
    }
    
    // ************
	// post
    // *************
    post {
        always {
            echo 'I will always say Hello again!'
        }
        success {
            archiveArtifacts artifacts: "**"
        }
    }
}
```

## Samples

Build Job 

```groovy
pipeline {
    agent {
        node {
            label 'BUILD'
        }
    }
    
    environment {
        TYPE = "build"
        PROJECT = "webapp"
        PHASE = "dev"
        GIT_URL = "git@github.com:mcpaint/learning-jenkins-pipeline.git"
        ARTIFACTS = "build/libs/**"
    }
    
    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: "30", artifactNumToKeepStr: "30"))
        timeout(time: 120, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
    }
    
    tools {
        jdk "jdk8-latest"
        nodejs "node-latest"
        gradle "gradle-latest"
    }
    
    stages {
        stage('Set Environment') {
            steps {
                script {
                    if (PHASE == 'dev') {
                        BRANCH = 'develop'
                    } else if (PHASE == 'prod') {
                        BRANCH = 'master'
                    }
                }
            }
        } 
        stage('Checkout') {
            steps {
                git url: "${GIT_URL}", branch: "${BRANCH}", poll: true, changelog: true
            }
        }
        stage('Builds') {
			failFast true
            parallel {
                stage('[Build] NPM') {
                    steps {
                        sh """
                            cd ${WORKSPACE}
                            npm set progress=false
                            yarn install
                            yarn run ${NPM_BUILD_MODE}
                        """
                    }
                }

                stage('[Build] Gradle') {
                    steps {
                        sh "gradle clean build -x test --stacktrace"
                    }
                }
            }
        }
    }
    
    post {
        success {
            archiveArtifacts artifacts: "${ARTIFACTS}"
            slackSend channel: '#ops-room',
                color: 'good',
                message: "The pipeline ${currentBuild.fullDisplayName} completed successfully."
        }
        failure {
            mail to: 'team@example.com',
             subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
             body: "Something is wrong with ${env.BUILD_URL}"
        }
    }
}
```
Build job 2

```groovy
node ('nodejs') {
  currentBuild.result = 'SUCCESS'

  stage ('Checkout') {
    // Clean workspace before checkout
    step ([$class: 'WsCleanup'])
    checkout scm
  }

  stage ('Build') {
    env.PATH = "/opt/jenkins/bin:${env.PATH}"
    catchError {
      // Install dependencies
      sh 'npm install'
      // Build assets with eg. webpack 
      sh 'npm run build'
    }
  }

  stage ('Test') {
    catchError {
      sh 'npm test'
      // Publish our test and coverage reports
      step([$class: 'JUnitResultArchiver', testResults: 'tests/test-report.xml'])
      step([
        $class: 'CloverPublisher',
        cloverReportDir: 'tests',
        cloverReportFileName: 'coverage-report.xml',
        healthyTarget: [methodCoverage: 67, conditionalCoverage: 75, statementCoverage: 67],
        unhealthyTarget: [methodCoverage: 10, conditionalCoverage: 15, statementCoverage: 10],
        failingTarget: [methodCoverage: 5, conditionalCoverage: 10, statementCoverage: 5]
      ])
    }
  }

  stage ('Deploy') {
    echo "We are currently working on branch: ${env.BRANCH_NAME}"

    switch (env.BRANCH_NAME) {
      case 'master': 
        env.DEPLOYMENT_ENVIRONMENT = 'prod';
        env.PROPERTY_FILE = 'env.prod.properties';
        break;
      case 'develop':
        env.DEPLOYMENT_ENVIRONMENT = 'test';
        env.PROPERTY_FILE = 'env.test.properties';
        break;
      default: env.DEPLOYMENT_ENVIRONMENT = 'no_deploy';
    }

    if (env.DEPLOYMENT_ENVIRONMENT != 'no_deploy') {
      catchError { sh './deploy.sh' }
    }
  }
}
```

Build job 3

```groovy

String GIT_VERSION

node {

  def buildEnv
  def devAddress

  stage ('Checkout') {
    deleteDir()
    checkout scm
    GIT_VERSION = sh (
      script: 'git describe --tags',
      returnStdout: true
    ).trim()
  }

  stage ('Build Custom Environment') {
    buildEnv = docker.build("build_env:${GIT_VERSION}", 'custom-build-env')
  }

  buildEnv.inside {

    stage ('Build') {
      sh 'sbt compile'
      sh 'sbt sampleClient/universal:stage'
    }

    stage ('Test') {
      parallel (
        'Test Server' : {
          sh 'sbt server/test'
        },
        'Test Sample Client' : {
          sh 'sbt sampleClient/test'
        }
      )
    }

    stage ('Prepare Docker Image') {
      sh 'sbt server/docker:stage'
    }
  }

  stage ('Build and Push Docker Image') {
    withCredentials([[$class: "UsernamePasswordMultiBinding", usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS', credentialsId: 'Docker Hub']]) {
      sh 'docker login --username $DOCKERHUB_USER --password $DOCKERHUB_PASS'
    }
    def serverImage = docker.build("sambott/grpc-test:${GIT_VERSION}", 'server/target/docker/stage')
    serverImage.push()
    sh 'docker logout'
  }

  stage ('Deploy to DEV') {
    devAddress = deployContainer("sambott/grpc-test:${GIT_VERSION}", 'DEV')
  }

  stage ('Verify Deployment') {
    buildEnv.inside {
      sh "sample-client/target/universal/stage/bin/demo-client ${devAddress}"
    }
  }
}

stage 'Deploy to LIVE'
  timeout(time:2, unit:'DAYS') {
    input message:'Approve deployment to LIVE?'
  }
  node {
    deployContainer("sambott/grpc-test:${GIT_VERSION}", 'LIVE')
  }

def deployContainer(image, env) {
  docker.image('lachlanevenson/k8s-kubectl:v1.5.2').inside {
    withCredentials([[$class: "FileBinding", credentialsId: 'KubeConfig', variable: 'KUBE_CONFIG']]) {
      def kubectl = "kubectl  --kubeconfig=\$KUBE_CONFIG --context=${env}"
      sh "${kubectl} set image deployment/grpc-demo grpc-demo=${image}"
      sh "${kubectl} rollout status deployment/grpc-demo"
      return sh (
        script: "${kubectl} get service/grpc-demo -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
        returnStdout: true
      ).trim()
    }
  }
}

```
Build job 4

```groovy
#!/usr/bin/env groovy
pipeline {
    agent { node { label 'swarm-ci' } }

    environment {
        TEST_PREFIX = "test-IMAGE"
        TEST_IMAGE = "${env.TEST_PREFIX}:${env.BUILD_NUMBER}"
        TEST_CONTAINER = "${env.TEST_PREFIX}-${env.BUILD_NUMBER}"
        REGISTRY_ADDRESS = "my.registry.address.com"

        SLACK_CHANNEL = "#deployment-notifications"
        SLACK_TEAM_DOMAIN = "MY-SLACK-TEAM"
        SLACK_TOKEN = credentials("slack_token")
        DEPLOY_URL = "https://deployment.example.com/"

        COMPOSE_FILE = "docker-compose.yml"
        REGISTRY_AUTH = credentials("docker-registry")
        STACK_PREFIX = "my-project-stack-name"
    }

    stages {
        stage("Prepare") {
            steps {
                bitbucketStatusNotify buildState: "INPROGRESS"
            }
        }

        stage("Build and start test image") {
            steps {
                sh "docker-composer build"
                sh "docker-compose up -d"
                waitUntilServicesReady
            }
        }

        stage("Run tests") {
            steps {
                sh "docker-compose exec -T php-fpm composer --no-ansi --no-interaction tests-ci"
                sh "docker-compose exec -T php-fpm composer --no-ansi --no-interaction behat-ci"
            }

            post {
                always {
                    junit "build/junit/*.xml"
                    step([
                            $class              : "CloverPublisher",
                            cloverReportDir     : "build/coverage",
                            cloverReportFileName: "clover.xml"
                    ])
                }
            }
        }

        stage("Determine new version") {
            when {
                branch "master"
            }

            steps {
                script {
                    env.DEPLOY_VERSION = sh(returnStdout: true, script: "docker run --rm -v '${env.WORKSPACE}':/repo:ro softonic/ci-version:0.1.0 --compatible-with package.json").trim()
                    env.DEPLOY_MAJOR_VERSION = sh(returnStdout: true, script: "echo '${env.DEPLOY_VERSION}' | awk -F'[ .]' '{print \$1}'").trim()
                    env.DEPLOY_COMMIT_HASH = sh(returnStdout: true, script: "git rev-parse HEAD | cut -c1-7").trim()
                    env.DEPLOY_BUILD_DATE = sh(returnStdout: true, script: "date -u +'%Y-%m-%dT%H:%M:%SZ'").trim()

                    env.DEPLOY_STACK_NAME = "${env.STACK_PREFIX}-v${env.DEPLOY_MAJOR_VERSION}"
                    env.IS_NEW_VERSION = sh(returnStdout: true, script: "[ '${env.DEPLOY_VERSION}' ] && echo 'YES'").trim()
                }
            }
        }

        stage("Create new version") {
            when {
                branch "master"
                environment name: "IS_NEW_VERSION", value: "YES"
            }

            steps {
                script {
                    sshagent(['ci-ssh']) {
                        sh """
                            git config user.email "ci-user@email.com"
                            git config user.name "Jenkins"
                            git tag -a "v${env.DEPLOY_VERSION}" \
                                -m "Generated by: ${env.JENKINS_URL}" \
                                -m "Job: ${env.JOB_NAME}" \
                                -m "Build: ${env.BUILD_NUMBER}" \
                                -m "Env Branch: ${env.BRANCH_NAME}"
                            git push origin "v${env.DEPLOY_VERSION}"
                        """
                    }
                }

                sh "docker login -u=$REGISTRY_AUTH_USR -p=$REGISTRY_AUTH_PSW ${env.REGISTRY_ADDRESS}"
                sh "docker-compose -f ${env.COMPOSE_FILE} build"
                sh "docker-compose -f ${env.COMPOSE_FILE} push"
            }
        }

        stage("Deploy to production") {
            agent { node { label "swarm-prod" } }

            when {
                branch "master"
                environment name: "IS_NEW_VERSION", value: "YES"
            }

            steps {
                sh "docker login -u=$REGISTRY_AUTH_USR -p=$REGISTRY_AUTH_PSW ${env.REGISTRY_ADDRESS}"
                sh "docker stack deploy ${env.DEPLOY_STACK_NAME} -c ${env.COMPOSE_FILE} --with-registry-auth"
            }

            post {
                success {
                    slackSend(
                            teamDomain: "${env.SLACK_TEAM_DOMAIN}",
                            token: "${env.SLACK_TOKEN}",
                            channel: "${env.SLACK_CHANNEL}",
                            color: "good",
                            message: "${env.STACK_PREFIX} production deploy: *${env.DEPLOY_VERSION}*. <${env.DEPLOY_URL}|Access service> - <${env.BUILD_URL}|Check build>"
                    )
                }

                failure {
                    slackSend(
                            teamDomain: "${env.SLACK_TEAM_DOMAIN}",
                            token: "${env.SLACK_TOKEN}",
                            channel: "${env.SLACK_CHANNEL}",
                            color: "danger",
                            message: "${env.STACK_PREFIX} production deploy failed: *${env.DEPLOY_VERSION}*. <${env.BUILD_URL}|Check build>"
                    )
                }
            }
        }
    }

    post {
        always {
            sh "docker-compose down || true"
        }

        success {
            bitbucketStatusNotify buildState: "SUCCESSFUL"
        }

        failure {
            bitbucketStatusNotify buildState: "FAILED"
        }
    }
}
```
Build job 5

```groovy
pipeline {

    /*
     * Run everything on an existing agent configured with a label 'docker'.
     * This agent will need docker, git and a jdk installed at a minimum.
     */
    agent {
        node {
            label 'docker'
        }
    }

    // using the Timestamper plugin we can add timestamps to the console log
    options {
        timestamps()
    }

    environment {
        //Use Pipeline Utility Steps plugin to read information from pom.xml into env variables
        IMAGE = readMavenPom().getArtifactId()
        VERSION = readMavenPom().getVersion()
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    /*
                     * Reuse the workspace on the agent defined at top-level of Pipeline but run inside a container.
                     * In this case we are running a container with maven so we don't have to install specific versions
                     * of maven directly on the agent
                     */
                    reuseNode true
                    image 'maven:3.5.0-jdk-8'
                }
            }
            steps {
                // using the Pipeline Maven plugin we can set maven configuration settings, publish test results, and annotate the Jenkins console
                withMaven(options: [findbugsPublisher(), junitPublisher(ignoreAttachments: false)]) {
                    sh 'mvn clean findbugs:findbugs package'
                }
            }
            post {
                success {
                    // we only worry about archiving the jar file if the build steps are successful
                    archiveArtifacts(artifacts: '**/target/*.jar', allowEmptyArchive: true)
                }
            }
        }

        stage('Quality Analysis') {
            parallel {
                // run Sonar Scan and Integration tests in parallel. This syntax requires Declarative Pipeline 1.2 or higher
                stage('Integration Test') {
                    agent any  //run this stage on any available agent
                    steps {
                        echo 'Run integration tests here...'
                    }
                }
                stage('Sonar Scan') {
                    agent {
                        docker {
                            // we can use the same image and workspace as we did previously
                            reuseNode true
                            image 'maven:3.5.0-jdk-8'
                        }
                    }
                    environment {
                        //use 'sonar' credentials scoped only to this stage
                        SONAR = credentials('sonar')
                    }
                    steps {
                        sh 'mvn sonar:sonar -Dsonar.login=$SONAR_PSW'
                    }
                }
            }
        }

        stage('Build and Publish Image') {
            when {
                branch 'master'  //only run these steps on the master branch
            }
            steps {
                /*
                 * Multiline strings can be used for larger scripts. It is also possible to put scripts in your shared library
                 * and load them with 'libaryResource'
                 */
                sh """
          docker build -t ${IMAGE} .
          docker tag ${IMAGE} ${IMAGE}:${VERSION}
          docker push ${IMAGE}:${VERSION}
        """
            }
        }
    }

    post {
        failure {
            // notify users when the Pipeline fails
            mail to: 'team@example.com',
                    subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
                    body: "Something is wrong with ${env.BUILD_URL}"
        }
    }
}
```
Build job 6

```groovy
pipeline {
    agent any
    environment {
        POM_VERSION = readMavenPom().getVersion()
        BUILD_RELEASE_VERSION = readMavenPom().getVersion().replace("-SNAPSHOT", "")
        IS_SNAPSHOT = readMavenPom().getVersion().endsWith("-SNAPSHOT")
        GIT_TAG_COMMIT = sh(script: 'git describe --tags --always', returnStdout: true).trim()
    }
    stages {
        stage('stage one') {
            steps {
                script {
                    tags_extra = "value_1"
                }
                echo "tags_extra: ${tags_extra}"
            }
        }
        stage('stage two') {
            steps {
                echo "tags_extra: ${tags_extra}"
            }
        }
        stage('stage three') {
            when {
                expression { tags_extra != 'bla' }
            }
            steps {
                echo "tags_extra: ${tags_extra}"
            }
        }
    }
}
```
Build job 7

```groovy

#!/usr/bin/env groovy
// Some fast steps to inspect the build server. Create a pipeline script job and add this:

node {
   DOCKER_PATH = sh (script: 'command -v docker', returnStdout: true).trim()
   echo "Docker path: ${DOCKER_PATH}"
   
   FREE_MEM = sh (script: 'free -m', returnStdout: true).trim()
   echo "Free memory: ${FREE_MEM}"
   
   echo sh(script: 'env|sort', returnStdout: true)

}
```
Build job 8

```groovy
#!/usr/bin/env groovy
pipeline {
  agent any

  stages {
    stage("Build") {
      steps {
        sh 'mvn -v'
      }
    }

    stage("Testing") {
      parallel {
        stage("Unit Tests") {
          agent { docker 'openjdk:7-jdk-alpine' }
          steps {
            sh 'java -version'
          }
        }
        stage("Functional Tests") {
          agent { docker 'openjdk:8-jdk-alpine' }
          steps {
            sh 'java -version'
          }
        }
        stage("Integration Tests") {
          steps {
            sh 'java -version'
          }
        }
      }
    }

    stage("Deploy") {
      steps {
        echo "Deploy!"
      }
    }
  }
}
```
Build job 9

```groovy
pipeline {
   agent any
    
   environment {
      VALUE_ONE = '1'
      VALUE_TWO = '2'
      VALUE_THREE = '3'
   }
    
   stages {
   
      // skip a stage while creating the pipeline
      stage("A stage to be skipped") {
         when {
            expression { false }  //skip this stage
         }
         steps {
            echo 'This step will never be run'
         }
      }
      
      // Execute when branch = 'master'
      stage("BASIC WHEN - Branch") {
         when {
            branch 'master'
	 }
         steps {
            echo 'BASIC WHEN - Master Branch!'
         }
      }
      
      // Expression based when example with AND
      stage('WHEN EXPRESSION with AND') {
         when {
            expression {
               VALUE_ONE == '1' && VALUE_THREE == '3'
            }
         }
         steps {
            echo 'WHEN with AND expression works!'
         }
      }
      
      // Expression based when example
      stage('WHEN EXPRESSION with OR') {
         when {
            expression {
               VALUE_ONE == '1' || VALUE_THREE == '2'
            }
         }
         steps {
            echo 'WHEN with OR expression works!'
         }
      }
      
      // When - AllOf Example
      stage("AllOf") {
        when {
            allOf {
                environment name:'VALUE_ONE', value: '1'
                environment name:'VALUE_TWO', value: '2'
            }
        }
        steps {
            echo "AllOf Works!!"
        }
      }
      
      // When - Not AnyOf Example
      stage("Not AnyOf") {
         when {
            not {
               anyOf {
                  branch "development"
                  environment name:'VALUE_TWO', value: '4'
               }
            }
         }
         steps {
            echo "Not AnyOf - Works!"
         }
      }
   }
}
```
Build job 10

```groovy
pipeline {
    // run on jenkins nodes tha has java 8 label
    agent { label 'java8' }
    // global env variables
    environment {
        EMAIL_RECIPIENTS = 'mahmoud.romeh@test.com'
    }
    stages {

        stage('Build with unit testing') {
            steps {
                // Run the maven build
                script {
                    // Get the Maven tool.
                    // ** NOTE: This 'M3' Maven tool must be configured
                    // **       in the global configuration.
                    echo 'Pulling...' + env.BRANCH_NAME
                    def mvnHome = tool 'Maven 3.3.9'
                    if (isUnix()) {
                        def targetVersion = getDevVersion()
                        print 'target build version...'
                        print targetVersion
                        sh "'${mvnHome}/bin/mvn' -Dintegration-tests.skip=true -Dbuild.number=${targetVersion} clean package"
                        def pom = readMavenPom file: 'pom.xml'
                        // get the current development version 
                        developmentArtifactVersion = "${pom.version}-${targetVersion}"
                        print pom.version
                        // execute the unit testing and collect the reports
                        junit '**//*target/surefire-reports/TEST-*.xml'
                        archive 'target*//*.jar'
                    } else {
                        bat(/"${mvnHome}\bin\mvn" -Dintegration-tests.skip=true clean package/)
                        def pom = readMavenPom file: 'pom.xml'
                        print pom.version
                        junit '**//*target/surefire-reports/TEST-*.xml'
                        archive 'target*//*.jar'
                    }
                }

            }
        }
        stage('Integration tests') {
            // Run integration test
            steps {
                script {
                    def mvnHome = tool 'Maven 3.3.9'
                    if (isUnix()) {
                        // just to trigger the integration test without unit testing
                        sh "'${mvnHome}/bin/mvn'  verify -Dunit-tests.skip=true"
                    } else {
                        bat(/"${mvnHome}\bin\mvn" verify -Dunit-tests.skip=true/)
                    }

                }
                // cucumber reports collection
                cucumber buildStatus: null, fileIncludePattern: '**/cucumber.json', jsonReportDirectory: 'target', sortingMethod: 'ALPHABETICAL'
            }
        }
        stage('Sonar scan execution') {
            // Run the sonar scan
            steps {
                script {
                    def mvnHome = tool 'Maven 3.3.9'
                    withSonarQubeEnv {

                        sh "'${mvnHome}/bin/mvn'  verify sonar:sonar -Dintegration-tests.skip=true -Dmaven.test.failure.ignore=true"
                    }
                }
            }
        }
        // waiting for sonar results based into the configured web hook in Sonar server which push the status back to jenkins
        stage('Sonar scan result check') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    retry(3) {
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "Pipeline aborted due to quality gate failure: ${qg.status}"
                            }
                        }
                    }
                }
            }
        }
        stage('Development deploy approval and deployment') {
            steps {
                script {
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                        timeout(time: 3, unit: 'MINUTES') {
                            // you can use the commented line if u have specific user group who CAN ONLY approve
                            //input message:'Approve deployment?', submitter: 'it-ops'
                            input message: 'Approve deployment?'
                        }
                        timeout(time: 2, unit: 'MINUTES') {
                            //
                            if (developmentArtifactVersion != null && !developmentArtifactVersion.isEmpty()) {
                                // replace it with your application name or make it easily loaded from pom.xml
                                def jarName = "application-${developmentArtifactVersion}.jar"
                                echo "the application is deploying ${jarName}"
                                // NOTE : CREATE your deployemnt JOB, where it can take parameters whoch is the jar name to fetch from jenkins workspace
                                build job: 'ApplicationToDev', parameters: [[$class: 'StringParameterValue', name: 'jarName', value: jarName]]
                                echo 'the application is deployed !'
                            } else {
                                error 'the application is not  deployed as development version is null!'
                            }

                        }
                    }
                }
            }
        }
        stage('DEV sanity check') {
            steps {
                // give some time till the deployment is done, so we wait 45 seconds
                sleep(45)
                script {
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                        timeout(time: 1, unit: 'MINUTES') {
                            script {
                                def mvnHome = tool 'Maven 3.3.9'
                                //NOTE : if u change the sanity test class name , change it here as well
                                sh "'${mvnHome}/bin/mvn' -Dtest=ApplicationSanityCheck_ITT surefire:test"
                            }

                        }
                    }
                }
            }
        }
        stage('Release and publish artifact') {
            when {
                // check if branch is master
                branch 'master'
            }
            steps {
                // create the release version then create a tage with it , then push to nexus releases the released jar
                script {
                    def mvnHome = tool 'Maven 3.3.9' //
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                        def v = getReleaseVersion()
                        releasedVersion = v;
                        if (v) {
                            echo "Building version ${v} - so released version is ${releasedVersion}"
                        }
                        // jenkins user credentials ID which is transparent to the user and password change
                        sshagent(['0000000-3b5a-454e-a8e6-c6b6114d36000']) {
                            sh "git tag -f v${v}"
                            sh "git push -f --tags"
                        }
                        sh "'${mvnHome}/bin/mvn' -Dmaven.test.skip=true  versions:set  -DgenerateBackupPoms=false -DnewVersion=${v}"
                        sh "'${mvnHome}/bin/mvn' -Dmaven.test.skip=true clean deploy"

                    } else {
                        error "Release is not possible. as build is not successful"
                    }
                }
            }
        }
        stage('Deploy to Acceptance') {
            when {
                // check if branch is master
                branch 'master'
            }
            steps {
                script {
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                        timeout(time: 3, unit: 'MINUTES') {
                            //input message:'Approve deployment?', submitter: 'it-ops'
                            input message: 'Approve deployment to UAT?'
                        }
                        timeout(time: 3, unit: 'MINUTES') {
                            //  deployment job which will take the relasesed version
                            if (releasedVersion != null && !releasedVersion.isEmpty()) {
                                // make the applciation name for the jar configurable
                                def jarName = "application-${releasedVersion}.jar"
                                echo "the application is deploying ${jarName}"
                                // NOTE : DO NOT FORGET to create your UAT deployment jar , check Job AlertManagerToUAT in Jenkins for reference
                                // the deployemnt should be based into Nexus repo
                                build job: 'AApplicationToACC', parameters: [[$class: 'StringParameterValue', name: 'jarName', value: jarName], [$class: 'StringParameterValue', name: 'appVersion', value: releasedVersion]]
                                echo 'the application is deployed !'
                            } else {
                                error 'the application is not  deployed as released version is null!'
                            }

                        }
                    }
                }
            }
        }
        stage('ACC E2E tests') {
            when {
                // check if branch is master
                branch 'master'
            }
            steps {
                // give some time till the deployment is done, so we wait 45 seconds
                sleep(45)
                script {
                    if (currentBuild.result == null || currentBuild.result == 'SUCCESS') {
                        timeout(time: 1, unit: 'MINUTES') {

                            script {
                                def mvnHome = tool 'Maven 3.3.9'
                                // NOTE : if you change the test class name change it here as well
                                sh "'${mvnHome}/bin/mvn' -Dtest=ApplicationE2E surefire:test"
                            }

                        }
                    }
                }
            }
        }
    }
    post {
        // Always runs. And it runs before any of the other post conditions.
        always {
            // Let's wipe out the workspace before we finish!
            deleteDir()
        }
        success {
            sendEmail("Successful");
        }
        unstable {
            sendEmail("Unstable");
        }
        failure {
            sendEmail("Failed");
        }
    }

// The options directive is for configuration that applies to the whole job.
    options {
        // For example, we'd like to make sure we only keep 10 builds at a time, so
        // we don't fill up our storage!
        buildDiscarder(logRotator(numToKeepStr: '5'))

        // And we'd really like to be sure that this build doesn't hang forever, so
        // let's time it out after an hour.
        timeout(time: 25, unit: 'MINUTES')
    }

}
def developmentArtifactVersion = ''
def releasedVersion = ''
// get change log to be send over the mail
@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "Gathering SCM changes"
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += " - ${truncated_msg} [${entry.author}]\n"
        }
    }

    if (!changeString) {
        changeString = " - No new changes"
    }
    return changeString
}

def sendEmail(status) {
    mail(
            to: "$EMAIL_RECIPIENTS",
            subject: "Build $BUILD_NUMBER - " + status + " (${currentBuild.fullDisplayName})",
            body: "Changes:\n " + getChangeString() + "\n\n Check console output at: $BUILD_URL/console" + "\n")
}

def getDevVersion() {
    def gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    def versionNumber;
    if (gitCommit == null) {
        versionNumber = env.BUILD_NUMBER;
    } else {
        versionNumber = gitCommit.take(8);
    }
    print 'build  versions...'
    print versionNumber
    return versionNumber
}

def getReleaseVersion() {
    def pom = readMavenPom file: 'pom.xml'
    def gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    def versionNumber;
    if (gitCommit == null) {
        versionNumber = env.BUILD_NUMBER;
    } else {
        versionNumber = gitCommit.take(8);
    }
    return pom.version.replace("-SNAPSHOT", ".${versionNumber}")
}

// if you want parallel execution , check below :
//stage('Quality Gate(Integration Tests and Sonar Scan)') {
//    // Run the maven build
//    steps {
//        parallel(
//                IntegrationTest: {
//                    script {
//                        def mvnHome = tool 'Maven 3.3.9'
//                        if (isUnix()) {
//                            sh "'${mvnHome}/bin/mvn'  verify -Dunit-tests.skip=true"
//                        } else {
//                            bat(/"${mvnHome}\bin\mvn" verify -Dunit-tests.skip=true/)
//                        }
//
//                    }
//                },
//                SonarCheck: {
//                    script {
//                        def mvnHome = tool 'Maven 3.3.9'
//                        withSonarQubeEnv {
//                            // sh "'${mvnHome}/bin/mvn'  verify sonar:sonar -Dsonar.host.url=http://bicsjava.bc/sonar/ -Dmaven.test.failure.ignore=true"
//                            sh "'${mvnHome}/bin/mvn'  verify sonar:sonar -Dmaven.test.failure.ignore=true"
//                        }
//                    }
//                },
//                failFast: true)
//    }
//}
```
Build job 11

```groovy
#!/bin/env groovy

/*
  Parameters:
    GIT_MAVEN_REPOSITORY - Maven registry to be used when pushing artifacts
    GIT_MAVEN_CREDENTIALS_ID - Credentials when using GIT_MAVEN_REPOSITORY
    DOCKER_PUSH_REGISTRY - Docker registry to be used when pushing image
    DOCKER_PUSH_REGISTRY_CREDENTIALS_ID - Credentials when using DOCKER_PUSH_REGISTRY
 */



pipeline {

    // parameters {
    //   string(name: 'PARAM_1', description: 'parameter #1', defaultValue: 'default_a')
    //   string(name: 'PARAM_2', description: 'parameter #2')
    //   string(name: 'PARAM_ONLY_IN_JENKINS_BRANCH', description: 'this param is only present in jenkins branch ')
    //   string(name: 'PARAM_EXTRA', description: 'en extra param ')
    //   // string(name: 'GIT_MAVEN_REPOSITORY', description: 'Maven registry to be used when pushing artifacts')
    //   // string(name: 'GIT_MAVEN_CREDENTIALS_ID', description: 'Credentials when using GIT_MAVEN_REPOSITORY')
    //   // string(name: 'DOCKER_PUSH_REGISTRY', description: 'Docker registry to be used when pushing image')
    //   // string(name: 'DOCKER_PUSH_REGISTRY_CREDENTIALS_ID', description: 'Credentials when using DOCKER_PUSH_REGISTRY')
    // }

  agent any

  environment {
    POM_GROUP_ID = readMavenPom().getGroupId()
    POM_VERSION = readMavenPom().getVersion()
    BUILD_RELEASE_VERSION = readMavenPom().getVersion().replace("-SNAPSHOT", "")
    GIT_TAG_COMMIT = sh (script: 'git describe --tags --always', returnStdout: true).trim()
    //IS_SNAPSHOT = readMavenPom().getVersion().endsWith("-SNAPSHOT")
    IS_SNAPSHOT = getMavenVersion().endsWith("-SNAPSHOT")
    version_a = getMavenVersion()
  }

  stages {
    stage('Init') {
      steps {
        echo "version_a: ${env.versiona_a}"
        script {println type: IS_SNAPSHOT.getClass()}
        echo sh(script: 'env|sort', returnStdout: true)
        echo "Variable params: ${params}"
        //echo "params.PARAM_1: ${params.PARAM_1}"

      }
    }
    stage('Build') {
      when {
        expression { false }  //disable this stage
      }
      steps {
        sh './mvnw -B clean package -DskipTests'
        archive includes: '**/target/*.jar'
        stash includes: '**/target/*.jar', name: 'jar'
      }
    }

  }
}



def getMavenVersion() {
  return sh(script: "./mvnw org.apache.maven.plugins:maven-help-plugin:2.1.1:evaluate -Dexpression=project.version | grep -v '\\['  | tail -1", returnStdout: true).trim()
}
```
Build job 12

```groovy
def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(label: label, containers: [
  containerTemplate(name: 'gradle', image: 'gradle:4.5.1-jdk9', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:latest', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:latest', command: 'cat', ttyEnabled: true)
],
volumes: [
  hostPathVolume(mountPath: '/home/gradle/.gradle', hostPath: '/tmp/jenkins/.gradle'),
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH
    def shortGitCommit = "${gitCommit[0..10]}"
    def previousGitCommit = sh(script: "git rev-parse ${gitCommit}~", returnStdout: true)
 
    stage('Test') {
      try {
        container('gradle') {
          sh """
            pwd
            echo "GIT_BRANCH=${gitBranch}" >> /etc/environment
            echo "GIT_COMMIT=${gitCommit}" >> /etc/environment
            gradle test
            """
        }
      }
      catch (exc) {
        println "Failed to test - ${currentBuild.fullDisplayName}"
        throw(exc)
      }
    }
    stage('Build') {
      container('gradle') {
        sh "gradle build"
      }
    }
    stage('Create Docker images') {
      container('docker') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
          credentialsId: 'dockerhub',
          usernameVariable: 'DOCKER_HUB_USER',
          passwordVariable: 'DOCKER_HUB_PASSWORD']]) {
          sh """
            docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}
            docker build -t namespace/my-image:${gitCommit} .
            docker push namespace/my-image:${gitCommit}
            """
        }
      }
    }
    stage('Run kubectl') {
      container('kubectl') {
        sh "kubectl get pods"
      }
    }
    stage('Run helm') {
      container('helm') {
        sh "helm list"
      }
    }
  }
}
```
Build job 13

```groovy
import java.text.SimpleDateFormat

def argocdAppPrefix = "hello-world-ks"
def deployable_branches = ["master"]
def ptNameVersion = "${argocdAppPrefix}-${UUID.randomUUID().toString().toLowerCase()}"
def imageName = "argoprojdemo/argo-cd-hello-world-app"
def deployRepoUrl = "git@github.com:argoproj/argo-cd-hello-world-config.git"
def argocdServer = "argo-cd-demo.argoproj.io"
def appWaitTimeout = 600

podTemplate(name: ptNameVersion, label: ptNameVersion, containers: [
    containerTemplate(name: 'builder', image: 'golang:1.10.3', ttyEnabled: true, command: 'cat', args: ''),
    containerTemplate(name: 'docker', image: 'docker:17.09', ttyEnabled: true, command: 'cat', args: '' ),
    containerTemplate(name: 'argo-cd-tools', image: 'argoproj/argo-cd-tools:latest', ttyEnabled: true, command: 'cat', args: '', envVars:[envVar(key: 'GIT_SSH_COMMAND', value: 'ssh -o StrictHostKeyChecking=no')] ),
    containerTemplate(name: 'argo-cd-cli', image: 'argoproj/argocd-cli:v0.7.1', ttyEnabled: true, command: 'cat', args: '', envVars:[envVar(key: 'ARGOCD_SERVER', value: argocdServer)] ),
    ],
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
  )

{
    // DO NOT CHANGE
    def isPR = env.CHANGE_ID != null
    def branch = env.CHANGE_ID != null ? env.CHANGE_TARGET : env.BRANCH_NAME
    def dateFormat = new SimpleDateFormat("yyyyMMddHHmm")
    def date = new Date()
    def date_tag = dateFormat.format(date)

    // exit gracefully if not the master branch (or, rather, not in deployable_branches)
    if (!deployable_branches.contains(branch)) {
        stage("Skipping pipeline") {
            println "Branch: ${branch} is not part of deployable_branches"
            println "Skipping pipeline"
        }
        currentBuild.result = 'SUCCESS'
        return
    }

    node(ptNameVersion) {
        // DO NOT CHANGE
        def scmInfo = checkout scm
        def gitCommit = "${scmInfo.GIT_COMMIT}"
        tag = "${env.BUILD_TAG}-${gitCommit}"

        // Build Stage
        stage('Build') {
            withCredentials([usernamePassword(credentialsId: "docker-credentials", passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                container('builder') {
                    sh "curl -O https://get.docker.com/builds/Linux/x86_64/docker-1.13.1.tgz && tar -xzf docker-1.13.1.tgz"
                    sh "mv docker/docker /usr/local/bin/docker && chmod 755 /usr/local/bin/docker"
                    sh "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
                    sh "make publish"
                }
            }
        }
        def env = "preprod"
        stage( "Deploy ${env}" ) {
            container('argo-cd-tools') {
                println("Deploying to ${argocdAppPrefix}")
                dir("deployment-${env}-${tag}") {
                    withCredentials([file(credentialsId: 'githubDeployKey', variable: 'GIT_DEPLOY_KEY')]) {
                        sh "mkdir /root/.ssh/ && cp \$GIT_DEPLOY_KEY /root/.ssh/id_rsa && chmod 400 /root/.ssh/id_rsa"
                        sh "git clone ${deployRepoUrl}"
                        sh "cd \$(basename '${deployRepoUrl}' .git) && ./update-image.sh ${env} ${argocdAppPrefix} ${imageName} ${gitCommit}"
                    }
                }          
            }
            container('argo-cd-cli') {
                withCredentials([string(credentialsId: "argocdAuthToken", variable: 'ARGOCD_AUTH_TOKEN')]) {
                    sh "/argocd app sync ${argocdAppPrefix}-${env}"
                    sh "/argocd app wait ${argocdAppPrefix}-${env} --timeout ${appWaitTimeout}"
                }
            }
        }
    }
}
```
Build job 14

```groovy
pipeline {
  agent {
    kubernetes {
      label 'jenkins-slave'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:18.09-dind
    securityContext:
      privileged: true
  - name: docker
    env:
    - name: DOCKER_HOST
      value: 127.0.0.1
    image: docker:18.09
    command:
    - cat
    tty: true
  - name: tools
    image: argoproj/argo-cd-ci-builder:v0.13.1
    command:
    - cat
    tty: true
"""
    }
  }
  stages {

    stage('Build') {
      environment {
        DOCKERHUB_CREDS = credentials('dockerhub')
      }
      steps {
        container('docker') {
          // Build new image
          sh "until docker ps; do sleep 3; done && docker build -t alexmt/argocd-demo:${env.GIT_COMMIT} ."
          // Publish new image
          sh "docker login --username $DOCKERHUB_CREDS_USR --password $DOCKERHUB_CREDS_PSW && docker push alexmt/argocd-demo:${env.GIT_COMMIT}"
        }
      }
    }

    stage('Deploy E2E') {
      environment {
        GIT_CREDS = credentials('git')
      }
      steps {
        container('tools') {
          sh "git clone https://$GIT_CREDS_USR:$GIT_CREDS_PSW@github.com/alexmt/argocd-demo-deploy.git"
          sh "git config --global user.email 'ci@ci.com'"

          dir("argocd-demo-deploy") {
            sh "cd ./e2e && kustomize edit set image alexmt/argocd-demo:${env.GIT_COMMIT}"
            sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
          }
        }
      }
    }

    stage('Deploy to Prod') {
      steps {
        input message:'Approve deployment?'
        container('tools') {
          dir("argocd-demo-deploy") {
            sh "cd ./prod && kustomize edit set image alexmt/argocd-demo:${env.GIT_COMMIT}"
            sh "git commit -am 'Publish new version' && git push || echo 'no changes'"
          }
        }
      }
    }
  }
}
```
