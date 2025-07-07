# Create pipelines for Netflix app
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