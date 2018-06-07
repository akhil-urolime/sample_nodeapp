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

def version = ''
def gitCommit = ''
def dockerPushToEcr(region, remoteRepositoryPathAndImageName, localImageName) {
  docker.withServer('tcp://localhost:2375') {
    withCredentials([[$class: 'StringBinding', credentialsId: AWS_ACCOUNT_ID, variable: 'AWS_ACCOUNT_ID'], [$class: 'UsernamePasswordMultiBinding', credentialsId: AWS_Key_and_Secret, passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID']]) {
      sh """
set +e
aws ecr describe-repositories --region $region --repository-names $remoteRepositoryPathAndImageName
create_result=\$?
set -e
if [ \$create_result != 0 ]; then
echo "Creating repository $remoteRepositoryPathAndImageName"
aws ecr create-repository --region $region --repository-name $remoteRepositoryPathAndImageName
fi
docker_login=\$(aws ecr get-login --region $region)
login_result=\$(\$docker_login)
latest_tag="${env.AWS_ACCOUNT_ID}.dkr.ecr.${region}.amazonaws.com/$remoteRepositoryPathAndImageName:latest"
// sha_tag="${env.AWS_ACCOUNT_ID}.dkr.ecr.${region}.amazonaws.com/$remoteRepositoryPathAndImageName:$commitSha"
docker tag $localImageName:latest \$latest_tag
//docker tag $localImageName:latest \$sha_tag
docker push \$latest_tag
//docker push \$sha_tag

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
}
