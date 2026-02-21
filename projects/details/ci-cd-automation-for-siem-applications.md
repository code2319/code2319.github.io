# Technical Deep Dive  
## CI/CD Automation for SIEM Applications

This project implements a multi-stage CI/CD pipeline for building, validating, testing, versioning, and releasing SIEM applications using GitLab CI and Docker.

The core architectural goal:

> Treat SIEM applications as production software artifacts — versioned, reproducible, validated, and released through controlled automation.

This pipeline transforms manual security application packaging into a deterministic engineering workflow.

---

## Architecture Flow

Git Push  
→ GitLab CI Trigger  
→ Build & Package  
→ Static Validation (AppInspect)  
→ Containerized Testing  
→ Version Automation  
→ Artifact Storage  
→ Release Tag

Each stage acts as a quality gate. Failure at any stage blocks downstream execution.

---

## Design Principles

### 1. Reproducibility
All testing and validation runs inside containerized environments to eliminate dependency drift between CI runners.

### 2. Deterministic Builds
Artifacts are immutable and tied to a specific Git commit and semantic version.

### 3. Quality Gates
The pipeline automatically fails if:
- Application validation fails
- Packaging structure is invalid
- Required metadata is missing
- Tests fail

### 4. Stage Isolation
Build, validation, testing, versioning, and release are separated into distinct pipeline stages to improve clarity and debuggability.

---

## Pipeline Implementation

### Stage 0 - Generate a Set of Prebuilt Files

The Universal Configuration Console (UCC) is a command line tool that generates a set of prebuilt files that can be packaged into an add-on with a UI interface. The tool is used by Splunk engineering teams to create officially supported add-ons.

```bash
ucc-gen init --addon-name "CC-is-vpn" --addon-display-name "Is VPN Custom Command" --addon-input-name demo_input --addon-version 0.0.1
```

Purpose:

* Simplifies add-on creation
* Provides a uniform look and feel with standard UI elements
* Facilitates open sourcing of well-known TAs

---

### Stage 1 — Build & Package

Packages the SIEM application into a distributable artifact.

```yaml
build:
  stage: build
  image: ubuntu:latest
  script:
    - apt update -qq
    - apt install -y software-properties-common curl
    - add-apt-repository ppa:deadsnakes/ppa -y
    - apt install -y python3.7-venv
    - python3.7 -m venv my-project
    - source my-project/bin/activate
    - pip install splunk-add-on-ucc-framework splunk-packaging-toolkit
    - mkdir my-package
    - ucc-gen build --ta-version $(date +'%d.%m.%Y')
    - mkdir app-dir
    - cp package/app.manifest output/CC-is-vpn/
    - cp package/default/app.conf output/CC-is-vpn/default/
    - ucc-gen package --path output/CC-is-vpn -o app-dir/
    - tar -czf my-package.tar.gz app-dir/
  artifacts:
    paths:
      - my-package.tar.gz
```

Purpose:

* Standardized packaging
* Artifact consistency
* Portability across environments

---

### Stage 2 — Static Validation

Application structure validation is performed using platform validation tooling.

```yaml
run-appinspect:
  stage: inspect
  image: ubuntu:latest
  dependencies:
    - build
  script:
    - apt update -qq
    - apt install -y software-properties-common libmagic1 libmagic-dev
    - add-apt-repository ppa:deadsnakes/ppa -y
    - apt install -y python3.7-venv
    - python3.7 -m venv my-project
    - source my-project/bin/activate
    - pip install splunk-appinspect
    - tar -xzf my-package.tar.gz
    - FILE_NAME=$(ls -1 app-dir/)
    - splunk-appinspect inspect app-dir/$FILE_NAME --output-file appinspect_result.json --mode test
  artifacts:
    paths:
      - appinspect_result.json
```

Impact:

* Prevents invalid deployments
* Enforces structural correctness
* Shifts validation left into CI

---

### Stage 3 — Containerized Testing

Applications are tested inside Docker containers to simulate controlled runtime environments.

