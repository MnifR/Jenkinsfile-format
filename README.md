# Learning Jenkins Pipeline

## Jenkinsfile

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
