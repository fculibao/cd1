env.DOCKERHUB_USERNAME = 'fculibao'

  node {
    checkout scm

    stage("Unit Test") {
      sh "docker run --rm -v ${WORKSPACE}:/go/src/cd1 golang go test cd1 -v --run Unit"
    }
    stage("Integration Test") {
      try {
        sh "docker build -t cd1 ."
        sh "docker rm -f cd1 || true"
        sh "docker run -d -p 8080:8080 --name=cd1 cd1"
        // env variable is used to set the server where go test will connect to run the test
        sh "docker run --rm -v ${WORKSPACE}:/go/src/cd1 --link=cd1 -e SERVER=cd1 golang go test cd1 -v --run Integration"
      }
      catch(e) {
        error "Integration Test failed"
      }finally {
        sh "docker rm -f cd1 || true"
        sh "docker ps -aq | xargs docker rm || true"
        sh "docker images -aq -f dangling=true | xargs docker rmi || true"
      }
    }
    stage("Build") {
      sh "docker build -t ${DOCKERHUB_USERNAME}/cd1:${BUILD_NUMBER} ."
    }
    stage("Publish") {
      withDockerRegistry([credentialsId: 'DockerHub']) {
        sh "docker push ${DOCKERHUB_USERNAME}/cd1:${BUILD_NUMBER}"
      }
    }
  }

  node {
    checkout scm

    stage("Staging") {
      try {
        sh "docker rm -f cd1 || true"
        sh "docker run -d -p 8080:8080 --name=cd1 ${DOCKERHUB_USERNAME}/cd1:${BUILD_NUMBER}"
        sh "docker run --rm -v ${WORKSPACE}:/go/src/cd1 --link=cd1 -e SERVER=cd1 golang go test cd1 -v"

      } catch(e) {
        error "Staging failed"
      } finally {
        sh "docker rm -f cd1 || true"
        sh "docker ps -aq | xargs docker rm || true"
        sh "docker images -aq -f dangling=true | xargs docker rmi || true"
      }
    }
  }

  node {
    stage("Production") {
      try {
        // Create the service if it doesn't exist otherwise just update the image
        sh '''
          SERVICES=$(docker service ls --filter name=cd1 --quiet | wc -l)
          if [[ "$SERVICES" -eq 0 ]]; then
            docker network rm cd1 || true
            docker network create --driver overlay --attachable cd1
            docker service create --replicas 3 --network cd1 --name cd1 -p 8080:8080 ${DOCKERHUB_USERNAME}/cd1:${BUILD_NUMBER}
          else
            docker service update --image ${DOCKERHUB_USERNAME}/cd1:${BUILD_NUMBER} cd1
          fi
          '''
        // run some final tests in production
        checkout scm
        sh '''
          sleep 60s 
          for i in `seq 1 20`;
          do
            STATUS=$(docker service inspect --format '{{ .UpdateStatus.State }}' cd1)
            if [[ "$STATUS" != "updating" ]]; then
              docker run --rm -v ${WORKSPACE}:/go/src/cd1 --network cd1 -e SERVER=cd1 golang go test cd1 -v --run Integration
              break
            fi
            sleep 10s
          done
          
        '''
      }catch(e) {
        sh "docker service update --rollback  cd1"
        error "Service update failed in production"
      }finally {
        sh "docker ps -aq | xargs docker rm || true"
      }
    }
  }

