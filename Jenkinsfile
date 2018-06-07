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

node{

    stage('Checkout source repo') {
      git 'https://github.com/akhil-urolime/sample_nodeapp.git'
      checkout scm
    }

    stage('Build docker image') {
    
    docker.withServer('tcp://localhost:2375') {
            docker.build demo

      }
    }

}
