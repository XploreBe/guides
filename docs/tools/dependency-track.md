# Dependency Track
Dependency Track is a tool that allows centralised monitoring of vulnerabilities imported through the dependencies of your applications. This tool bases his security analysis solely on a provided SBOM.

An SBOM (Software Bill of Materials) is a "list of ingrediënts" that make up your software. This list is compared to a set of CVE reports to analyse the included vulnerabilities and their severity.

## Why would you need an SBOM?
Supply chain attacks are on the rise. It is important that component providers also provide this format to ensure a certain level of security. The American government requires software suppliers to include an SBOM to ensure the safety and integrity of the components they are providing.

## Setting up Dependency-Track
The application itself has a straightforward and well-documented setup:
- Deployment through WAR (deprecated): https://docs.dependencytrack.org/getting-started/deploy-exewar/
- Deployment through a docker container: https://docs.dependencytrack.org/getting-started/deploy-docker/

The application will run 2 separate docker containers by default, frontend and backend separately. It's recommended that you also provide the application with its dedicated database.

I've set up a docker-compose script that will help you start out. It contains the frontend, backend and dedicated postgres DB: 

````
version: '3.7'

#####################################################
# This Docker Compose file contains three services
#    Dependency-Track API Server
#    Dependency-Track FrontEnd
#    Postgres Database
#####################################################

volumes:
  dependency-track:
  postgres:
services:
  dtrack-apiserver:
    image: dependencytrack/apiserver
    depends_on:
      - postgres
    environment:
      # The Dependency-Track container can be configured using any of the
      # available configuration properties defined in:
      # https://docs.dependencytrack.org/getting-started/configuration/
      # All properties are upper case with periods replaced by underscores.
      #
      # Database Properties
      - ALPINE_DATABASE_MODE=external
      - ALPINE_DATABASE_URL=jdbc:postgresql://10.3.61.1:5433/dtrack
      - ALPINE_DATABASE_DRIVER=org.postgresql.Driver
      - ALPINE_DATABASE_USERNAME=dtrack
      - ALPINE_DATABASE_PASSWORD=admin
      - ALPINE_DATABASE_POOL_ENABLED=true
      - ALPINE_DATABASE_POOL_MAX_SIZE=20
      - ALPINE_DATABASE_POOL_MIN_IDLE=10
      - ALPINE_DATABASE_POOL_IDLE_TIMEOUT=300000
      - ALPINE_DATABASE_POOL_MAX_LIFETIME=600000
    deploy:
      resources:
        limits:
          memory: 12288m
        reservations:
          memory: 8192m
      restart_policy:
        condition: on-failure
    ports:
      - '8081:8080'
    volumes:
      - 'dependency-track:/data'
    restart: unless-stopped
  dtrack-frontend:
    image: dependencytrack/frontend
    depends_on:
      - dtrack-apiserver
    environment:
    # The base URL of the API server.
    # NOTE:
    #   * This URL must be reachable by the browsers of your users.
    #   * The frontend container itself does NOT communicate with the API server directly, it just serves static files.
    #   * When deploying to dedicated servers, please use the external IP or domain of the API server.
    - API_BASE_URL=http://localhost:8081
    # - "OIDC_ISSUER="
    # - "OIDC_CLIENT_ID="
    # - "OIDC_SCOPE="
    # - "OIDC_FLOW="
    # - "OIDC_LOGIN_BUTTON_TEXT="
    # volumes:
    # - "/host/path/to/config.json:/app/static/config.json"
    ports:
      - "8080:8080"
    restart: unless-stopped
  postgres:
    image: postgres:latest
    environment:
      - POSTGRES_PASSWORD=admin
      - POSTGRES_USER=dtrack
    ports:
      - '5432:5432'
    volumes:
      - 'postgres:/data'
    restart: unless-stopped
````

## Usage
Dependency-Track contains several useful features that can be helpful in tracking vulnerabilities within your infrastructure.

### Dashboard
Grafana-like dashboard showing the general status of your applications. It indicates the total of vulnerable components, audited vulnerabilities, release overview of vulnerabilities, ...

### Projects
List overview of your existing applications and their analysed version. It gives a representation of the severity for each application. Each application is assigned a risk score along with the total amount of vulnernabilities. 

The risk score is a calculation based on severity and likeliness of all the detected vulnerabilities.

### Components
Useful list view that allows specific searches on vulnerabilities and components. Most common usage would be to check if you're vulnerable to a specific CVE.

### Vulnerabilities
Shows which vulnerabilities are present within your infrastructure. This interface also allows maintenance of these vulnerabilities, for example suppressing certain vulnernabilities

## Generating SBOMs
The generation of an SBOM can be implemented withing your build framework:
- Gradle: https://github.com/CycloneDX/cyclonedx-gradle-plugin
- Maven: https://github.com/CycloneDX/cyclonedx-maven-plugin

These plugins will generate an SBOM while which you can (automatically) upload into Dependency-Track. Jenkins has a dedicated plugin to automate the upload of the generated file: https://plugins.jenkins.io/dependency-track/

### Sources:
- https://docs.dependencytrack.org/
- https://www.cisa.gov/sbom
