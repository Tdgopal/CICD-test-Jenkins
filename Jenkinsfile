pipeline {
    agent any	
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"	    
        NEXUS_URL = "172.31.47.53:8081"
        NEXUS_REPOSITORY = "test"
	NEXUS_REPO_ID    = "test"
        NEXUS_CREDENTIAL_ID = "nexus_login"
        ARTVERSION = "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}"
	//NEXUS_ARTIFACT = "${env.NEXUS_PROTOCOL}://${env.NEXUS_URL}/repository/${env.NEXUS_REPOSITORY}/com/team/project/tmart/${env.ARTVERSION}/tmart-${env.ARTVERSION}.war"
	scannerHome = tool 'sonar'
	ecr_repo = '339712852090.dkr.ecr.us-east-2.amazonaws.com/ecrrepo'
        ecrCreds = 'awscreds'
	image = ''
    }
	
    stages{
	stage('Maven Build'){
            steps {
                sh 'mvn clean install -DskipTests'
            }}
	    
        stage('JUnit Test') {
          steps {
            script {
              sh 'mvn test'
             // junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
            }}
	}
	stage ('Checkstyle Analysis'){
            steps {
		script{
                sh 'mvn checkstyle:checkstyle'
		// recordIssues enabledForFailure: false, tool: checkStyle()
	    }}
	}
        stage('SonarQube Scan') {
	  when {
		  not{
                   expression {
                       return params.SonarQube  }}}
          steps {
	    script{
              withSonarQubeEnv('sonarscanner') {
	       echo "Stage: SonarQube Scan"
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=test \
                   -Dsonar.projectName=test \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
		    }
		echo "Quality Gate"   
		timeout(time: 5, unit: 'MINUTES') {
                       def qualitygate = waitForQualityGate(webhookSecretId: 'sqhook')
                       if (qualitygate.status != "OK") {
			   catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                                sh "exit 1"  }}}
	  }}}

        stage("Publish Artifact to Nexus") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION";
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: ARTVERSION,
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: artifactPath,
                                type: pom.packaging],
                                [artifactId: pom.artifactId,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } 
		    else {
                        error "*** File: ${artifactPath}, could not be found";
                    }}}}
	    
	stage('Docker Image Build') {
          steps {
             script {
                image = docker.build(ecr_repo + ":$BUILD_ID", "./") 
	  }}}
	    
        stage('Push Image to ECR'){
           steps {
              script {
                 docker.withRegistry("https://" + ecr_repo, "ecr:us-east-2:" + ecrCreds) {
                   image.push("$BUILD_ID")
                   image.push('latest') }
	      }}
       post {
        always {
            sh 'docker builder prune --all -f' } 
       }}	    
	    
        stage('Deploy to EKS'){
		 
            steps {
		script{
		 
                 sh '''
		 kubectl apply -f ./k8s/eksdeploy.yml
		 kubectl get deployments  
                 sleep 10
                 kubectl get svc
                 '''   }}
         post {
          always { cleanWs() }
        } 
	} 
}
	post {
          always { cleanWs() }
        }
}
