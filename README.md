# jenkins
Step1 :Adding Basic pipeline

--Create a Jenkinsfile with the following content
node('maven') {
stage 'buildInDevelopment'
stage 'deployInDevelopment'
stage 'deployInTesting'
}


---Configure CICD Project
oc new-project cicd
oc new-build https://github.com/shrishs/jenkins --context-dir=jenkinsfile1 --name=sample-pipeline --strategy=pipeline
---Check the jenkins instance.
---Log in using openshift credential
---Check Jenkins service account

---Create a development Project
oc new-project development
oc new-app eap70-basic-s2i -e SOURCE_REPOSITORY_URL=https://github.com/jboss-developer/jboss-eap-quickstarts -e CONTEXT_DIR=kitchensink


--Call the build config create in development project in cicd pipeline

Change  the pipeline by changing the jenkins file with the following content or Can use the Jenkinsfile2
node('maven') {
stage 'buildInDevelopment'
openshiftBuild(namespace: 'development',buildConfig: 'eap-app', showBuildLogs: 'true')
stage 'deployInDevelopment'
stage 'deployInTesting'
}

--Start the pipeline:One will the get following exception
com.openshift.restclient.authorization.ResourceForbiddenException: User "system:serviceaccount:cicd:jenkins" cannot get buildconfigs in project "development" 


Give jenkins use development role on development project
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n development
--Start the pipeline:Pipeline will be executed successfully

Step2 :Promoting Images created in Development project to Testing

--Configure a testing project.
oc new-project testing
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n testing


--Change the jenkins file with the following content or can use Jenkinsfile3
node('maven') {
stage 'buildInDevelopment'
openshiftBuild(namespace: 'development',buildConfig: 'eap-app', showBuildLogs: 'true')
stage 'deployInDevelopment'
openshiftDeploy(namespace: 'development', deploymentConfig: 'eap-app')
openshiftScale(namespace: 'development', deploymentConfig: 'eap-app',replicaCount: '1')
stage 'deployInTesting'
openshiftTag(namespace: 'development', sourceStream: 'eap-app',  sourceTag: 'latest', destinationStream: 'eap-app', destinationTag: 'promoteToQA')
openshiftDeploy(namespace: 'testing', deploymentConfig: 'eap-app', )
openshiftScale(namespace: 'testing', deploymentConfig: 'eap-app',replicaCount: '2')
}


--Create the deployment config in testing project.
oc create deploymentconfig eap-app --image=172.30.90.23:5000/development/eap-app:promoteToQA
--Create the service in testing project.
oc expose dc eap-app --port=8080
--Create the route in testing project.
oc expose svc eap-app
--Provide the image puller role to complete Service Account grpup
oc policy add-role-to-group system:image-puller system:serviceaccounts:testing -n development

