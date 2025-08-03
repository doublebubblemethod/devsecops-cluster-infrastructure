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

    Clean Workspace: Clean the workspace before starting.
    Checkout from Git: Pull the latest code from your Git repository.
    Static Analysis: Run code analysis using SonarQube.
    Quality Gate: Check if the code meets quality standards and Code Coverage ≥ 80%
    Install Dependencies: Install required dependencies like npm install.


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


 ERRORS efd4008b-16ce-6c99-46b9-d1c4889aa81f
 
+ /home/jenkins/agent/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar-scanner/bin/sonar-scanner -Dsonar.projectName=Petclinc -Dsonar.projectKey=Petclinc
12:29:32.832 INFO  Scanner configuration file: /home/jenkins/agent/tools/hudson.plugins.sonar.SonarRunnerInstallation/sonar-scanner/conf/sonar-scanner.properties
12:29:32.837 INFO  Project root configuration file: NONE
12:29:32.851 INFO  SonarScanner CLI 7.1.0.4889
12:29:32.853 INFO  Java 17.0.8.1 Eclipse Adoptium (64-bit)
12:29:32.853 INFO  Linux 6.11.0-29-generic amd64
12:29:32.885 INFO  User cache: /home/jenkins/.sonar/cache
12:29:38.569 ERROR Failed to query server version: Call to URL [https://sonar-service.sonar.svc.cluster.local/api/v2/analysis/version] failed: Connect timed out
12:29:38.569 INFO  EXECUTION FAILURE
12:29:38.571 INFO  Total time: 5.774s

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
