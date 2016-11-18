#!/usr/bin/groovy
@Library('github.com/fabric8io/fabric8-pipeline-library@master')

def utils = new io.fabric8.Utils()

node {

  def envStage = utils.environmentNamespace('staging')
  def envProd = utils.environmentNamespace('production')
  def newVersion = ""

  def PROJECT_NAME = "almighty-core"
  def PACKAGE_NAME = 'github.com/almighty/almighty-core'
  def GOPATH_IN_CONTAINER="/tmp/go"
  def DOCKER_BUILD_DIR = "${env.WORKSPACE}/${PROJECT_NAME}-build"
  def DOCKER_IMAGE_CORE = "${PROJECT_NAME}"
  def DOCKER_IMAGE_DEPLOY = "${PROJECT_NAME}-deploy"
  def DOCKER_RUN_INTERACTIVE_SWITCH = ""
  def BUILD_TAG = "${PROJECT_NAME}-local-build"
  def DOCKER_CONTAINER_NAME = "${BUILD_TAG}"
  def PACKAGE_PATH= "${GOPATH_IN_CONTAINER}/src/${PACKAGE_NAME}"

  clientsNode{

    stage 'Checkout'
    def checkoutDir = "go/src/${PACKAGE_NAME}"
    sh "mkdir -pv ${checkoutDir}"
    dir ("${checkoutDir}") {
      checkout scm
      newVersion = sh(returnStdout: true, script: 'git rev-parse `git rev-parse --abbrev-ref HEAD`').take(6)
    }
    env.setProperty('VERSION',newVersion)


    def CUR_DIR = pwd() + "/${checkoutDir}"

    //def GROUP_ID = sh(returnStdout: true, script: 'id -g').trim()
    //def USER_ID = sh(returnStdout: true, script: 'id -u').trim()
    //echo "-u ${USER_ID}:${GROUP_ID}"

    def namespace = utils.getNamespace()
    def newImageName = "${env.FABRIC8_DOCKER_REGISTRY_SERVICE_HOST}:${env.FABRIC8_DOCKER_REGISTRY_SERVICE_PORT}/${namespace}/${env.JOB_NAME}:${newVersion}"

    container('client') {

      stage 'Create Builder'
      //sh "make docker-start"
      sh "mkdir -p ${DOCKER_BUILD_DIR}"
      sh "docker build -t ${DOCKER_IMAGE_CORE} -f ${CUR_DIR}/Dockerfile.builder ${CUR_DIR}"
      sh "ls -la ${CUR_DIR}"
      sh "docker run --detach=true -t ${DOCKER_RUN_INTERACTIVE_SWITCH} --name=\"${DOCKER_CONTAINER_NAME}\" -e GOPATH=${GOPATH_IN_CONTAINER}	-w ${PACKAGE_PATH} ${DOCKER_IMAGE_CORE}"


      stage 'Get Deps'
      //sh "make docker-deps"
      sh "docker exec -t ${DOCKER_RUN_INTERACTIVE_SWITCH} \"${DOCKER_CONTAINER_NAME}\" bash -ec 'ls -la'"
      sh "docker exec -t ${DOCKER_RUN_INTERACTIVE_SWITCH} \"${DOCKER_CONTAINER_NAME}\" bash -ec 'make deps'"
      
      stage 'Generate'
      //sh "make docker-generate"
      sh "docker exec -t ${DOCKER_RUN_INTERACTIVE_SWITCH} \"${DOCKER_CONTAINER_NAME}\" bash -ec 'make generate'"

      stage 'Compile'
      //sh "make docker-build"
      sh "docker exec -t ${DOCKER_RUN_INTERACTIVE_SWITCH} \"${DOCKER_CONTAINER_NAME}\" bash -ec 'make build'"

      stage 'Run UnitTests'
      //sh "make docker-test-unit"
      sh "docker exec -t ${DOCKER_RUN_INTERACTIVE_SWITCH} \"${DOCKER_CONTAINER_NAME}\" bash -ec 'make test-unit'"

      stage 'Get BuilArifacts'
      sh "docker cp \"${DOCKER_CONTAINER_NAME}\":${PACKAGE_PATH}/bin/ ${CUR_DIR}"

      stage 'Delete Builder'
      sh "docker rm --force ${DOCKER_CONTAINER_NAME}"

      stage 'Create Runtime'
      //sh "make docker-image-deploy"
      sh "docker build -t ${DOCKER_IMAGE_DEPLOY} -f ${CUR_DIR}/Dockerfile.deploy ${CUR_DIR}"

      stage 'Push Runtime'
      sh "docker tag ${DOCKER_IMAGE_DEPLOY} ${newImageName}"
      sh "docker push ${newImageName}"

    }
  }

  def rc = getKubernetesJson {
    port = 8080
    label = 'golang'
    icon = 'https://cdn.rawgit.com/fabric8io/fabric8/dc05040/website/src/images/logos/gopher.png'
    version = newVersion
    imageName = clusterImageName
  }
  
  rc = setupEnv(rc)

  stage 'Rollout Staging'
  kubernetesApply(file: rc, environment: envStage)

  stage 'Approve'
  approve{
    room = null
    version = canaryVersion
    console = fabric8Console
    environment = envStage
  }

  stage 'Rollout Production'
  kubernetesApply(file: rc, environment: envProd)
}

@NonCPS
def setupEnv(json) {

  def slurp = new groovy.json.JsonSlurperClassic().parseText(json)
  def rc = new groovy.json.JsonBuilder(slurp)  

  env = rc.objects[1].spec.template.spec[0].env
  env << [name: "ALMIGHTY_POSTGRES_HOST", value: "db"] 
  env << [name: "ALMIGHTY_POSTGRES_PORT", value: "5432"]
  env << [name: "ALMIGHTY_POSTGRES_USER", value: "postgres"]
  env << [name: "ALMIGHTY_POSTGRES_PASSWORD", value: "mysecretpassword"]
  
  return new groovy.json.JsonBuilder(rc).toPrettyString()
}