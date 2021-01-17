---
title: "Shell Script for Automating Development Setup"
date: 2021-01-17T14:56:17Z
draft: false
tags: [shell, unix, workflow]
---

## Introduction

A common frustration I have when working on side projects is that it often takes some time to set up the development environment. Every time I want to make a change it takes a few minutes to open editors and start all the services. Often I work in 30 minute to hour long stints so I don't like wasting any time. Not alone that but after some time away from a project it can be difficult to remember all the steps to get it running. This became such an inconvenience that I decided to automate this so that I can spend more time on development. The goal is for it to take less than one minute before I can begin working on the project.

The project that will be used as an example is a standard web application with the usual three tiers: client, REST API, and database. On top of this, there is also a Keycloak service for handling authorization and authentication.

In production, all of these services could be orchestrated using Kubernetes or Docker Swarm. However, for development, each of these services are run differently. The client is an Angular project and is run using `ng serve`, the REST API is a Spring Boot Gradle project and ran with `./gradlew bootRun`, the database is PostgreSQL and is a systemd service, and finally the Keycloak service is a Docker container. There are also two different editors: VS Code for the client and IntelliJ IDEA for the REST API.

## Design Considerations

Initially, when deciding what to use to automate this, it was between a shell script and Ansible. These were the advantages I see with both approaches.

Shell:
+ Less complex
+ Fewer dependencies

Ansible:
+ Idempotency built-in
+ OS independent

Ultimately, after considering my requirements, I opted for the shell script since I will be the only one using it and I wanted something as convenient as possible to run and maintain. However, that means we will have to build in some level of idempotency ourselves so, for example, if we encounter an issue while running the script we don't have to worry about tidying up the state of our environment before rerunning it. Or we should at least log the state so that it will be easy to debug any issues. To achieve this we will implement the below pseudocode when starting each service where we check if it is already running.

```txt
for each service
    if service is not running
        start service
    else
        print a message

for each service
    if service had to be started
        wait for service to be up
        print a message
```

We start all of the services before waiting to check if they are running. This way they can start in parallel rather than sequentially so we only have to wait for the service which takes the longest to start rather than the sum all the startup times. Although there may be some overhead is starting them all at once.

There is also a strict directory structure required where all repositories are the direct children of the same parent directory as shown below.

```txt
parent-directory
    |_ keycloak-repo
    |   |_ realm-export.json
    |   |_ ...
    |_ rest-api-repo
    |   |_ ...
    |_ web-client-repo
    |   |_ ...
    |_ my-workspace.code-workspace
```

## The Script

