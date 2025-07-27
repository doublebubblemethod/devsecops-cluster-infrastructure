1. Jenkins Plugins You Need for CI Pipeline
    Git Plugin: To pull your code from your GitHub repository or any other Git repository.

    Maven Plugin: Since Spring PetClinic is a Maven-based project, this plugin is essential for building the application.

    SonarQube Scanner Plugin: For static code analysis using SonarQube.

    Pipeline Plugin: To define your CI/CD pipeline as code using a Jenkinsfile (this is essential if you're using declarative pipeline syntax).

    Docker Pipeline Plugin: If you plan to build Docker images as part of your CI pipeline, this plugin is necessary.

    Kubernetes Plugin: If you're deploying parts of your pipeline or interacting with Kubernetes from Jenkins, this is helpful.

    Blue Ocean Plugin: A nicer UI for Jenkins, especially when working with pipelines.

    JUnit Plugin: To publish test results (if your tests are run using JUnit).

    Email Notification Plugin: For sending notifications when your pipeline passes/fails.

2. Database for the Application (MySQL/PostgreSQL)
    Deploying the Database in Kubernetes. You should create a Kubernetes Pod or Deployment for the database (either MySQL or PostgreSQL). 

3. Create Approle role and policy in Vault and add this secret in jenkins credential menager
4. Setup Jenkins Build Agents on Kubernetes Pods using your home-made jenkins-inbound-agent docker image with vault CA cert and mvn tool
5. In sonarqube create a token and copy generated mvn command:
