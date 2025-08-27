# Create pipelines for Netflix app
https://github.com/doublebubblemethod/petclinic-orchestrated.git
It is generally recommended to separate the concerns of CI (Continuous Integration) and CD (Continuous Deployment) into distinct pipelines or stages:
    1. Separation of Concerns
    2. Clarity
    3. Faster Feedback
CI Pipeline (for validating code)
    Build: Compile your application or run tests.
    Static Analysis: Run tools like SonarQube, check code quality, and ensure there are no vulnerabilities.
    Unit Tests: Run automated tests to verify the functionality of the code.
    Code Coverage: Ensure that your code is well-tested.
    Build Docker Image (optional): Build the application’s Docker image, but do not push it until approval.
    Push Image: Optionally, push the image to the registry (but only if your pipeline passed all tests).

My CI Pipeline implementation:
    Scan:
        Clean Workspace: Clean the workspace before starting.
        Checkout from Git: Pull the latest code from your Git repository.
        Static Analysis: Run code analysis using SonarQube.
        Quality Gate: Check if the code meets quality standards and Code Coverage ≥ 80%
        Owasp Dependency Check
        Send email notification with buils status
    Build_and_Push:
        Build an image using kaniko
        Scan container image using Trivy
        Push image to container registry
        Send email notification with trivy report


CD Pipeline (for deploying to different environments)
    Deploy to Staging/Dev: First, deploy to a non-production environment to test the application in an environment similar to production.
    Acceptance Tests: Run smoke or end-to-end tests in the staging environment.
    Deploy to Production: After validation in staging, deploy to production.
    Manual Approval: Some teams prefer to have a manual approval step before pushing to production, even if automated testing passes.

My CD Pipeline implementation:
    Checkout from Git
    Docker Build & Push: Build a Docker image and push it to a container registry 
    Deploy to Kubernetes: Deploy the application to Kubernetes using kubectl to apply the necessary configuration files 
    
## Vocabulary
A Quality Gate checks the results of the SonarQube analysis and ensures that the code meets certain quality thresholds (such as a minimum level of code coverage, no critical vulnerabilities, no blocker issues, etc.). 
SonarQube's Built-in Quality Gate (Default) comes with: 
    New Code Coverage ≥ 80%
    No New Blocker or Critical Issues
    No New Bugs
    No New Code Smells

# CI Setup
Create pipeline
    paste my [text](netflix-CI.jenkinsfile)
    ensure that Vault secrets block correctly handles secrets
Set up SonarQube
        Create a manual project, named Petclinic
        In Account create a taken por general usage, add it to Vault and to Jenkins
        Configure a webhook to make Sonarqube give a feedback to Jenkins pipeline about the code quality gates; Use excactly the Jenkins url you have set up in jenkins system settings -> jenkins location

## Deploy Trivy
Trivy is an open source vulnerability scanner for containers. It’s designed to be simple, fast, and comprehensive. Trivy can scan container images and detect vulnerabilities in the OS packages and application dependencies.

Best practicies: 
  Scan container images regularly: It’s important to scan container images regularly to ensure that they’re free of vulnerabilities. You can set up Trivy to scan container images on a regular basis, such as daily or weekly.

  Use Trivy in combination with other security measures: Trivy is just one security measure for Kubernetes. It’s important to use Trivy in combination with other security measures, such as RBAC, network policies, and pod security policies.

  Store Trivy cache in a persistent volume: Trivy cache can become quite large over time, especially if you’re scanning a lot of container images. It’s recommended to store the Trivy cache in a persistent volume to avoid losing it when the Pod is deleted.

    In Application's CI pipeline we run:
    `trivy image --docker-host string  askdragon/petclinic:{buils number var}`

## Owasp dependency check
stage('OWASP FS SCAN') {
    steps {
        // Run OWASP Dependency-Check
        dependencyCheck additionalArguments: '--scan ./target --format TXT', odcInstallation: 'DP-check'
        
        // Publish the generated TXT report
        dependencyCheckPublisher pattern: '**/dependency-check-report.txt'
    }
}

Collecting Dependency-Check artifact

