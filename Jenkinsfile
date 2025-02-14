pipeline {
  agent any
  // ENV
  environment {
    DOCKER_ID = "codask"
    DOCKER_IMAGE = "movie-db"
    DOCKER_TAG = "v.${BUILD_ID}.0"
    NAMESPACE=credentials("NAMESPACE")
    DOCKER_PASS = credentials("DOCKER_HUB_PASS")
    KUBECONFIG = credentials("config")
    BRANCH = "${BRANCH}"
  }    
  stages {
    stage('Prod deployment accepted') {
      steps {
        script {                  
          if ("${BRANCH}" != "master" && "${NAMESPACE}" == "prod") {
            echo "Stop"
            return
          }
        }
      }
    }
    /// =====================
    //      DOCKERHUB
    /// =====================
    // Build & push movie_db
    stage('Build & push movie-db') { 
      environment {
        DOCKER_IMAGE = "movie_db"
      }
      steps {
        script {
          sh '''
            docker login -u $DOCKER_ID -p $DOCKER_PASS
            cd movie-service
            docker build -t $DOCKER_ID/$DOCKER_IMAGE .
            docker push $DOCKER_ID/$DOCKER_IMAGE
            cd ~
          '''
        }
      }
    }
    // Build & push cast_db
    stage('Build & push cast-db') { 
      environment {
        DOCKER_IMAGE = "cast_db"
      }
      steps {
        script {
          sh '''
            docker login -u $DOCKER_ID -p $DOCKER_PASS
            cd cast-service
            docker build -t $DOCKER_ID/$DOCKER_IMAGE .
            docker push $DOCKER_ID/$DOCKER_IMAGE
            cd ~
          '''
        }
      }
    }
    /// =====================
    //         K8S
    /// =====================
    // Deploy movie_service, cast_service, nginx & fasapiapp
    stage('Deploy') {
      environment {
        HELM_PATH = "/usr/local/bin/helm"
      }
      steps {
        script {
          sleep 120
        }
        script {
          sh """
            rm -Rf .kube
            mkdir .kube
            cat $KUBECONFIG > .kube/config
            $HELM_PATH upgrade -i movie-db ./movie-service --namespace ${NAMESPACE} --set namespace=${NAMESPACE}
            $HELM_PATH upgrade -i cast-db ./cast-service --namespace ${NAMESPACE} --set namespace=${NAMESPACE}
            $HELM_PATH upgrade -i nginx ./nginx --namespace ${NAMESPACE} --set namespace=${NAMESPACE}
            $HELM_PATH upgrade -i fastapiapp ./charts --namespace ${NAMESPACE} --set namespace=${NAMESPACE}
          """
        }
      }
    }
    /// =====================
    //          TESTING
    /// =====================
    // Test connections (movies & casts)
    stage ('Tests') {
      environment {
        movie_data = '{"name":"Starsky & Hutch","plot":"string","genres":["Policy"],"casts_id":[1],"id":2}'
        cast_data = '{"name": "Todd Phillips","nationality": "American","id": 1}'
      }      
      steps ("Tests connections") {
        script {
          sleep 120
        }
        script {
          command = """ 
            curl -X 'POST' 'http://63.34.162.113:8090/api/v1/movies/' -H 'accept: application/json' \
            -H 'Content-Type: application/json' \
            -d '{
            "name": "Starsky & Hutch",
            "plot": "string",
            "genres": [ "Policy"],
            "casts_id": [1]
          }' """
          result = sh ( script: command, returnStdout:true)
          println "result: \n" + result
        }
        script {
          command = """
            curl -X 'POST' \
            'http://63.34.162.113:8090/api/v1/casts/' \
            -H 'accept: application/json' \
            -H 'Content-Type: application/json' \
            -d '{
            "name": "Todd Phillips",
            "nationality": "American"
          }'"""
          result = sh ( script: command, returnStdout:true)
          println "result: \n" + result
        }
      }
    } 
  }
}
