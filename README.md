
# Sonarqube + Dependency Check POC

Run [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) on a project and upload reports to [Sonarqube](https://www.sonarsource.com/products/sonarqube/).

> Dependency-Check is a Software Composition Analysis (SCA) tool that attempts
> to detect publicly disclosed vulnerabilities contained within a project’s dependencies.
> -- <cite>owasp.org</cite>

> SonarQube is a self-managed, automatic code review tool that systematically helps you deliver clean code. 
> -- <cite>docs.sonarsource.com</cite>

Requirements for running this POC:
- Docker

## Install and configure Sonarqube

```bash
docker pull sonarqube:10.1.0-community
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 sonarqube:10.1.0-community
```

Then open the GUI to configure Sonarqube: http://localhost:9000/
- change Sonarqube admin password
- install _dependency check_ plugin via the Sonarqube marketplace GUI (Sonarqube server will restart)
- create a project in Sonarqube, with an access token

## Setup Java, Maven and dev tooling 

A sample application `my-app`, with a vulnerability on `log4j-core`, is provided in `my-app` folder

To build the sample application, use the all-in-one Maven image from Docker:

```bash
docker pull maven:3.6.3-jdk-11-slim
```

Start a shell in a new container:

```bash
cd ./my-app
docker run --rm -it -v .:/root/src/my-app --link sonarqube --entrypoint bash maven:3.6.3-jdk-11-slim
```

The following Maven plugins will be used (already added in `pom.xml`):
- `sonar-maven-plugin` from group `org.sonarsource.scanner.maven`
- `dependency-check-maven` from group `org.owasp`

## Run Dependency-Check on the sample app

```bash
cd /root/src/my-app

mvn clean install
mvn dependency-check:check
```

The report is available locally in `target/dependency-check-report.html`

## Upload reports to Sonarqube

```bash
mvn verify sonar:sonar \
  -Dsonar.projectKey=test-api \
  -Dsonar.projectName='test api' \
  -Dsonar.host.url=http://sonarqube:9000 \
  -Dsonar.token=sqp_e28a16dbd21b082110cf54ae2c2c1a9b8e7136c0
```

Warning: the `sonar-maven-plugin` version has to be compatible with Sonarqube version
