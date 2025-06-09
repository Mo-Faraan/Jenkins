- Jenkins is an open-source automation server used to automate parts of software development, including building, testing, and deploying.
- Pipeline-as-Code is written using Declarative or Scripted Pipelines (you’ve been using Declarative).
- Jenkinsfile is the pipeline script stored inside the project repo.
- Stages group steps logically.

- Steps are single tasks like sh 'mvn clean install' or checkout scm.
- Declarative Pipeline:
- The most common, readable way to define CI/CD as code using the pipeline { ... } syntax.
- Stages and Steps:
Pipelines are divided into stages (logical build steps) and steps (individual commands or actions).

- Agent:
The environment where pipeline steps run. Can be a physical/virtual machine or a Docker container.
- Top-level agent:
Applies to all stages unless overridden.
- Stage-level agent:
Allows running specific stages on different agents or Docker images.
- Label-based agent:
Use agent { label 'docker' } to target a node with a specific label.
- Docker agent:
Use agent { docker { image 'maven:3.9.6-eclipse-temurin-8' } } for containerized builds.

- {when using docker agent at the top level, each stage corresponds to individual components}
- {when using different agents for different stages, Each agent or agent block may use a different workspace directory (e.g., /workspace/job@2).}
- {can set MAVEN_REPO = "${WORKSPACE}/.m2/repository" variable and then run mvn install etc with -Dmaven.repo.local=$MAVEN_REPO argument}
- {set up volumes for using dependencies inter pipeline}

example basic :
```
pipeline {
    agent { label 'your-agent' } // or docker { image '...' }
    environment { ... }          // optional env vars
    options { ... }              // build options like timeout
    stages {
        stage('...') {
            steps {
                ...
            }
        }
    }
    post {
        always { ... }
        success { ... }
        failure { ... }
    }
}
```


- Jenkins - docker agents :
- agent { docker { image 'maven:3.9.6-eclipse-temurin-17' } } spins a fresh container for the build.
- This container:
-Has a clean environment.
-Is ephemeral (destroyed after build).
-Shares workspace with the host via volume mount.
-Key Point: Docker volumes are ephemeral unless explicitly persisted (e.g., using volumes: in Docker agent block).


1. Maven_Pipeline : Set up a pipeline to pull, build and deploy to gitlab registry and set up email notis
```
pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
        }
    }
    stages {
        
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://git.hatio.co/mohammed.farhan/mavenproject.git'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Run') {
            steps {
                sh 'java -jar target/*.jar'
            }
        }
        stage('Deploy to GitLab Maven Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'gitlab-maven-token', usernameVariable: 'GITLAB_USER', passwordVariable: 'GITLAB_TOKEN')]) {
                    writeFile file: 'settings.xml', text: """
        <settings>
          <servers>
            <server>
              <id>gitlab-maven</id>
              <username>${GITLAB_USER}</username>
              <password>${GITLAB_TOKEN}</password>
            </server>
          </servers>
        </settings>
        """
                    sh 'mvn deploy -DskipTests=true --settings settings.xml'
                }
            
            }
        }
    }
    post {
        success {
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Good news! The build was successful.\nCheck details at: ${env.BUILD_URL}",
                to: 'mdfarhantm29@gmail.com',
                from: 'farhantest29@gmail.com'
            )
        }
        failure {
            emailext (
                subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Build failed. Check details at: ${env.BUILD_URL}",
                to: 'mdfarhantm29@gmail.com',
                from: 'farhantest29@gmail.com'
            )
        }
    }
}
```
should install email extension plugin, should also set up credentials

2. maven_pipeline 2:
Pipeline which does sparse checkout of specific sub directory.
```
pipeline {
    agent any
    stages {
        stage('Build') {
            agent {
                docker {
                    image 'maven:3.9.6-eclipse-temurin-17'
                }
            }
            steps {
                deleteDir()
                checkout scmGit(
                    branches: [[name: 'dev2']],
                    extensions: [
                        sparseCheckout(sparseCheckoutPaths: [[path: 'Maven4Jenkins/New folder/mavenproject']])
                    ],
                    userRemoteConfigs: [[
                        url: 'https://git.hatio.co/mohammed.farhan/mavenproject.git',
                        credentialsId: 'mavenProjectToken']]
                )
                dir('Maven4Jenkins/New folder/mavenproject') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Run') {
            steps {
                dir('Maven4Jenkins/New folder/mavenproject') {
                    sh 'java -jar target/*.jar'
                }
            }
        }
    }
   
}
```

3. depProject :
Pipeline to build a jar of 1 project and use it as dependency in another project
```
pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
        }
    }
    stages {
        stage('Build') {
            steps {
                deleteDir()
                checkout scmGit(
                    branches: [[name: 'main']],
                    extensions: [
                        sparseCheckout(sparseCheckoutPaths: [[path: 'demo']])
                    ],
                    userRemoteConfigs: [[
                        url: 'https://git.hatio.co/mohammed.farhan/mavenprojectdep.git'
                    ]]
                )
                dir('demo') {
                    sh 'mvn clean install'
                    sh 'cp pom.xml target/demo-pom.xml'
                }
            }
        }
    }
    post {
        success{
            archiveArtifacts artifacts: 'demo/target/*.jar, demo/target/*.xml', fingerprint: true
        }
    }
   
}
```
here we archive both jar and pom and build and install it in the 2nd pipeline.

depProject 2 :
```
pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
        }
    }
    environment {
        MAVEN_REPO = "${WORKSPACE}/.m2/repository"
    }

    stages {
        stage('Checout & copy') {
            steps {
                deleteDir()
                checkout scmGit(
                    branches: [[name: 'main']],
                    extensions: [
                        sparseCheckout(sparseCheckoutPaths: [[path: 'demo2']])
                    ],
                    userRemoteConfigs: [[
                        url: 'https://git.hatio.co/mohammed.farhan/mavenprojectdep.git'
                    ]]
                )
                copyArtifacts(
                    projectName: 'DepProject',
                    selector: lastSuccessful(),
                    filter: 'demo/target/*.jar',
                    flatten: true,                  // avoids dir hierarchy.
                    target: 'demo2/libs'
                )
                copyArtifacts(
                    projectName: 'DepProject',
                    selector: lastSuccessful(),
                    filter: 'demo/target/*.xml',
                    flatten: true,
                    target: 'demo2/libs'
                )
                dir('demo2/libs'){
                    sh '''
                            mvn install:install-file \
                            -Dfile=demo-1.0-SNAPSHOT.jar \
                            -DpomFile=demo-pom.xml \
                            -Dmaven.repo.local=$MAVEN_REPO
                        '''
                    
                }
            }
        }
        stage("sdbhc"){
            steps{
                dir('demo2'){
                    sh 'mvn clean package -Dmaven.repo.local=$MAVEN_REPO'
                }
            }
        }  
    }
}
```
here we set up MAVEN_REPO = "${WORKSPACE}/.m2/repository" common maven repo for all builds.

4. sample pipeline :
```
pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-8'
        }
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://kjdbcs.git',
                credentialsId: 
            }
        }
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

    }
    post{
        success{
            archiveArtifacts artifacts: 'demo/target/*.jar', fingerprint: true
        }
    }
}
```

