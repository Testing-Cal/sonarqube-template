def getFolderName() {
  def array = pwd().split("/")
  return array[array.length - 2];
}

def parseJsonString(jsonString, key){
  def datas = readJSON text: jsonString
  String Values = writeJSON returnText: true, json: datas[key]
  return Values
}

def parseJsonArray(jsonString){
  def datas = readJSON text: jsonString
  return datas
}

def pushToCollector(){
  print("Inside pushToCollector...........")
    def job_name = "$env.JOB_NAME"
    def job_base_name = "$env.JOB_BASE_NAME"
    String generalProperties = parseJsonString(env.JENKINS_METADATA,'general')
    generalPresent = parseJsonArray(generalProperties)
    if(generalPresent.tenant != '' &&
    generalPresent.lazsaDomainUri != ''){
      echo "Job folder - $job_name"
      echo "Pipeline Name - $job_base_name"
      echo "Build Number - $currentBuild.number"
      sh """curl -k -X POST '${generalPresent.lazsaDomainUri}/collector/orchestrator/devops/details' -H 'X-TenantID: ${generalPresent.tenant}' -H 'Content-Type: application/json' -d '{\"jobName\" : \"${job_base_name}\", \"projectPath\" : \"${job_name}\", \"agentId\" : \"${generalPresent.agentId}\", \"devopsConfigId\" : \"${generalPresent.devopsSettingId}\", \"agentApiKey\" : \"${generalPresent.agentApiKey}\", \"buildNumber\" : \"${currentBuild.number}\" }' """
    }
}

def agentLabel = "${env.JENKINS_AGENT == null ? "":env.JENKINS_AGENT}"
pipeline {
  agent any
  environment {
    BRANCHES = "${env.GIT_BRANCH}"
    COMMIT = "${env.GIT_COMMIT}"
    RELEASE_NAME = "sonarqube"
    SERVICE_PORT = "${APP_PORT}"
    DOCKERHOST = "${DOCKERHOST_IP}"
    ACTION = "${ACTION}"
    DEPLOYMENT_TYPE = "${DEPLOYMENT_TYPE == ""? "EC2":DEPLOYMENT_TYPE}"
    KUBE_SECRET = "${KUBE_SECRET}"
    foldername = getFolderName()

  }
  stages {
    stage('Initialization') {
      agent { label agentLabel }
      steps {
        script {
          TEMP_STAGE_NAME = "$STAGE_NAME"
          def job_name = "$env.JOB_NAME"
          print(job_name)
          def values = job_name.split('/')
          namespace_prefix = values[0].replaceAll("[^a-zA-Z0-9\\-\\_]+","").toLowerCase().take(50)
          namespace = "$namespace_prefix-$env.foldername".toLowerCase()
          service = values[2].replaceAll("[^a-zA-Z0-9\\-\\_]+","").toLowerCase().take(50)
          print("kube namespace: $namespace")
          print("service name: $service")
          env.namespace_name=namespace
          env.service=service
          
        }
      }
    }

    stage('Deploy') {
      agent { label agentLabel }
      steps {
        script {
          echo "echoed folder--- $foldername"
          TEMP_STAGE_NAME = "$STAGE_NAME"
          if (env.DEPLOYMENT_TYPE == 'EC2') {

            sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker volume create --name sonarqube_data"'
            sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker volume create --name sonarqube_logs"'
            sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker volume create --name sonarqube_extensions"'
            sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker rm -f  sonarqube || true"'
            sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker run -d --restart always --name sonarqube -p 9001:9000  -v sonarqube_conf:/opt/sonarqube/conf   -v sonarqube_extensions:/opt/sonarqube/extensions   -v sonarqube_logs:/opt/sonarqube/logs  -v sonarqube_data:/opt/sonarqube/data sonarqube:lts"'

          }
          if (env.DEPLOYMENT_TYPE == 'KUBERNETES') {
                          withCredentials([file(credentialsId: "$KUBE_SECRET", variable: 'KUBECONFIG')]) {
                  sh '''

                    rm -rf kube
                    mkdir -p kube
                    cp "$KUBECONFIG" kube
                    kubectl create ns "$namespace_name" || true
                    kubectl apply -f ./charts -n "$namespace_name"
                    sleep 30
                    kubectl get svc -n "$namespace_name"
                  '''
                  script {
                    env.temp_service_name = "$RELEASE_NAME".take(63)
                    def url = sh (returnStdout: true, script: '''kubectl get svc -n "$namespace_name" | grep "$temp_service_name" | awk '{print $4}' ''').trim()
                    if (url != "<pending>") {
                      print("##\$@\$ http://$url ##\$@\$")
                    }
                  }
            }
          }
          
        }
      }
    }

    stage('Destroy') {
      agent { label agentLabel }
      when {
        expression {
          env.DEPLOYMENT_TYPE == 'EC2' && env.ACTION == 'DESTROY'
        }
      }
      steps {
        script {
          TEMP_STAGE_NAME = "$STAGE_NAME"
          if (env.DEPLOYMENT_TYPE == 'EC2') {
            sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker stop sonarqube"'
          }
          if (env.DEPLOYMENT_TYPE == 'KUBERNETES') {
              withCredentials([file(credentialsId: "$KUBE_SECRET", variable: 'KUBECONFIG')]) {
                  sh '''
                    kubectl delete -f ./charts -n "$namespace_name"
                  '''
            }
          }
          
        }
      }
    }
  }
}
