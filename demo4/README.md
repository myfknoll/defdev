# Demo3: Mobile security framework!

Run the dependency-check container from the defdev docker registry ("defdev/dependency-check"). 
In the pipe-line-script file there is already a great part of the Jenkinsfile and how the container should run pre-defined.
However, in this demo we have to find out how to effectively run the Docker container.

### Dockerfile

    FROM java:8

    MAINTAINER Riccardo ten Cate <riccardo.ten.cate@owasp.org>

    ENV VERSION_URL="https://jeremylong.github.io/DependencyCheck/current.txt"
    ENV DOWNLOAD_BASEURL="https://dl.bintray.com/jeremy-long/owasp"

    COPY entrypoint.sh /

    RUN mkdir -p /opt/dependency-check \
     && wget -O /tmp/dependency-check-latest.zip "${DOWNLOAD_BASEURL}/dependency-check-$(wget -O - -o /dev/null "${VERSION_URL}")-release.zip" \
     && unzip /tmp/dependency-check-latest.zip -d /opt \
     && rm /tmp/dependency-check-latest.zip \
     && chmod +x /entrypoint.sh

    ENTRYPOINT ["/entrypoint.sh"]


### Entrypoint.sh

    #!/bin/bash

    set -e

    exit_env_error() {
        echo "Error: env var '${1}' not set" >&2
        exit 1
    }

    PROJECT_FOLDER="${PROJECT_FOLDER:-/project}"
    OUTPUT_FOLDER="${OUTPUT_FOLDER:-/project}"
    OUTPUT_FORMAT="${OUTPUT_FORMAT:-XML}"


    [ -z "${SOURCE_REPO}" ] && exit_env_error SOURCE_REPO
    [ -z "${DOJO_URL}" ] && exit_env_error DOJO_URL
    [ -z "${DOJO_ENGAGEMENT_ID}" ] && exit_env_error DOJO_ENGAGEMENT_ID
    [ -z "${DOJO_API_KEY}" ] && exit_env_error DOJO_API_KEY

    rm -rf "${PROJECT_FOLDER}" "${OUTPUT_FOLDER}"
    git clone "${SOURCE_REPO}" "${PROJECT_FOLDER}"

    mkdir -p "${OUTPUT_FOLDER}"

    /opt/dependency-check/bin/dependency-check.sh \
        --project "${PROJECT_FOLDER}" \
        --format "${OUTPUT_FORMAT}" \
        --out "${OUTPUT_FOLDER}" \
        --enableExperimental \
        --scan "${PROJECT_FOLDER}/**"

    cat "${OUTPUT_FOLDER}/dependency-check-report.xml"

    curl -k --request POST --url "${DOJO_URL}"/api/v1/importscan/ --header 'authorization: ApiKey '"${DOJO_API_KEY}"' ' --header 'cache-control: no-cache' --header 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' --form minimum_severity=Info --form scan_date=2018-05-01 --form verified=False --form file=@"${OUTPUT_FOLDER}"/dependency-check-report.xml --form tags=Test_automation --form active=True --form engagement=/api/v1/engagements/"${DOJO_ENGAGEMENT_ID}"/ --form 'scan_type=Dependency Check Scan'

