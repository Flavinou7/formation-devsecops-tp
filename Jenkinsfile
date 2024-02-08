pipeline {
  agent any

  stages {
     


    stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they  be d
            }
        }


    stage('UNIT test & jacoco ') {
      steps {
        sh "mvn test"
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
          jacoco execPattern: 'target/jacoco.exec'
        }
      }
 
    } 


      stage('Mutation Tests - PIT') {
  	        steps {
    	          sh "mvn org.pitest:pitest-maven:mutationCoverage"
  	          }
    	post {
     	  always {
       	  pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      	}
    	}
	    }




  
    stage('Vulnerability Scan - Docker Trivy') {
       steps {
//--------------------------replace variable  token_github on file trivy-image-scan.sh
         withCredentials([string(credentialsId: 'trivy_fr', variable: 'TOKEN')]) {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
        sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh"      
        sh "sudo bash trivy-image-scan.sh"
        }
        }
       }
     }



    stage('Vulnerability Scan - Docker') {
        steps {
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
              sh "mvn dependency-check:check"
         }
      }
          post {
            always {
              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
         }
     }
 }


    stage('SonarQube Analysis - SAST') {
        steps {
catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
 
          withSonarQubeEnv('SonarQubeConfig') {
              sh "mvn sonar:sonar \
            -Dsonar.projectKey=sonarqube_flavinou \
            -Dsonar.projectName=sonarqube_flavinou \
            -Dsonar.host.url=http://mytpm.eastus.cloudapp.azure.com:9999"
              }
            }
        }
    }



      stage('Docker Build and Push') {
  	steps {
    	withCredentials([string(credentialsId: 'docker_hub_password_fr', variable: 'DOCKER_HUB_PASSWORD')]) {
      	sh 'sudo docker login -u flavinou7 -p $DOCKER_HUB_PASSWORD'
      	sh 'printenv'
      	sh 'sudo docker build -t flavinou7/devops-app:""$GIT_COMMIT"" .'
      	sh 'sudo docker push flavinou7/devops-app:""$GIT_COMMIT""'
    	}

  	}
	}


      stage('Deployment Kubernetes  ') {
  	steps {
    	withKubeConfig([credentialsId: 'kubeconfig']) {
           	sh "sed -i 's#replace#flavinou7/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
           	sh "kubectl apply -f k8s_deployment_service.yaml"
         	}
  	  }

	  }


    stage('Vulnerability Scan - Kubernetes') {
   	steps {
     	parallel(
       	"OPA Scan": {
         	sh 'sudo docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
       	},
       	"Kubesec Scan": {
         	sh "sudo bash kubesec-scan.sh"
       	},
       	"Trivy Scan": {
         	sh "sudo bash trivy-k8s-scan.sh"
       	}
     	)
   	}
 	}




    }
}