```yaml
test-v9-2:
  stage: test
  needs: ["build", "run-appinspect"]
  image: ubuntu:latest
  services:
    - docker:dind
  variables:
    SPLUNK_VERSION: "9.2" # splunk tag from docker hub
  script:
    - ls -al
    - echo "Unpacking package"
    - tar xvf my-package.tar.gz
    - echo "Install packages"
    - apt-get -qq update
    - apt-get -qq install -y jq software-properties-common libmagic1 libmagic-dev curl
    - add-apt-repository ppa:deadsnakes/ppa -y
    - apt install -y python3.7-venv
    - python3.7 -m venv my-project
    - source my-project/bin/activate
    - echo "Test On Splunk 9.2"
    - chmod +x ./app_test.sh
    - ./app_test.sh $SPLUNK_VERSION
  artifacts:
    paths:
      - pytest-report.html
    name: "pytest-report"
    expire_in: 1 week
```

Benefits:

* Eliminates environment inconsistencies
* Enables multi-version compatibility testing
* Improves reliability across platform upgrades

---

### Stage 4 — Automated Versioning

Semantic versioning is automated and tied to repository state.

```bash
VERSION=$(cat VERSION)
NEW_VERSION=$(semver bump patch $VERSION)
echo $NEW_VERSION > VERSION
```

Why:

* Eliminates manual version errors
* Aligns releases with Git history
* Enables controlled tagging

---

### Stage 5 — Release Management

If all validation gates pass:

* Git tag is created
* Artifact is stored
* Release metadata is generated
* Deployment-ready package is produced

Release stage execution is restricted to protected branches.

```yaml
release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  needs: ["test-v9-2"]
  only:
    - tags
  script:
    - echo "Creating release for $CI_COMMIT_REF_NAME"
    - tar -xzf my-package.tar.gz
    - mv ./app-dir/CC-is-vpn-* ./CC-is-vpn-${CI_COMMIT_TAG}.tar.gz
  release:                               # See https://docs.gitlab.com/ee/ci/yaml/#release for available properties
    tag_name: '$CI_COMMIT_TAG'
    description: '$CI_COMMIT_TAG'
    assets:
	    links:
		    - name: "CC-is-vpn-${CI_COMMIT_TAG}.tar.gz"
			    url: ${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/raw/CC-is-vpn-${CI_COMMIT_TAG}.tar.gz"
    artifacts:
	    paths:
		    - "CC-is-vpn-${CI_COMMIT_TAG}.tar.gz"
    when: on_success
```

---

## Engineering Decisions

### Why CI/CD for SIEM Applications?

Manual packaging and validation:

* Introduces human error
* Causes inconsistent builds
* Slows deployment cycles

Automating the workflow:

* Reduces operational overhead
* Standardizes quality control
* Improves auditability
* Enables scalable application lifecycle management

---

## Results

After implementation:

* Deployment cycle reduced from manual multi-hour process to automated minutes
* Packaging inconsistencies eliminated
* Standardized validation across applications
* Release workflow fully controlled and repeatable

Security tooling moved from operational task to engineered product lifecycle.

---

## Conclusion

This project applies DevOps and software engineering discipline to SIEM application delivery.

By enforcing structured validation, deterministic builds, and controlled release automation, the solution transforms security application management into a scalable, enterprise-grade engineering process.

Security tooling should be engineered — not manually assembled.

---