The first part of the script we set the default values for variables, parse named arguments and set our constants. The named arguments are optional but I rather them over positional arguments. The loop for parsing named arguments was taken directly from the [Unix Stack Exchange](https://unix.stackexchange.com/a/353639).

```shell
#!/bin/sh
echo "$@"

# Parse named arguments
ROOT_PATH=/home
OPEN_EDITORS="true"

for ARGUMENT in "$@"
do
    KEY=$(echo ${ARGUMENT} | cut -f1 -d=)
    VALUE=$(echo ${ARGUMENT} | cut -f2 -d=)   

    case "$KEY" in
        ROOT_PATH)          ROOT_PATH=${VALUE} ;;
        OPEN_EDITORS)       OPEN_EDITORS=${VALUE} ;;     
        *)
    esac
done

readonly KEYCLOAK_REPO_PATH=${ROOT_PATH}/social-care-keycloak
readonly REST_API_REPO_PATH=${ROOT_PATH}/platform-rest-api
readonly WEB_CLIENT_REPO_PATH=${ROOT_PATH}/platform-web-client
readonly IDEA_BIN_PATH=/snap/intellij-idea-community/current/bin
readonly KEYCLOAK_DOCKER_NAME=keycloak
readonly RQUIRED_PACKAGES="curl jq"
```

The script requires some packages which will need to be installed if they are not already. The assumption is that the developer already has the tools to run the project installed, for example, Docker and Angular, so there's no point overcomplicating the script with those. Again, this section was taken straight from [Stack Overflow](https://stackoverflow.com/a/10439058/3987401).

```shell
# Packages
echo "Installing required packages..."
for i in ${RQUIRED_PACKAGES}; do
    PKG_OK=$(dpkg-query -W --showformat='${Status}\n' ${i} | grep "install ok installed")
    echo "Checking for ${i}: ${PKG_OK}"
    if [ "" = "${PKG_OK}" ]; then
        echo "+ Installing $i..."
        sudo apt -y install $i
    fi
done
```

The next step is to check if all the repositories are cloned to the correct location. If any repository is missing a message is printed and the script is terminated.

```shell
# Repositories
echo "Checking if repositories exist..."
for i in ${KEYCLOAK_REPO_PATH} ${REST_API_REPO_PATH} ${WEB_CLIENT_REPO_PATH}; do
    if [ -d ${i} ]
    then
        echo "The directory ${i} exists."
    else
        echo "The directory ${i} does not exist..."
        echo "Please clone the repository to the correct location..."
        echo "Exiting!"
        exit 0
    fi
done
```

We optionally open our editors if `OPEN_EDITORS` is set to true. The editors are optional since sometimes we just want to demo the project. As mentioned we'll open VS Code and IntelliJ IDEA. However, we won't bother with the same amount of checks and logging for the editors since we don't care what state they're in as long as they're running once the script finishes. For example, if IDEA is already running and we try to start it again the new process will simply be terminated. Also, the editors we use may change over time. 

```shell
# Editors
if [ ${OPEN_EDITORS} = "true" ]
then
    echo "Opening editors..."
    gnome-terminal --title="idea" --working-directory=${IDEA_BIN_PATH} --tab -- sh idea.sh
    code ${ROOT_PATH}/team-social-care.code-workspace
fi
```

PostgreSQL is the first service that we will start. We use cURL to check the response code of an HTTP GET request to the login page and store it for later. If the response is not a 200 OK then we need to start PostgreSQL. The other services will follow this pattern.

```shell
# PostgreSQL
POSTGRESQL_INITIAL_HTTP_CODE=$(curl -s -o /dev/null -w ''%{http_code}'' localhost/pgadmin4/login)

if [ ${POSTGRESQL_INITIAL_HTTP_CODE} -ne 200 ]
then
    echo "Starting PostgreSQL..."
    systemctl start postgresql
    echo "PostgreSQL has started."
else
    echo "PostgreSQL is already running."
fi
```

The next service we need to start is the Keycloak docker container. It may be easier to just start a new Docker container every time. We could use the below command similar to the one below to remove the container if it exists and then run a new one.

```shell
docker ps -a -q --filter "name=${KEYCLOAK_DOCKER_NAME}" | grep -q . && docker stop ${KEYCLOAK_DOCKER_NAME} > /dev/null 2>&1 && docker rm -fv ${KEYCLOAK_DOCKER_NAME} > /dev/null 2>&1
```

They are ephemeral after all. However, containers do take time to start up and there may already be test users already registered in an existing Keycloak container which we want to use. Therefore, we first check if a container with the name `keycloak` is running. If there isn't then check if a `keycloak` container has been stopped. If it's stopped then simply restart it. Otherwise, start a new Keycloak container. Again we store the initial state for later.

```shell
# Keycloak
KEYCLOAK_INITIAL_CONTAINER_CHECK=$(docker ps -q --filter "name=${KEYCLOAK_DOCKER_NAME}")

if [ ${KEYCLOAK_INITIAL_CONTAINER_CHECK} ]
then
    echo "Keycloak is already running."
else
    if [ $(docker ps -a -q --filter "name=${KEYCLOAK_DOCKER_NAME}") ]
    then
        echo "Starting existing Keycloak..."
        gnome-terminal --title="keycloak" --working-directory=${KEYCLOAK_REPO_PATH} --tab -- sh -c "docker start ${KEYCLOAK_DOCKER_NAME}"
    else
        echo "Starting Keycloak..."
        gnome-terminal --title="keycloak" --working-directory=${KEYCLOAK_REPO_PATH} --tab -- sh -c "docker run -p 8081:8080 -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin --name ${KEYCLOAK_DOCKER_NAME} jboss/keycloak"
    fi

    echo "Keycloak has started."
fi
```

The REST API is next and is pretty straightforward. We check if port `8080` is already in use. If it is then print a message and carry on since we assume the REST API is already running, although this may not be the case. If it is not in use then start the REST API.

```shell
# REST API
REST_API_INITIAL_PORT_CHECK=$(lsof -Pi :8080 -sTCP:LISTEN -t)

if [ ${REST_API_INITIAL_PORT_CHECK} ]
then
    echo "Port 8080 is in use..."
    echo "REST API may already be running..."
    echo "If not, kill process on port and retry."
else
    echo "Starting REST API..."
    gnome-terminal --title="rest-api" --working-directory=${REST_API_REPO_PATH} --tab -- sh -c './gradlew bootRun'
    echo "REST API has started."
fi
```

The client is almost identical to the REST API where we check if the port is in use first before running the angular project.

```shell
# Web client
WEB_CLIENT_INITIAL_PORT_CHECK=$(lsof -Pi :4200 -sTCP:LISTEN -t)

if [ ${WEB_CLIENT_INITIAL_PORT_CHECK} ]
then
    echo "Port 4200 is in use..."
    echo "Web client may already be running..."
    echo "If not, kill process on port and retry."
else
    echo "Starting web client..."
    gnome-terminal --title="web-client" --working-directory=${WEB_CLIENT_REPO_PATH} --tab -- sh -c 'ng serve'
    echo "Web client has started."
fi
```

Now we have to wait for the services that were started to be running. We check which ones were started using the value of the initial check we stored earlier for each service. If they had to be started we can wait for them to be up. To do this we define a function that will continue to check the response code for a given URL until it gets a 200 OK. We could wait for all services regardless if they were already running or not but, for the sake of better logging, we want to determine which ones were started.

```shell
# Health check
wait_for_response() {
    while [ "$(curl -s -o /dev/null -w ''%{http_code}'' ${1})" -ne "200" ];
    do
        echo "Waiting for $1..."
        sleep 5;
    done
}

if [ ${POSTGRESQL_INITIAL_HTTP_CODE} -ne 200 ]
then
    wait_for_response "localhost/pgadmin4/login"
    echo "PostgreSQL is now running."
fi

if [ -z ${KEYCLOAK_INITIAL_CONTAINER_CHECK} ]
then
    wait_for_response "localhost:8081"
    echo "Keycloak is now running."
fi

if [ -z ${REST_API_INITIAL_PORT_CHECK} ]
then
    wait_for_response "localhost:8080/swagger-ui/"
    echo "REST API is now running."
fi

if [ -z ${WEB_CLIENT_INITIAL_PORT_CHECK} ]
then
    wait_for_response "localhost:4200"
    echo "Web client is now running."
fi
```

The last step is to do any post start up tasks. We only have two. The first is to import our custom realm into Keycloak which involves using cURL to get an access token for our admin user and using that to send a realm JSON file to Keycloaks REST API now that the container is up. If an existing Keycloak container was started it may already have the realm imported so we may not get a 200 response but it should not cause any issues. The second is to open the URL for the Angular application in a browser.

```shell
# Post start up
echo "Importing realm settings..."
access_token=$(curl -s -X POST http://localhost:8081/auth/realms/master/protocol/openid-connect/token -H 'Content-Type: application/x-www-form-urlencoded' -d 'grant_type=password' -d 'client_id=admin-cli' -d 'username=admin' -d 'password=admin' | jq --raw-output '.access_token')
import_realm_response=$(curl -s -o /dev/null -w ''%{http_code}'' -X POST http://localhost:8081/auth/admin/realms -H "Authorization: bearer ${access_token}" -H 'Content-Type: application/json' -d "@${ROOT_PATH}/social-care-keycloak/realm-export.json")
echo "Import realm response code: ${import_realm_response}"

xdg-open http://localhost:4200

echo "Finished!"
```

## Going Further

We can make this even easier for ourselves by creating a desktop shortcut that will run this script with predefined arguments as they aren't likely to change very often. There are a lot of examples of how to do this online. First, we need to create a `.desktop` file with the command we want to run along with a few more properties.

```txt
[Desktop Entry]
Type=Application
Terminal=true
Name=social-care
Icon=/home/me/repos/script/icon.svg
Exec=/home/me/repos/script/setup.sh ROOT_PATH=/home/me/repos
```

Rather than explaining how to make it executable, you can follow the steps in this [answer](https://askubuntu.com/a/1256301/926763) since that's what helped me. After making it executable you should be able to launch your development environment by simply clicking a desktop icon.

## Results

To test the execution speed of the script all of the services, except PostgreSQL, were terminated and the Keycloak Docker container was removed. The dependencies were still installed. The `time` command was used to give a rough measure of how long the script took to run, although this can vary.

```txt
me@ubuntu:~/repos$ time sh setup.sh ROOT_PATH=/home/me/repos/
ROOT_PATH=/home/me/repos/
Installing required packages...
Checking for curl: install ok installed
Checking for jq: install ok installed
Checking if repositories exist...
The directory /home/me/repos/keycloak-repo exists.
The directory /home/me/repos/rest-api-repo exists.
The directory /home/me/repos/web-client-repo exists.
Opening editors...
PostgreSQL is already running.
Starting Keycloak...
Keycloak has started.
Starting REST API...
REST API has started.
Starting web client...
Web client has started.
Waiting for localhost:8081...
Waiting for localhost:8081...
Waiting for localhost:8081...
Waiting for localhost:8081...
Waiting for localhost:8081...
Keycloak is now running.
REST API is now running.
Waiting for localhost:4200...
Web client is now running.
Importing realm settings...
Import realm response code: 201
Finished!

real    1m43.760s
user    0m1.639s
sys     0m0.407s
```

In this case, the script took approximately 1 minute 43 seconds. To put that into perspective, running these steps manually took 3 minutes 4 seconds, keeping in mind that is not my first time running through this manually. I've had a lot of practice. In total it took 1 minute 21 seconds less and saved a lot of clicking and typing.

## Conclusion

Putting it all together the script is approximately 160 lines long. However, a lot of those are printing messages. Ultimately, the number of checks and logging is up to you.

Unfortunately, it didn't hit the sub one minute goal. The Keycloak container is by far the slowest service to start, however, we may not need, or want, to start that every time. I'm not very familiar with writing shell scripts so there may be room for further optimisations but it will do for now.

We were able to build some level of idempotency into the script so that we don't have to worry about the state of the environment. If some services are already running then we skip them and print a message to let us know what exactly has changed.

This script likely won't suit your setup but it's not meant to. It's meant to serve as an example of how you can easily automate your own. Hopefully, you can take some principles from this and apply them to your projects.