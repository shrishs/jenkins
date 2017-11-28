
http://github.com/shrishs/jenkins

Step1 :Adding Basic pipeline

--Create a Jenkinsfile with the following content or Can use the Jenkinsfile1

node('maven') {
stage 'buildInDevelopment'
stage 'deployInDevelopment'
stage 'deployInTesting'
}

**Instead node('maven') ,one can also use node {} as we are not going to use any maven specific activities.
---Configure CICD Project

oc new-project cicd

oc new-build https://github.com/shrishs/jenkins --context-dir=jenkinsfile1 --name=sample-pipeline --strategy=pipeline

---Check the jenkins instance.
---Log in using openshift credential
---Check Jenkins service account

Above command can also be run by providing jenkinsfile content inside the buildconfig and creating the buildconfig object as follows.

oc create -f jenkinsbc.yaml

---Create a development Project

oc new-project development

oc new-app eap70-basic-s2i -e SOURCE_REPOSITORY_URL=https://github.com/jboss-developer/jboss-eap-quickstarts -e CONTEXT_DIR=kitchensink

**Instead one can create project ui

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
   input message: "Promote to STAGE?", ok: "Promote"
openshiftTag(namespace: 'development', sourceStream: 'eap-app',  sourceTag: 'latest', destinationStream: 'eap-app', destinationTag: 'promoteToQA')
openshiftDeploy(namespace: 'testing', deploymentConfig: 'eap-app', )
openshiftScale(namespace: 'testing', deploymentConfig: 'eap-app',replicaCount: '2')
}


--Create the deployment config in testing project.

oc create deploymentconfig eap-app --image=172.30.90.23:5000/development/eap-app:promoteToQA
To update the imagePullPomicy to Always instead of IfNotPresent

oc patch dc/eap-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"default-container","imagePullPolicy":"Always"}]}}}}'


**Instead of doing above two step ,One can copy the dc from development

oc export dc -n development -o yaml >java-appdc.yaml

Change the below line in java-appdc.yaml

- image: xx.xxx.xx.xxx:5000/development/java-app@sha256:e3e310f3b27251fa164d20f1a81cb3ca9a4d6e20146ddfa117f4174f964b4f8d
to
- image: xx.xxx.xx.xxx:5000/development/java-app:promoteToQA
oc create -f java-appdc.yaml -n testing



--Create the service in testing project.
oc expose dc eap-app --port=8080

--Create the route in testing project.
oc expose svc eap-app

--Provide the image puller role to complete Service Account grpup
oc policy add-role-to-group system:image-puller system:serviceaccounts:testing -n development

