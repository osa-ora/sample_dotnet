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

oc create -f bc_jenkins_slave_template.yaml -n cicd //this will add the template to use 
or you can use it directly from the GitHub: oc process -f https://raw.githubusercontent.com/osa-ora/sample_dotnet/main/cicd/bc_jenkins_slave_template.yaml -n cicd | oc create -f -

Now use the template to create the Jenkins slave template
oc describe template jenkins-slave-template //to see the template details
oc process -p GIT_URL=https://github.com/osa-ora/sample_dotnet -p GIT_BRANCH=main -p GIT_CONTEXT_DIR=cicd -p DOCKERFILE_PATH=dockerfile_dotnet_node -p IMAGE_NAME=jenkins-dotnet-slave jenkins-slave-template | oc create -f -

oc start-build jenkins-dotnet-slave 
oc logs bc/jenkins-dotnet-slave -f

oc new-app jenkins-persistent  -n cicd

oc project dev //this is project for application development
oc policy add-role-to-user edit system:serviceaccount:cicd:default -n dev
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n dev

```


## 2) Configure Jenkins 
In case you completed 1st step before provision Openshift Jenkins, it will auto-detect the slave dotnet image based on the label and annotation and no thing need to be done to configure it, otherwise you can do it manually for existing Jenkins installation

From inside Jenkins --> go to Manage Jenkins ==> Configure Jenkins then scroll to cloud section:
https://{JENKINS_URL}/configureClouds
Now click on Pod Templates, add new one with name "jenkins-dotnet-slave", label "jenkins-dotnet-slave", container template name "jnlp", docker image "image-registry.openshift-image-registry.svc:5000/cicd/jenkins-dotnet-slave" 

See the picture:
<img width="1242" alt="Screen Shot 2021-01-04 at 12 09 05" src="https://user-images.githubusercontent.com/18471537/103524212-d2d93800-4e85-11eb-818b-21e7e8811ba4.png">

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
       label "jenkins-dotnet-slave"
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
       label "jenkins-dotnet-slave"
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

<img width="906" alt="Screen Shot 2021-01-03 at 18 51 22" src="https://user-images.githubusercontent.com/18471537/103484483-c1494f00-4df7-11eb-9841-bdaa8537f225.png">

## 5) Deployment Across Environments

Environments can be another Openshift project in the same Openshift cluster or in anither cluster.

In order to do this for the same cluster, you can enrich the pipeline and add approvals to deploy to a new project, all you need is to have the required privilages using "oc policy" as we did before and add deploy stage in the pipeline script to deploy into this project.

```
oc project {project_name} //this is new project to use
oc policy add-role-to-user edit system:serviceaccount:cicd:default -n {project_name}
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n {project_name}
```
Add more stages to the pipleine scripts like:
```
    stage('Deployment to Staging Approval') {
        steps {
            timeout(time: 5, unit: 'MINUTES') {
                input message: 'Proceed with Application Deployment in Staging environment ?', ok: 'Approve Deployment'
            }
        }
    }
    stage('Deploy To Openshift Staging') {
      steps {
        sh "oc project ${staging_proj_name}"
        sh "oc start-build ${app_name} --from-dir=."
        sh "oc logs -f bc/${app_name}"
      }
    }
```

Also you can use Openshift plugin and configure different Openshift cluster to automated the deployments across many environments:

```
stage('preamble') {
	steps {
		script {
			openshift.withCluster() {
			//could be openshift.withCluster( 'another-cluster' ) {
				//name references a Jenkins cluster configuration
				openshift.withProject() { 
				//coulld be openshift.withProject( 'another-project' ) {
					//name references a project inside the cluster
					echo "Using project: ${openshift.project()} in cluster:  ${openshift.cluster()}"
				}
			}
		}
	}
}
```
And then configure any additonal cluster (other than the default one which running Jenkins) in Openshift Client plugin configuration section:

<img width="1400" alt="Screen Shot 2021-01-05 at 10 24 45" src="https://user-images.githubusercontent.com/18471537/103623100-4b043400-4f40-11eb-90cf-2209e9c4bde2.png">

## 6) Copy Container Images
In case you need to copy container images between different Openshift clusters, you can use Skopeo command to do this.
The easiest way is to copy the container to Quay.io repository and then import it.

In the docker folder you can see a Skopeo Jenkins slave image where you can use it to execute the copy command. Here is the steps that you need to do this:

From Quay Side:
1. Create Quay.io account and repository for example test repository
2. Create a Robot account in the respository with efficient permission e.g. read/write.   
From Openshift Side:  
3. Create Skopeo Jenkins slave image:
```
oc process -p GIT_URL=https://github.com/osa-ora/sample_dotnet -p GIT_BRANCH=main -p GIT_CONTEXT_DIR=docker -p DOCKERFILE_PATH=dockerfile_skopeo -p IMAGE_NAME=jenkins-slave-skopeo jenkins-slave-template | oc create -f -
```
Make sure this new image exist in Jenkins Pod Templates as we did before.  
4. Create User with enough privilages to pull the image for example skopeo user and get the token of this user
You can use either image-puller or image-builder based on the required use case
```
oc create serviceaccount skopeo
oc adm policy add-role-to-user system:image-puller -n cicd system:serviceaccount:cicd:skopeo
oc adm policy add-role-to-user system:image-builder -n cicd system:serviceaccount:cicd:skopeo
oc describe secret skopeo-token -n cicd
```
You can also use any user with privilage to do this.  
5. Run Jenkins pipleline in that Skopeo slave to execute the copy command:
```
pipeline {
    agent {
       // Using the skopeo agent
       label "jenkins-slave-skopeo"
    }
    stages {
      stage('Copy Image To Quay') {
        steps {
          sh "skopeo copy docker://image-registry.openshift-image-registry.svc:5000/${ocp_img_name} docker://quay.io/${quay_repo} --src-tls-verify=false --src-creds='${ocp_id}' --dest-creds ${quay_id}"
        }
      }
      stage('Copy Image From Quay') {
        steps {
          sh "skopeo copy docker://quay.io/${quay_repo} docker://image-registry.openshift-image-registry.svc:5000/${ocp_img_name} --dest-tls-verify=false --format=v2s2 --src-creds='${quay_id}' --dest-creds ${ocp_id}"
        }
      }
    }
}
```
Where the following parameters are supplie for example:
```
quay_repo is the quay repository 
quay_id is the robot account:token granted access to Quay repository
ocp_img_name is the name of the image in OCP
ocp_id is username:token of user with privilages to pull/push image in OCP
```
Note that I used the flag srv-tls-verify=false and dest-tls-verity=false with OCP as in my environment it uses self signed certificate otherwise it will fail.  
Note also in case you encounter any issues in format versions you can use the flag "--format=v2s2"  
More details here: https://github.com/nmasse-itix/OpenShift-Examples/blob/master/Using-Skopeo/README.md

