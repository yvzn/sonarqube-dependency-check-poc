
# SonarQube + Dependency Check POC

The goal is to run [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) on a given project and display the reports to [SonarQube](https://www.sonarsource.com/products/sonarqube/).

> Dependency-Check is a Software Composition Analysis (SCA) tool that attempts
> to detect publicly disclosed vulnerabilities contained within a project’s dependencies.
> -- <cite>owasp.org</cite>

By default, Dependency-Check only produces reports on the local machine (or CI/CD agent): enters SonarQube.

> SonarQube is a self-managed, automatic code review tool that systematically helps you deliver clean code. 
> -- <cite>docs.sonarsource.com</cite>

SonarQube performs quality reviews on the source code itself and display the results in a central server. Dependency-Check reports are a nice addition to the code quality reviews provided by SonarQube.

This POC uses [Apache Maven](https://maven.apache.org/) as a build tool to orchestrate Dependency-Check and SonarQube reports. Docker is used for convenience and faster setup.

Requirements:
- Docker

## Install and configure SonarQube

```bash
docker pull sonarqube:10.1.0-community
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube:10.1.0-community
```

Then open the GUI to configure SonarQube: http://localhost:9000/
- change SonarQube admin password
- install _dependency check_ plugin via the SonarQube marketplace GUI (SonarQube server will restart)
- create a project in SonarQube, with an access token

## Setup Java and Maven

A sample application `my-app`, with a vulnerability on `log4j-core`, is provided in `my-app` folder

To build the sample application, use the all-in-one Maven image from Docker:

```bash
docker pull maven:3.6.3-jdk-11-slim
```

Start a shell in a new container, mounting the sample application at `/root/src/my-app` and linking to the previously created `sonarqube` container:

```bash
cd ./my-app
docker run --rm -it -v .:/root/src/my-app --link sonarqube --entrypoint bash maven:3.6.3-jdk-11-slim
```

Build the sample application:

```bash
cd /root/src/my-app
mvn clean install
```

The following Maven plugins will be used to interact with Dependency-Check and SonarQube:
- Dependency-Check: `dependency-check-maven` from group `org.owasp`
- SonarQube: `sonar-maven-plugin` from group `org.sonarsource.scanner.maven`

## Run Dependency-Check on the sample app

Add `dependency-check-maven` to Maven plugins (`build`):
- see sample application for reference

Run Dependency-Check:

```bash
cd /root/src/my-app
mvn dependency-check:check
```

The reports are available locally in `target/dependency-check-report.*`

## Run SonarQube review

Add `sonar-maven-plugin` to Maven plugins (`build`):
- see sample application for reference

Define the following Maven properties:
- `sonar.dependencyCheck.htmlReportPath`: path to Dependency-Check report (HTML)
- `sonar.dependencyCheck.jsonReportPath`: path to Dependency-Check report (JSON)
- see sample application for reference

Run SonarQube plugin:
- performs the code review
- uploads the review to central SonarQube server
- uploads the Dependency-Check reports to central SonarQube server

```bash
mvn verify sonar:sonar \
  -Dsonar.projectKey=<<project key>> \
  -Dsonar.projectName='<<project name>>' \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.token=<<project access token>>
```

Warning: the `sonar-maven-plugin` version has to be compatible with SonarQube version
