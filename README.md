# Sample .Net WebApi with Sonarqube & Trivy

#### The purpose of this project is to demonstrate vulnerability scans in the code and docker image using sonarqube and trivy.

## CI/CD Pipeline Overview

- **build**: Restores dependencies and builds the project.
- **test**: Runs unit tests.
- **sonar**: Runs SonarQube analysis using SonarScanner for MSBuild.
- **security**: Scans source code and Docker images for vulnerabilities using Trivy.

## Customizing for Any Project

To adapt this pipeline for your own .NET project:

1. Update the project paths in the `dotnet build` and `dotnet test` commands to match your solution or project files.
2. Change the SonarQube project key and organization in the `sonar` stage:
   - `/k:"your_project_key"`
   - `/o:"your_organization"`
3. Adjust Trivy scan paths and Docker image names as needed.
4. Add or remove stages as required for your workflow.

## Setting Up SonarQube Cloud and Integrating with GitLab

### 1. Create a SonarQube Cloud Account

- Go to [SonarCloud](https://sonarcloud.io/) and sign up using your GitLab account or email.
- Create a new organization and project.

### 2. Generate a SonarQube Token

- In SonarCloud, go to **My Account > Security**.
- Generate a new token and copy it.

### 3. Add SonarQube Token to GitLab CI/CD

- In your GitLab project, go to **Settings > CI/CD > Variables**.

- The pipeline uses the token via `/d:sonar.login="$SONAR_TOKEN"`.
- Set your project key and organization in the `dotnet sonarscanner begin` command.

### 5. Example SonarQube Stage

```yaml
sonar:
  stage: sonar
  image: mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}
  script:
    - dotnet tool install --global dotnet-sonarscanner
    - export PATH="$PATH:$HOME/.dotnet/tools"
    - dotnet sonarscanner begin /k:"your_project_key" /o:"your_organization" /d:sonar.login="$SONAR_TOKEN"
    - dotnet build ./src/YourProject/YourProject.csproj -c Release
    - dotnet sonarscanner end /d:sonar.login="$SONAR_TOKEN"
```

## Integrating GitLab with SonarQube Cloud

- SonarCloud can automatically detect pull requests and provide code quality feedback in GitLab merge requests.
- Ensure your project is linked to GitLab in SonarCloud settings.
- Use the correct project key and organization for analysis.

## Adding Secrets in GitLab

To securely add secrets (tokens, passwords, API keys) for use in your pipeline:

1. Go to your GitLab project.
2. Navigate to **Settings > CI/CD > Variables**.
3. Click **Add Variable**.
4. Enter the variable name (e.g., `SONAR_TOKEN`, `DOCKER_PASSWORD`).
5. Enter the value and mark as **Protected** or **Masked** if needed.
6. Reference the variable in your `.gitlab-ci.yml` using `$VARIABLE_NAME`.

## Trivy Security Scanning

- Trivy is installed and run in the `security` stage.
- It scans both the source code and the built Docker image for vulnerabilities.
- You can customize severity levels and scan targets in the script section.

### Communicating Non-Exploitable Vulnerabilities with VEX

If you're an OSS maintainer and Trivy has detected vulnerabilities in your project that you believe are not actually exploitable, consider issuing a VEX (Vulnerability Exploitability eXchange) statement.

VEX allows you to communicate the actual status of vulnerabilities in your project, improving security transparency and reducing false positives for your users.

#### How to Issue a VEX Statement

1. Create a VEX document in [CycloneDX](https://cyclonedx.org/docs/1.4/json/) or [OpenVEX](https://openvex.dev/) format. This document should specify which vulnerabilities are not exploitable and why.
2. Publish the VEX document in your repository (e.g., as `vex.json` or `vex.xml`).
3. Reference the VEX document in your project documentation or security advisories.
4. Users and security tools (including Trivy) can use this VEX document to filter out non-exploitable vulnerabilities.

#### Example VEX Document (CycloneDX JSON)

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "version": 1,
  "vulnerabilities": [
    {
      "id": "CVE-2023-12345",
      "analysis": {
        "state": "not_affected",
        "justification": "Component is not used at runtime."
      }
    }
  ]
}
```

#### Resources

- [CycloneDX VEX Documentation](https://cyclonedx.org/docs/1.4/json/)
- [OpenVEX Documentation](https://openvex.dev/)
- [Trivy VEX Support](https://aquasecurity.github.io/trivy/latest/docs/vex/)

### Example Trivy Usage

```yaml
trivy:
  stage: security
  image: docker:latest
  services:
    - name: docker:dind
      command: ["--host=tcp://0.0.0.0:2375", "--tls=false"]
  before_script:
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
    - export PATH="$PATH:./bin"
  script:
    - ./bin/trivy fs --exit-code 1 --severity CRITICAL,HIGH ./src
    - docker build -t your-image:latest .
    - ./bin/trivy image --exit-code 1 --severity CRITICAL,HIGH your-image:latest
```

## Troubleshooting

- Ensure all paths and variable names are correct.
- Remove unsupported files like `sonar-project.properties` for SonarScanner for MSBuild.
- Make sure Trivy is run from the correct path (`./bin/trivy`).
- Set executable permissions if needed: `chmod +x ./bin/trivy`.

## References

- [SonarCloud Documentation](https://docs.sonarcloud.io/)
- [GitLab CI/CD Variables](https://docs.gitlab.com/ee/ci/variables/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
