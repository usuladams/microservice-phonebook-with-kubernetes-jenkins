pipeline {
  agent any
  environment {
        APP_NAME="phonebook"
        APP_REPO_NAME="${APP_NAME}-app"
        PROJECT_ID = 'k8s-demo-464210'
        CLUSTER_NAME = 'baby-cluster'
        LOCATION = 'europe-west3'
        CREDENTIALS_ID = 'gcp-k8s-token'
        ARTIFACT_REGISTRY="${LOCATION}-docker.pkg.dev/${PROJECT_ID}/${APP_REPO_NAME}"
        NAMESPACE = 'default'
        CREDENTIALS_ID2 = 'gcp-k8s'
  }

  stages {
    stage('Create GCP Artifact Repo') {
      steps {
        echo "Creating GCP Artifact Repo for ${APP_NAME} app"
        
        // Service Account kimliğiyle gcloud'a login ol
        withCredentials([file(credentialsId: "${CREDENTIALS_ID}", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
          sh '''
            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
            gcloud config set project ${PROJECT_ID}


            gcloud artifacts repositories describe ${APP_REPO_NAME} --location=${LOCATION} > /dev/null 2>&1 \
            && echo "Artifact Repo '${APP_REPO_NAME}' already exists." \
            || (echo "Creating Artifact Repo '${APP_REPO_NAME}'..." && \
            gcloud artifacts repositories create ${APP_REPO_NAME} \
              --repository-format=docker \
              --location="${LOCATION}" \
              --description="GCP Repo for phonebook app" \
              --async \
              --disable-vulnerability-scanning)
         '''
        }
      }
    }
    stage('Build App Docker Images') {
      steps {
        echo 'Building App Dev Images'
        sh "docker build -t ${ARTIFACT_REGISTRY}/${APP_REPO_NAME}:web-b${BUILD_NUMBER} ${WORKSPACE}/images/image_for_web_server"
        sh "docker build -t ${ARTIFACT_REGISTRY}/${APP_REPO_NAME}:result-b${BUILD_NUMBER} ${WORKSPACE}/images/image_for_result_server"
        sh 'docker image ls'
       }
    }
    stage('Push Images to GCP Artifact Repo') {
        steps {
          echo "Pushing ${APP_NAME} App Images to GCP Artifact Repo"
          sh "gcloud auth configure-docker europe-west3-docker.pkg.dev --quiet"
          sh "docker push ${ARTIFACT_REGISTRY}/${APP_REPO_NAME}:web-b${BUILD_NUMBER}"
          sh "docker push ${ARTIFACT_REGISTRY}/${APP_REPO_NAME}:result-b${BUILD_NUMBER}"
        }
      }
    stage('Install Ingress Controller') {
        steps {
            echo "Install Ingress Controller in Kubernetes Cluster on GKE"
            withCredentials([file(credentialsId: "${CREDENTIALS_ID}", variable: 'KUBECONFIG_FILE')]) {
            sh '''
                gcloud auth activate-service-account --key-file=$KUBECONFIG_FILE
                gcloud container clusters get-credentials $CLUSTER_NAME --region ${LOCATION} --project $PROJECT_ID
                kubectl config current-context
                kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.3/deploy/static/provider/cloud/deploy.yaml
                kubectl get ingress
            '''
          }
        }
    }
    stage('Deploy to GKE') {
        steps{
          // Deploy öncesi image tag güncelleme
          sh '''
            sed -i "s|IMAGE_TAG_WEB_SERVER|${ARTIFACT_REGISTRY}/${APP_REPO_NAME}:web-b${BUILD_NUMBER}|" k8s/manifest.yml
            sed -i "s|IMAGE_TAG_RESULT_SERVER|${ARTIFACT_REGISTRY}/${APP_REPO_NAME}:result-b${BUILD_NUMBER}|" k8s/manifest.yml
          '''
          step([
          $class: 'KubernetesEngineBuilder',
          projectId: env.PROJECT_ID,
          clusterName: env.CLUSTER_NAME,
          location: env.LOCATION,
          manifestPattern: 'k8s/manifest.yml',
          namespace: env.NAMESPACE,
          credentialsId: env.CREDENTIALS_ID2,
          verifyDeployments: true,
          verifyTimeoutInMinutes: 5])
        }
    }

}
}