# Openshift CI/CD for simple DotNet Application

This Project Handle the basic CI/CD of DotNet application 

To run locally with DotNet installed:   

```
dotnet build
dotnet test 
dotnet run
```

To use Jenkins on Openshift for CI/CD, first we need to build DotNet Jenkins Slave template to use in our CI/CD 

## 1) Build The Environment

You can run use the build :  

```
oc project cicd //this is the project for cicd
oc project dev //this is project for application development

oc new-app jenkins-persistent  -n cicd
oc policy add-role-to-user edit system:serviceaccount:cicd:default -n dev
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n dev

oc create -f bc_jenkins_slave.yaml -n cicd //this will add the template to use 
or you can use it directly from the GitHub: oc process -f https://raw.githubusercontent.com/osa-ora/sample_dotnet/main/cicd/bc_jenkins_slave.yaml -n cicd | oc create -f -

Now use the template to create the Jenkins slave template
oc describe template jenkins-slave //to see the template details
oc process -p GIT_URL=https://github.com/osa-ora/sample_dotnet -p GIT_BRANCH=main -p GIT_CONTEXT_DIR=cicd -p DOCKERFILE_PATH=dockerfile_dotnet_node -p IMAGE_NAME=jenkins-dotnet-slave jenkins-slave | oc create -f -

oc start-build jenkins-dotnet-slave 
oc logs bc/jenkins-dotnet-slave -f
```


## 2) Configure Jenkins 
From inside Jenkins --> go to Manage Jenkins ==> Configure Jenkins then scroll to cloud section:
https://{JENKINS_URL}/configureClouds
Now click on Pod Templates, add new one with name "dotnet", label "dotnet", container template name "jnlp", docker image "image-registry.openshift-image-registry.svc:5000/cicd/jenkins-dotnet-slave" 

See the picture:
<img width="1303" alt="Screen Shot 2021-01-03 at 17 04 16" src="https://user-images.githubusercontent.com/18471537/103481802-be456300-4de5-11eb-8a69-038da8cb7a42.png">

## 3) (Optional) SonarQube on Openshift
Provision SonarQube for code scanning on Openshift using the attached template.
oc process -f https://raw.githubusercontent.com/osa-ora/sample_dotnet/main/cicd/sonarcube-persistent-template.yaml | oc create -f -

Open SonarQube and create new project, give it a name, generate a token and use them as parameters in our next CI/CD steps

<img width="808" alt="Screen Shot 2021-01-03 at 17 01 17" src="https://user-images.githubusercontent.com/18471537/103481690-55f68180-4de5-11eb-8205-76cf44801c2a.png">

Make sure to select DotNet here.

## 4) Build Jenkins CI/CD using Jenkins File

Now create new pipeline for the project, where we checkout the code, run unit testing, run sonar qube analysis, build the application, get manual approval for deployment and finally deploy it on Openshift.
Here is the content of the file:

```
pipeline {
	options {
		// set a timeout of 20 minutes for this pipeline
		timeout(time: 20, unit: 'MINUTES')
	}
    agent {
    // Using the dotnet builder agent
       label "dotnet"
    }
    stages {
    stage('Checkout') {
      steps {
        git branch: 'main', url: '${git_url}'
        sh "ls -l"
        sh "oc version"
      }
    }
    stage('Unit Testing') {
      steps {
        sh "dotnet test ${unit_tests_folder}"
        sh "ls -l ${unit_tests_folder}"
        sh "ls -l ${unit_tests_folder}/TestResults"
        sh "rm -R ${unit_tests_folder}"
        sh "ls -l"
      }
    }
    stage('Sonar Qube') {
      steps {
        sh "dotnet sonarscanner begin /k:\"${sonarqube_proj}\" /d:sonar.host.url=\"${sonarqube_url}\" /d:sonar.login=\"${sonarqube_token}\""
        sh "dotnet build"
        sh "dotnet sonarscanner end /d:sonar.login=\"${sonarqube_token}\""
      }
    }
    stage('Deployment Approval') {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                input message: 'Proceed with Application Deployment?', ok: 'Approve Deployment'
            }
        }
    }
    stage('Deploy To Openshift') {
      steps {
        sh "oc project ${proj_name}"
        sh "oc start-build ${app_name} --from-dir=."
        sh "oc logs -f bc/${app_name}"
      }
    }
  }
} // pipeline
```
As you can see this pipeline pick the gradle slave image that we built, note the label must match what we configurd before:
```
agent {
    // Using the dotnet builder agent
       label "dotnet"
    }
```

The pipeline uses many parameters:
```
- String parameter: proj_name //this is Openshift project for the application
- String parameter: app_name //this is the application name
- String parameter: git_url //this is the git url of our project, default is https://github.com/osa-ora/sample_dotnet
- String parameter: sonarqube_url: //this is the sonarqube url in case it is used for code scanning
- String parameter: sonarqube_token //this is the token 
- String parameter: sonarqube_proj // the project name in sonarqube
```

<img width="1270" alt="Screen Shot 2021-01-03 at 15 47 50" src="https://user-images.githubusercontent.com/18471537/103480534-92be7a80-4ddd-11eb-96b2-23007d19c242.png">

The project assume that you already build and deployed the application before, so we need to have a Jenkins freestyle project where we initally execute the following commands in Jenkins after checkout the code:
```
rm -R Tests
oc project ${proj_name}
oc new-build --image-stream=dotnet:latest --binary=true --name=${app_name}
oc start-build ${app_name} --from-dir=.
oc logs -f bc/${app_name}
oc new-app ${app_name} --as-deployment-config
oc expose svc ${app_name} --port=8080 --name=${app_name}
```
This will make sure our project initally deployed and ready for our CI/CD configurations, where proj_name and app_name is Openshift project and application name respectively.

You can enrich the project and add approvals to deploy to differnet environments as well, all you need is to have the privilages using "oc policy" as we did before and add deploy stage in the pipeline step.


