# Introduction
This readme contains the Custom Resource Definitions for a pipeline usable by Tekton and Openshift Pipelines. This pipeline is designed to deploy a container image for a simple application made up of two microservices called [calc](https://github.com/GregJohnStewart/calc) and [calc-cache](https://github.com/GregJohnStewart/calc-cache).

# Files
The repo contains the following files:
- pipeline-java.yml
    - Contains pipeline resource to build a container image for calc-cache, the Java microservice
    - Runs the following tasks:
        1. Clone the calc-cache repository using the 'git-clone' cluster task
        2. Build the java binaries using the maven cluster task
        3. Run Sonarscanner on the java binaries using a docker image of sonarscanner from Docker Hub
        4. Create and push an image based on the java binaries to the Openshift internal registry using the 'buildah' cluster task
- pipeline-dotnet.yml
    - Contains the pipeline resource to build a container image for calc, the .NET microservice
    - Runs the following tasks:
        1. Clone the calc repository using the 'git-clone' cluster task
        2. Utilizing a custom task and custom [container image](https://quay.io/repository/rh_ee_andsmith/sonarscanner), build .NET application while also running the sonarscanner that is provided by .NET as a global tool
- eventListner-java.yml
    - Contains the trigger definition that listens to a github webhook and initiates a pipeline run once a request is recieved from the calc-cache repository
    - Once executed, the trigger will create a PipelineRun resource with the following parameters and workspaces:
    - Parameters
        - GIT_REPO: repo to clone for the 'fetch-repository' task, which in this case is calc-cache
        - GIT_REVISION: the branch, tag, hash, etc. to clone down from the GIT_REPO repository, which is main in this case
    - Workspaces
        - source: volume claim template that creates a PVC where the code from the GIT_REPO will be stored as it moves through the pipeline
        - sonnarscanner-api-token-calc-cache: Sonnarscanner api token that gives access to push reports to the sonarqube instance for the calc-cache project
        - emptyDir: Empty directory to satisfy the requirements for the maven cluster task
- eventListener-dotnet.yml
    - Contains the trigger definition that listens to a github webhook and initiates a pipeline run once a request is recieved from the calc repository
    - Once executed, the trigger will create a PipelineRun resource with the following parameters and workspaces:
    - Parameters
        - GIT_REPO: repo to clone for the 'fetch-repository' task, which in this case is calc
        - GIT_REVISION: the branch, tag, hash, etc. to clone down from the GIT_REPO repository, which is main in this case
    - Workspaces
        - source: volume claim template that creates a PVC where the code from the GIT_REPO will be stored as it moves through the pipeline
        - sonnarscanner-api-token-calc-cache: Sonnarscanner api token that gives access to push reports to the sonarqube instance for the calc-cache project

# Deployment
In order to push these resource files to an Openshift Cluster, please perform the following:

1. Make sure that a Sonarqube server has been provisioned and is available to the Openshift cluster
2. Create projects inside of Sonarqube for calc and calc-cache
3. Create api tokens for the calc and calc-cache sonarqube projects
4. Take those sonarqube project tokens and put them into Openshift Secrets, with the key for each being 'token'
5. In the pipeline-dotnet.yml file, replace the 'sonar.host.url' with the url of the location where the sonarqube server is located
6. In the pipeline-java.yml file, replace the SONAR_HOST_URL env with the url for your sonarqube instance
7. In the eventListener-java.yml and eventListener-dotnet.yml files, replace the GIT_URL parameter with a forked copy of calc-cache and calc respectively
8. Create a ConfigMap that give values the following variables:
  - QUARKUS_DATASOURCE_JDBC_URL
  - quarkus.hibernate-orm.database.generation
  - quarkus.rest-client.calc-service.url
  - Here are some defaults:
  `QUARKUS_DATASOURCE_JDBC_URL: 'jdbc:mysql://mysql/calc'
  quarkus.hibernate-orm.database.generation: drop-and-create
  quarkus.rest-client.calc-service.url: 'http://microservice-dotnet:8080'`
  
9. Use `oc create -f pipeline-dotnet.yml` to deploy the .NET pipeline
10. Use `oc create -f pipeline-java.yml` to deploy the Java pipeline
11. Use `oc create -f eventListener-java.yml` to deploy the Java trigger
12. Use `oc create -f eventListener-dotnet.yml` to deploy the .NET trigger
13. Create routes that expose the services created by the Java and .NET triggers
14. Start the .NET and Java pipelines to create some initial container images in the container registry
15. Use `oc new-app microservice-dotnet` and `oc new-app microservice-java` to create the initial Openshift resources required to deploy the microservices on OpenShift

# Questions?
- If you have any questions regarding the the files or deployment process of these applications, reach out to either Andrew Smith <andsmith@redhat.com> or Greg Stewart <gstewart@redhat.com>