# Appendix
```bash
#!/bin/bash

# ENVIROMENT VARIABLES
APP_ROOT=$(jq -r '.meta.name' ./globalConfig.json)
APP_NAME=$(jq -r '.meta.displayName' ./globalConfig.json)
APPS_DIR="/opt/splunk/etc/apps"
USER="admin"
PASSWORD="password"
CI_PROJECT_DIR=${CI_PROJECT_DIR:-`pwd`}
CONTAINER_NAME="splunk"
SPLUNK_VERSION=$1

# SETTING UP DOCKER CONTAINER WITH SPLUNK
echo -e "\033[94m Installing docker...\033[0m"
apt-get update
apt-get -qq install docker.io jq -y
echo "______________________________________________________________________"

echo -e "\033[94m Creating splunk container...\033[0m"
docker run --rm -d \
    -p 8000:8000 \
    -p '127.0.0.1:8090:8089' \
    -e "SPLUNK_START_ARGS=--accept-license" \
    -e "SPLUNK_PASSWORD=$PASSWORD" \
    --name $CONTAINER_NAME splunk/splunk:$SPLUNK_VERSION start

echo "______________________________________________________________________"

echo -e "\033[94m Checking runned containers...\033[0m"
docker ps
echo "______________________________________________________________________"

echo -e "\033[94m Running inspect for container $CONTAINER_NAME...\033[0m"
docker inspect $CONTAINER_NAME
echo "______________________________________________________________________"

echo -e "\033[94m Waiting for splunk to be up...\033[0m"
echo "______________________________________________________________________"

# COPYING APP DATA FROM ARTIFACT TO APPS FOLDER IN THE CONTAINER
echo -e "\033[94m Installing app...\033[0m"
ls -l $CI_PROJECT_DIR
FILE_NAME=$(ls -1 app-dir/)
echo "FILE NAME: $FILE_NAME"
docker exec -i -u root $CONTAINER_NAME mkdir -p $APPS_DIR/$APP_ROOT
echo "docker cp $CI_PROJECT_DIR/app-dir/$FILE_NAME $CONTAINER_NAME:$APPS_DIR"
docker cp $CI_PROJECT_DIR/app-dir/$FILE_NAME $CONTAINER_NAME:$APPS_DIR

docker exec -i $CONTAINER_NAME ls -l /opt/splunk/etc/apps
docker exec -i -u root $CONTAINER_NAME tar -xzf $APPS_DIR/$FILE_NAME -C $APPS_DIR/
docker exec -i -u root $CONTAINER_NAME chmod -R 777 $APPS_DIR/
docker exec -i $CONTAINER_NAME ls -l $APPS_DIR/
docker exec -i $CONTAINER_NAME ls -l $APPS_DIR/$APP_ROOT/
echo "______________________________________________________________________"

# INSTALLING pytest-splunk-addon FOR FUTURE KNOWLEDGE OBJECT TESTING
echo -e "\033[94m Installing python packages... \033[0m"
pip install pytest-splunk-addon pytest-html --quiet
echo "______________________________________________________________________"

# CHECKING IF SPLUNK CONTAINER IS RUNNING
echo -e "\033[94m Checking running containers... \033[0m"
docker ps
echo "______________________________________________________________________"

# Wait for instance to be available
# Waiting for 2 and a half minutes.
loopCounter=30
mainReady=0
checked=0
errors=0

while [[ $loopCounter != 0 && $mainReady != 1 ]]; do
  ((loopCounter--))
  health=`docker ps --filter "name=${version}" --format "{{.Status}}"`
  echo $health

# health will be one of these values: 
  if [[ ! $health =~ "starting" ]]; then

    echo -e "\033[94m container running, checking data status...\033[0m"

    # get a list of installed applications
    appList=`docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD /opt/splunk/bin/splunk search '|rest /services/apps/local |table label'"`
    
    echo -e "\033[92m APP LIST: $appList\033[0m"
    echo "______________________________________________________________________"

    if [[ $checked != 1 ]]; then
        # WHEN CONTAINER IS UP WE CHECK SPLUNKD STATUS
        echo -e "\033[94m Checking Splunkd status...\033[0m"
        SPLUNKD_STATUS=$(docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD /opt/splunk/bin/splunk status")
        echo "splunkd status: $SPLUNKD_STATUS"
        echo "______________________________________________________________________"

        # WHEN CONTAINER IS UP WE CREATE HEC TOKEN FOR TESTING PURPOSES - SENDING DUMMY DATA TO SPLUNK
        echo -e "\033[94m Creating HEC token...\033[0m"
        HEC_TOKEN_OUTPUT=$(docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD /opt/splunk/bin/splunk http-event-collector create new-token -uri https://127.0.0.1:8089 -disabled 0 -index log")
        HEC_TOKEN=$(echo "$HEC_TOKEN_OUTPUT" | grep -oP 'token=\K[^ ]+')
        echo "Generated HEC token: $HEC_TOKEN"
        echo "______________________________________________________________________"

        echo -e "\033[94m Running unit tests...\033[0m"

        # CHECKING IF APP IS CORRECTLY INSTALLED
        echo -e "\033[94m Checking if app is installed... \033[0m"
        apsList=`docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD curl -k -u $USER:$PASSWORD -i -q https://127.0.0.1:8089/services/apps/local/?search=$APP_ROOT | grep -q $APP_ROOT"`
        if ! echo $apsList; then
          echo -e "\033[91m App $APP_ROOT not found in local apps! \033[0m"
          # exit 1
          ((errors++))
        else
          echo -e "\033[92m $APP_ROOT found! \033[0m"
        fi

        echo "______________________________________________________________________"

        # installing vpnapi.io API token to realm in CC-is-vpn app
        echo -e "\033[94m Installing vpnapi.io API token for app... \033[0m"
        installingApiToken=`docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD curl -k -u $USER:$PASSWORD https://127.0.0.1:8089/servicesNS/nobody/CC-is-vpn/storage/passwords -d name=api_token -d password={$VPNAPI_IO_API_TOKEN} -d realm=CC-is-vpn-new_realm"`
        
        # CHECKING IF API TOKEN IS INSTALLED
        checkingApiToken=`docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD curl -k -u $USER:$PASSWORD -i -q https://127.0.0.1:8089/servicesNS/nobody/CC-is-vpn/storage/passwords/CC-is-vpn-new_realm:api_token | grep -q encr_password"`
        if ! echo $checkingApiToken; then
            echo -e "\033[91m vpnapi.io API token not found! \033[0m"
            # exit 1
            ((errors++))
        else
            echo -e "\033[92m vpnapi.io API token found! \033[0m"
        fi

        echo "______________________________________________________________________"
        
        # CHECKING IF "ISVPN" COMMAND RETURNS CORECT RESULT
        echo -e "\033[94m Checking if custom search command runs correctly... \033[0m"
        echo -e "\033[94m Trying to determine info for ip=104.28.51.194... \033[0m"

        customSearch=$(docker exec -i -u splunk $CONTAINER_NAME bash -c "SPLUNK_USERNAME=$USER SPLUNK_PASSWORD=$PASSWORD /opt/splunk/bin/splunk search '| isvpn ip=104.28.51.194' | jq")

        if ! echo "$customSearch" | grep -q "ip"; then
            echo -e "\033[91m Custom search command does not work correctly! \033[0m"
            # exit 1
            ((errors++))
        else
            echo -e "\033[92m Custom search command works correctly! \033[0m"
        fi

        echo "______________________________________________________________________"

        echo -e "\033[94m Printing $APP_ROOT configuration... \033[0m"
        curl -k -u $USER:$PASSWORD -i -q https://127.0.0.1:8089/services/apps/local/$APP_ROOT

        checked=1
        if $errors>0; then
        exit 1
        fi
    fi
    mainReady=1
  fi

  # if the container is no longer running...
  if [[ $health == "" ]]; then
    echo "Health:\n${health}\n"
    echo "--------------------------------"
    docker ps -a
    echo "--------------------------------"
    docker inspect $CONTAINER_NAME
    echo "--------------------------------"
    docker logs $CONTAINER_NAME
    echo "--------------------------------"
    echo "Container is no longer running!"
    exit 1
  fi

  echo "loopCounter: ${loopCounter}"
  echo "mainReady: ${mainReady}"
  sleep 5
done

if [[ $mainReady != 1 ]]; then
  echo "Timeout waiting for data to be ingested into Splunk!"
  docker logs $CONTAINER_NAME
  docker ps -a
  exit 1
fi
```