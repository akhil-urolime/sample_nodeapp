properties([
  // [
  //   $class: 'GithubProjectProperty',
  //   displayName: '',
  //   projectUrlStr: 'https://github.com/akhil-urolime/sample_nodeapp.git'
  // ],
  parameters([
    [
      $class: 'CredentialsParameterDefinition',
      credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl',
      defaultValue: 'AWS_ACCOUNT_ID',
      description: '',
      name: 'AWS_ACCOUNT_ID',
      required: true
    ],
    [
      $class: 'CredentialsParameterDefinition',
      credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl',
      defaultValue: '',
      description: '',
      name: 'AWS_Key_and_Secret',
      required: true
    ]   
  ]),
  pipelineTriggers([[$class: 'GitHubPushTrigger']])
])

def remoteImageTag  = "test-10"

def taskDefile      = "file://aws/task-definition-${remoteImageTag}.json"
def ecRegistry      = "https://753233110933.dkr.ecr.us-west-1.amazonaws.com"

def version = ''
def gitCommit = ''
def dockerPushToEcr(region, remoteRepositoryPathAndImageName, localImageName) {
  docker.withServer('tcp://localhost:2375') {
    withCredentials([[$class: 'StringBinding', credentialsId: AWS_ACCOUNT_ID, variable: 'AWS_ACCOUNT_ID'], [$class: 'UsernamePasswordMultiBinding', credentialsId: AWS_Key_and_Secret, passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID']]) {
      sh """
echo "Git Tag"
git describe --tags
set +e
git_tag="DEPLOY.pod.1.3"
aws ecr describe-repositories --region $region --repository-names $remoteRepositoryPathAndImageName
create_result=\$?
set -e
if [ \$create_result != 0 ]; then
echo "Creating repository $remoteRepositoryPathAndImageName"
aws ecr create-repository --region $region --repository-name $remoteRepositoryPathAndImageName
fi
docker_login=\$(aws ecr get-login --region $region)
login_result=\$(\$docker_login)
latest_tag="${env.AWS_ACCOUNT_ID}.dkr.ecr.${region}.amazonaws.com/$remoteRepositoryPathAndImageName:$git_tag"
docker tag $localImageName:latest \$latest_tag
docker push \$latest_tag
nildigests=\$(aws ecr list-images --region $region --repository-name "$remoteRepositoryPathAndImageName" | jq -r '.imageIds[] | select(has("imageTag") | not) | .imageDigest')
for i in \$nildigests; do
aws ecr batch-delete-image --region $region --repository-name "$remoteRepositoryPathAndImageName" --image-ids "imageDigest=\$i"
done
"""
    }
  }
}
node{

    stage('Checkout source repo') {
      git 'https://github.com/akhil-urolime/sample_nodeapp.git'     
      checkout scm
    }

    stage('Build docker image') {
    
    docker.withServer('tcp://localhost:2375') {
            docker.build('podchaser-demo')
      }
    }
    stage("Publish docker image in us-east-1") {
      dockerPushToEcr('us-west-1', 'podchaser', 'podchaser-demo')
    }
    stage("Deploy") {
        // Replace BUILD_TAG placeholder in the task-definition file -
        // with the remoteImageTag (imageTag-BUILD_NUMBER)
        sh  "                                                                     \
          sed -e  's;%BUILD_TAG%;${remoteImageTag};g'                             \
                  aws/task-definition.json >                                      \
                  aws/task-definition-${remoteImageTag}.json                      \
        "
        // Get current [TaskDefinition#revision-number]
        def currTaskDef = sh (
          returnStdout: true,
          script:  "                                                              \
            aws ecs describe-task-definition  --task-definition pod-task          \
                                              | egrep 'revision'                  \
                                              | tr ',' ' '                        \
                                              | awk '{print \$2}'                 \
          "
        ).trim()

        def currentTask = sh (
          returnStdout: true,
          script:  "                                                              \
            aws ecs list-tasks  --cluster pod1                                    \
                                --family  pod-task                                \
                                --output text                                     \
                                | egrep 'TASKARNS'                                \
                                | awk '{print \$2}'                               \
          "
        ).trim()

        /*
        / Scale down the service
        /   Note: specifiying desired-count of a task-definition in a service -
        /   should be fine for scaling down the service, and starting a new task,
        /   but due to the limited resources (Only one VM instance) is running
        /   there will be a problem where one container is already running/VM,
        /   and using a port(80/443), then when trying to update the service -
        /   with a new task, it will complaine as the port is already being used,
        /   as long as scaling down the service/starting new task run simulatenously
        /   and it is very likely that starting task will run before the scaling down service finish
        /   so.. we need to manually stop the task via aws ecs stop-task.
        */
        if(currTaskDef) {
          sh  "                                                                   \
            aws ecs update-service  --cluster pod1                                \
                                    --service service1                            \
                                    --task-definition pod-task:${currTaskDef}     \
                                    --desired-count 0                             \
          "
        }
        if (currentTask) {
          sh "aws ecs stop-task --cluster ${clusterName} --task ${currentTask}"
        }

        // Register the new [TaskDefinition]
        sh  "                                                                     \
          aws ecs register-task-definition  --family  pod-task                    \
                                            --cli-input-json ${taskDefile}        \
        "

        // Get the last registered [TaskDefinition#revision]
        def taskRevision = sh (
          returnStdout: true,
          script:  "                                                              \
            aws ecs describe-task-definition  --task-definition pod-task          \
                                              | egrep 'revision'                  \
                                              | tr ',' ' '                        \
                                              | awk '{print \$2}'                 \
          "
        ).trim()

        // ECS update service to use the newly registered [TaskDefinition#revision]
        //
        sh  "                                                                     \
          aws ecs update-service  --cluster pod1                                  \
                                  --service service1                              \
                                  --task-definition pod-task:${taskRevision}      \
                                  --desired-count 1                               \
        "
      }

}