Parsing file /home/jenkins/agent/workspace/Scan/dependency-check-report.json

ERROR: Unable to parse /home/jenkins/agent/workspace/Scan/dependency-check-report.json

## Build Image with Kaniko
To be able to pull repositories that are private, create a special k8s secret and moun them to the pods:
``` kubectl create secret docker-registry kaniko-secret \
    --docker-server="https://index.docker.io/v1/" \
    --docker-username="<your username>" \
    --docker-password="<your password>" \
    --docker-email=<your email> \
    --namespace jenkins```  
This secret gets mounted to the kaniko pod for it to authenticate the Docker registry to push the built image.

We can make kaniko build the image in both ways:
1. Using args in manifest (in container specs)
2. Use in jenkins pipeline

Check out `jenkins-agents/kaniko-trivy-CI.jenkinsfile` to see how i exended jenkins agent manifestation and built a docker image.  

## Setup E-mail notification
documentation tbd
        
## Further configs   
Create a GitHub Webhook

Create a Webhook in your repository to trigger the Jenkins job on push. You may skip this step if you already have a Webhook configured.

    Go to the GitHub Webhook creation page for your repository and enter the following information:

        URL: Enter the following URL, replacing the values between *** as needed:

    ***JENKINS_SERVER_URL***/job/***JENKINS_JOB_NAME***/build?token=***JENKINS_BUILD_TRIGGER_TOKEN***

Under Which events would you like to trigger this webhook? select Let me select individual events and check the following:

    Pushes

Click Add webhook.
Sonarqube recommendarion: 
"node {
stage('SCM') {
    checkout scm
}
stage('SonarQube Analysis') {
    def mvn = tool 'Default Maven';
    withSonarQubeEnv() {
    sh "${mvn}/bin/mvn clean verify sonar:sonar -Dsonar.projectKey=Petclinic -Dsonar.projectName='Petclinic'"
    }
}
}"



Agent build-and-push-33-zbj0l-x6c6f-7gjm3 is provisioned from template Build_and_Push_33-zbj0l-x6c6f
---
apiVersion: "v1"
kind: "Pod"
metadata:
  name: "build-and-push-33-zbj0l-x6c6f-7gjm3"
  namespace: "jenkins"
spec:
  containers:
  - args:
    - "99d"
    command:
    - "sleep"
    image: "gcr.io/kaniko-project/executor:debug"
    imagePullPolicy: "Always"
    name: "kaniko"
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
    volumeMounts:
    - mountPath: "/kaniko/.docker"
      name: "kaniko-secret"
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
    workdir: "/home/jenkins/agent/"
  - command:
    - "trivy"
    - "server"
    - "--cache-dir"
    - "/data/cache"
    image: "aquasec/trivy"
    name: "trivy"
    volumeMounts:
    - mountPath: "/data/cache"
      name: "cache"
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
    image: "jenkins/inbound-agent:3309.v27b_9314fd1a_4-1"
    name: "jnlp"
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
  nodeSelector:
    kubernetes.io/os: "linux"
  restartPolicy: "Never"
  volumes:
  - name: "kaniko-secret"
    projected:
      sources:
      - secret:
          items:
          - key: ".dockerconfigjson"
            path: "config.json"
          name: "kaniko-secret"
  - name: "cache"
    persistentVolumeClaim:
      claimName: "trivy-pv-claim"
  - emptyDir:
      medium: ""
    name: "workspace-volume"
+ /kaniko/executor --dockerfile /home/jenkins/agent/workspace/Build_and_Push/Dockerfile --context /home/jenkins/agent/workspace/Build_and_Push '--destination=askdragon/petclinic:33'
[36mINFO[0m[0001] Resolved base name eclipse-temurin:17-jdk-alpine to runtime 
[36mINFO[0m[0001] Retrieving image manifest build              
[36mINFO[0m[0001] Retrieving image build from registry index.docker.io 
error building image: unable to complete operation after 0 attempts, last error: GET https://index.docker.io/v2/library/build/manifests/latest: UNAUTHORIZED: authentication required; [map[Action:pull Class: Name:library/build Type:repository]]
