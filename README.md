# Using Containers for Deployment in Activo

## Introduction
A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another. A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings.

## Technologies Used
- [Docker](docker.com) - Docker is a computer program that performs operating-system-level virtualization
- [Kubernetes](https://kubernetes.io/) - Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications
![Chart_02_Kubernetes-Architecture](https://user-images.githubusercontent.com/6943256/56995631-3338b200-6b9a-11e9-94db-4b37427321e8.png)
- [Google Cloud Platform](cloud.google.com)
- [Circle CI](circleci.com)

## How the application works with containers
Docker is being used to build images, and containers can be ran from the images built, kubernetes is used for container orchestration in the application

## The frontend Dockerfile
```
# base image 
FROM node:8-alpine AS build

LABEL maintainer="Anyama Richard" MAINTAINER="Idrees Ibraheem <idrees.ibraheem@andela.com>"
LABEL application="activo-web"

# the working directory where the application would be started
WORKDIR /root/activo-web

# The Yarn.lock and package.json file is copied so that the versions 
# in the package.json are not upgraded from what is present in the 
# local package.json to a higher version in the container image.
COPY yarn.lock /root/activo-web
COPY package.json /root/activo-web

# RUN yarn install 

# RUN yarn remove node-sass

# # same issue above occurs here
# RUN yarn add node-sass

# # same issue above occurs here
# RUN yarn add react-tooltip

# update the Alpine image and install curl
RUN apk update && apk add curl 

# installing Alpine Dependencies, but the context for the command from `yarn install` is explained above
# from lines 17 > 39 

RUN apk add --no-cache --virtual .build-deps1 g++ gcc libgcc libstdc++ linux-headers make python && \
    apk add --no-cache --virtual .npm-deps cairo-dev jpeg-dev libjpeg-turbo-dev pango pango-dev && \
    yarn install && yarn remove node-sass && yarn add node-sass && yarn add react-tooltip 

# Build the application final image with the base alpine image
FROM node:8-alpine

WORKDIR /app

# update the Alpine image and install curl
RUN apk update && apk add curl

# copy dependencies and the dist/ directory from the previous build stage.
COPY --from=build /root/activo-web/node_modules ./node_modules/

# Copy all files from the root directory into the image
COPY . .
```
 The frontend image is built from a `node:8-alpine` image, the commands are being executed in sequence, the frontend can be executed with the command `docker build -t latest .`

## The backend Dockerfile
```
FROM alpine:3.7 as build

# Maintainer
LABEL maintainer="Idrees Ibraheem <idrees.ibraheem@andela.com>"

ENV LC_ALL C.UTF-8
ENV LANG C.UTF-8

# Working directory where all the setup would take place in the image
WORKDIR /activo-api

# The default user that should be used
USER root

# copy the Pipfile & pipfile.lock which contains dependencies to be installed
COPY ./Pipfile /activo-api
COPY ./Pipfile.lock /activo-api

# Install Alpine packages needed for the provisioning of the instance with python
# and other packages
RUN apk -U update 
RUN apk add --no-cache --update build-base 
RUN apk add curl git libffi-dev ncurses-dev openssl libressl python3-dev musl-dev libpq readline-dev tk-dev xz-dev zlib-dev py-psycopg2 linux-headers
RUN apk add bash zlib-dev zlib bzip2-dev libffi libffi-dev readline-dev postgresql-libs sqlite-dev py3-virtualenv postgresql-dev && \
    curl -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | sh && \
    export PATH="/root/.pyenv/bin:$PATH" && \
    eval "$(pyenv init -)" && \
    eval "$(pyenv virtualenv-init -)"  && \
    pyenv install 3.6.5 -s && \
    export PATH="/root/.pyenv/versions/3.6.5:$PATH" && \
    export LC_ALL=C.UTF-8 && \
    export LANG=C.UTF-8 && \
    export PATH="/root/activo-api/start.sh:$PATH" && \
    pip3 install --user pipenv==2018.5.18 && \
    python3 -m pip install pipenv==2018.5.18 && \
    python3 -m pipenv install gunicorn 

# check the root dir for the .cache dir
RUN ls -ahl /root

# remove the .cache dir
RUN rm -rf /root/.cache

# build the minimal application image from Alpine
FROM alpine:3.7 

USER root

# Install packages needed for the application to run
RUN apk -U update 
RUN apk add --no-cache curl git libffi-dev ncurses-dev openssl libressl python3 py3-flask  python3-dev musl-dev libpq readline-dev tk-dev xz-dev zlib-dev py-psycopg2 linux-headers \
    bash zlib-dev zlib bzip2-dev libffi libffi-dev readline-dev postgresql-libs sqlite-dev py3-virtualenv postgresql-dev


WORKDIR /activo-api

COPY --from=build /root /root

# RUN export PATH="/root/root/start.sh:$PATH" 
RUN export PATH="/root/activo-api/start_api.sh:$PATH" && \
     export PATH="/root/.pyenv /versions/3.6.5:$PATH" 
RUN ls -ahl /root

RUN rm -rf .cache

COPY . .

```
 The commands are being executed in sequece when the dockerfile is ran, the file can be executed with the command `docker build -t latest .`
 
 ## The backend Circle CI config
 ```
 git default: &defaults
  docker:
      - image: gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-ci-image
        auth:
          username: _json_key
          password: '${SERVICE_ACCOUNT}'
        environment:
          ACTIVO_PATH: /home/circleci/activo-api
          CC_TEST_REPORTER_ID: ${CC_TEST_REPORTER_ID}
          FLASK_ENV: testing
          FLASK_APP: manage.py
          PGUSER: circleci
          PG_HOST: localhost
          TEST_DATABASE_URL: postgresql://circleci@localhost/circlecidb
      - image: postgres:9.6
        environment:
          POSTGRES_USER: circleci
          POSTGRES_DB: activo_test
          POSTGRES_PASSWORD: ''
  # specify working directory
  working_directory: ~/activo-api

release_default: &release_defaults
  docker:
    - image: gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-ci-image
      auth:
        username: _json_key
        password: '${SERVICE_ACCOUNT}'
  working_directory: ~/activo-api

cmd_wait_for_postgres: &cmd_wait_for_postgres
  run:
    name: Waiting for Postgres to be ready
    command: |
      dockerize -wait tcp://localhost:5432 -timeout 1m
cmd_install_dependencies: &cmd_install_dependencies
  run:
    name: Install dependencies
    command: |
      curl -L https://raw.githubusercontent.com/pyenv/pyenv-installer/master/bin/pyenv-installer | bash
      export PATH="/home/circleci/.pyenv/bin:$PATH"
      eval "$(pyenv init -)"
      eval "$(pyenv virtualenv-init -)"
      pyenv install 3.6.5 -s
      pyenv local 3.6.5
      pip3 install --user pipenv==2018.5.18
      python3 -m pip install pipenv==2018.5.18
      python3 -m pipenv install

# clone the infrastructure repository
cmd_clone_infra_repo: &cmd_clone_infra_repo
  run:
    name: Clone infrastructure repository and create service Account
    command: |
      git clone -b develop ${INFRASTRUCTURE_REPO} /home/circleci/activo-infra
      touch /home/circleci/activo-infra/ansible/account.json
      echo ${SERVICE_ACCOUNT} > /home/circleci/activo-infra/ansible/account.json

# setup environment for k8s GCP authentication and cluster connection
cmd_set_up_k8s_cluster_prod: &cmd_set_up_k8s_cluster_prod
  run:
    name: Setup environment for kubernetes deployment & authentication
    command: |
      cd /home/circleci/activo-infra
      gcloud config set project ${GCLOUD_ACTIVO_PROJECT}
      gcloud auth activate-service-account --key-file=./ansible/account.json
      gcloud container clusters get-credentials ${K8S_CLUSTER_PROD} --zone="europe-west1-b"  --project ${GCLOUD_ACTIVO_PROJECT}

# setup environment for k8s GCP authentication and cluster connection
cmd_set_up_k8s_cluster_sandbox: &cmd_set_up_k8s_cluster_sandbox
  run:
    name: Setup environment for kubernetes deployment & authentication
    command: |
      cd /home/circleci/activo-infra
      gcloud config set project ${GCLOUD_ACTIVO_PROJECT}
      gcloud auth activate-service-account --key-file=./ansible/account.json
      gcloud container clusters create ${K8S_CLUSTER_SANDBOX} --zone="europe-west1-b"
      gcloud container clusters get-credentials ${K8S_CLUSTER_SANDBOX} --zone="europe-west1-b"  --project ${GCLOUD_ACTIVO_PROJECT}
cmd_set_up_k8s_cluster_staging: &cmd_set_up_k8s_cluster_staging
  run:
    name: Setup environment for kubernetes deployment & authentication
    command: |
      cd /home/circleci/activo-infra
      gcloud config set project ${GCLOUD_ACTIVO_PROJECT}
      gcloud auth activate-service-account --key-file=./ansible/account.json
      gcloud container clusters get-credentials ${K8S_CLUSTER_STAGING} --zone="europe-west1-b"  --project ${GCLOUD_ACTIVO_PROJECT}
cmd_install_dependencies: &cmd_save_cache
    save_cache:
        key: api-dependencies-{{ checksum "Pipfile.lock" }}
        paths:
          - $(python3 -m pipenv --venv)

cmd_restore_cache: &cmd_restore_cache
    restore_cache:
        keys:
          - api-dependencies-{{ checksum "Pipfile.lock" }}
          # fallback to using the latest cache if no exact match is found
          - api-dependencies-

cmd_download_cc_test_reporter: &cmd_download_cc_test_reporter
  run:
    name:  Download cc-test-reporter
    command: |
      mkdir -p tmp/
      curl -L https://codeclimate.com/downloads/test-reporter/test-reporter-latest-linux-amd64 > /tmp/cc-test-reporter
      chmod +x /tmp/cc-test-reporter  

cmd_attach_workspace: &cmd_attach_workspace
  attach_workspace:
    at: tmp/  

# Python CircleCI 2.0 configuration file
version: 2
jobs:
  build:
    <<: *defaults
    steps:
      - checkout
      - *cmd_install_dependencies
      - *cmd_save_cache
      - *cmd_wait_for_postgres
      - run:
          name: Set up database
          command: |
            source $(python3 -m pipenv --venv)/bin/activate
            # flask db init
            # flask db migrate
            flask db upgrade
      - *cmd_download_cc_test_reporter
  run_tests:
    <<: *defaults
    steps:
      - checkout
      - *cmd_attach_workspace
      - *cmd_install_dependencies
      - *cmd_save_cache
      - *cmd_wait_for_postgres
      - *cmd_restore_cache
      - *cmd_download_cc_test_reporter
      - run:
          name: Run tests
          command: |
            source $(python3 -m pipenv --venv)/bin/activate
            pytest --cov=api/ tests --cov-report xml
            /tmp/cc-test-reporter format-coverage coverage.xml -t "coverage.py" -o "tmp/cc.testreport.json"

      - persist_to_workspace:
          root: tmp/
          paths:
            - cc.testreport.json

  upload_coverage:
    <<: *defaults
    steps:
      - checkout
      - *cmd_download_cc_test_reporter
      - *cmd_attach_workspace
      - run:
          name: Upload coverage results to Code Climate
          command: |
            /tmp/cc-test-reporter upload-coverage -i tmp/cc.testreport.json

  build_docker_image:
      <<: *defaults
      steps:
        - checkout
        - setup_remote_docker
        - *cmd_clone_infra_repo
        - run:
            name: Bake Docker Image
            command: |
                  # build image for development
                  if [[ "${CIRCLE_BRANCH}" == "develop" ]]; then
                    # copy env vars
                    env >> .env
                    sed -i 's/testing/staging/g' .env

                    # replace the index to the one assigned to staging
                    sed -i  "s|:6379\/7|:6379\/8|g" .env

                    echo "DATABASE_URI=${STAGING_DB_URL}" >> .env
                    
                    # build docker image and push to GCR
                    docker login -u _json_key -p "$(echo $SERVICE_ACCOUNT)" https://gcr.io
                    docker build -t gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-${DEPLOY_STAGING_ENV}:${CIRCLE_SHA1} -f docker-release/Dockerfile .
                    docker push gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-${DEPLOY_STAGING_ENV}:${CIRCLE_SHA1}
                  fi

                  #  build image for production
                  if [[ "${CIRCLE_BRANCH}" == "master" ]]; then
                    # copy env vars
                    env
                    env >> .env
                    # replace the env value for deployment from 'staging' to 'production'
                    sed -i 's/testing/production/g' .env 

                    echo "DATABASE_URI=${PROD_DB_URL}" >> .env
                  
                    # replace the index to the one assigned to production
                    sed -i  "s|:6379\/7|:6379\/9|g" .env

                    # build docker image and push to GCR
                    docker login -u _json_key -p "$(echo $SERVICE_ACCOUNT)" https://gcr.io
                    docker build -t gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-${DEPLOY_PROD_ENV}:${CIRCLE_SHA1} -f docker-release/Dockerfile .
                    docker push gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-${DEPLOY_PROD_ENV}:${CIRCLE_SHA1}
                  fi

  build_docker_image_sandbox:
      <<: *defaults
      steps:
        - checkout
        - setup_remote_docker
        - *cmd_install_dependencies
        - *cmd_clone_infra_repo
        - attach_workspace:
            at: ~/activo-api
        - run:
            name: Bake Docker Image
            command: |
              if [[ "${CIRCLE_BRANCH}" =~ "sandbox" ]]; then
                # copy env vars 
                env >> .env
                sed -i 's/testing/staging/g' .env
                echo "DATABASE_URI=${STAGING_DB_URL}" >> .env
                docker login -u _json_key -p "$(echo $SERVICE_ACCOUNT)" https://gcr.io
                docker build -t gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-${DEPLOY_ENV_SANDBOX}:${CIRCLE_SHA1} -f docker-release/Dockerfile .
                docker push gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-${DEPLOY_ENV_SANDBOX}:${CIRCLE_SHA1}
              fi
  release_to_sandbox:
    <<: *release_defaults
    steps:
      - run: exit 0

  release_to_staging:
    <<: *release_defaults
    steps:
      - run: exit 0

  release_to_production:
    <<: *release_defaults
    steps:
      - run: exit 0

  create_ops_sandbox: # deployment to sandbox
    #  docker environment for running k8s deployment
    docker:
      - image: gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-ci-ansible
        auth:
          username: _json_key
          password: '${SERVICE_ACCOUNT}'
    steps:
      - checkout
      - *cmd_clone_infra_repo
      - *cmd_set_up_k8s_cluster_sandbox
      - attach_workspace:
          at: ~/activo-api
      - run:
          name: install envsubst
          command: sudo apt-get install -y gettext
      - run:
          name: Create K8s Objects
          command: |
            export DEPLOYMENT_ENVIRONMENT=${DEPLOY_SANDBOX_ENV}
            cd /home/circleci/activo-infra
            
            envsubst < ./deploy/api/deployment.yml.tpl > deployment.yml
            envsubst < ./deploy/shared/shared-ingress-service.yml.tpl > ingress.yml
            envsubst < ./deploy/api/service.yml.tpl > service.yml
            envsubst < ./deploy/api/autoscaler.yml.tpl > autoscaler.yml
            envsubst < ./deploy/shared/nginx-service.yml.tpl > nginx-service.yml

            # Authenticate for ansible-playbook
            echo ${SERVICE_ACCOUNT} > /home/circleci/activo-infra/ansible/account.json
            export GOOGLE_APPLICATION_CREDENTIALS=/home/circleci/activo-infra/ansible/account.json

            ansible-playbook ./ansible/playbook.main.yml
            
            # notify slack
            bash ./deploy/notify_slack.sh
  cleanup:
      <<: *defaults
      steps:
        - checkout
        - *cmd_clone_infra_repo
        - attach_workspace:
            at: ~/activo-api
        - deploy:
            name: Delete services and cluster
            command: |
              if [[ "${CIRCLE_BRANCH}" != "master" || "${CIRCLE_BRANCH}" != "develop" ]]; then
              
                gcloud config set project ${GCLOUD_ACTIVO_PROJECT}
                gcloud auth activate-service-account --key-file=./ansible/account.json
                gcloud container clusters get-credentials ${K8S_CLUSTER_SANDBOX} --zone="europe-west1-b"  --project ${GCLOUD_ACTIVO_PROJECT}
                
                # TODO  add ansible script to delete k8s OBJECTs created before deleting the cluster
                gcloud container clusters delete ${K8S_CLUSTER_SANDBOX}
              fi

  deploy_to_heroku: 
    #  docker environment for running k8s deployment
    docker:
      - image: gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-ci-image
        auth:
          username: _json_key
          password: '${SERVICE_ACCOUNT}'
    steps:
      - checkout
      - setup_remote_docker
      - *cmd_clone_infra_repo
      - attach_workspace:
          at: ~/activo-api
      - run:
          name: Deploy to Heroku
          command: | 
            # Deploy to the activo-web sandbox
            if [[ "$CIRCLE_BRANCH" == "develop" ]]; then
              #  copy env vars into the instance
              env
              env >> .env

              # install heroku CLI
              curl https://cli-assets.heroku.com/install.sh | sh

              # Build docker image and push to Heroku registry
              docker login --username=$HEROKU_LOGIN --password=$HEROKU_API_KEY registry.heroku.com

              # login to heroku docker registry
              heroku container:login

              # NOTICE

              # if you are sharing the dyno with a new user 
              # ensure that you add the user as a collaborator 
              # else they would not be able to push images to the
              # the Heroku registry.

              #  Build docker image for the docker container
              #  Tag and Push image to heroku container registory
              docker build -t registry.heroku.com/activo-new-testing/activo-api-${DEPLOY_STAGING_ENV}:${CIRCLE_SHA1} -f docker-heroku/Dockerfile .
              docker tag registry.heroku.com/activo-new-testing/activo-api-${DEPLOY_STAGING_ENV}:${CIRCLE_SHA1} registry.heroku.com/activo-new-testing/web
              docker push registry.heroku.com/activo-new-testing/web 

              # re-tag docker image for another sandbox env and push 
              docker tag registry.heroku.com/activo-new-testing/activo-api-${DEPLOY_STAGING_ENV}:${CIRCLE_SHA1} registry.heroku.com/e2e-api-develop/web
              docker push registry.heroku.com/e2e-api-develop/web 

              #  release built application to activo-new-testing dyno
              heroku container:release web -a activo-new-testing

              # release application application to e2e-develop-api dyno
              heroku container:release web -a e2e-api-develop
              
            fi
            
            # deployment config for sandbox
            if [[ "$CIRCLE_BRANCH" == "sandbox" || "$CIRCLE_BRANCH" == "test-heroku-sandbox" ]]; then
              #  copy env vars into the instance
              env
              env >> .env

              # install heroku CLI
              curl https://cli-assets.heroku.com/install.sh | sh

              # Build docker image and push to Heroku registry
              docker login --username=$HEROKU_LOGIN --password=$HEROKU_API_KEY registry.heroku.com

              # login to heroku docker registry
              heroku container:login

              #  Build docker image for the docker container
              #  Tag and Push image to heroku container registory
              docker build -t registry.heroku.com/activo-api-sandbox/activo-api-sandbox:${CIRCLE_SHA1} -f docker-heroku/Dockerfile .
              docker tag registry.heroku.com/activo-api-sandbox/activo-api-sandbox:${CIRCLE_SHA1} registry.heroku.com/activo-api-sandbox/web
              docker push registry.heroku.com/activo-api-sandbox/web 
              
              #  release built application in heroku
              heroku container:release web -a activo-api-sandbox
            fi


  # Staging Deployment
  deploy_staging: 
    #  docker environment for running k8s deployment
    docker:
      - image: gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-ci-ansible
        auth:
          username: _json_key
          password: '${SERVICE_ACCOUNT}'
    steps:
      - checkout
      - *cmd_clone_infra_repo
      - *cmd_set_up_k8s_cluster_staging
      - attach_workspace:
          at: ~/activo-api
      - run:
          name: install envsubst
          command: sudo apt-get install -y gettext
      - run:
          name: Deploy to Staging
          command: | 
            if [[ "$CIRCLE_BRANCH" == "develop" ]]; then
              # use envsubst to run run variable substitution 
              export DEPLOYMENT_ENVIRONMENT=${DEPLOY_STAGING_ENV}
              export K8S_INGRESS_IP=${K8S_STAGING_STATIC_IP}
              export WEB_HOST_DOMAIN=${ACTIVO_WEB_STAGING_URL}
              export API_HOST_DOMAIN=${ACTIVO_API_STAGING_URL}
              cd /home/circleci/activo-infra

              envsubst < ./deploy/api/deployment.yml.tpl > deployment.yml
              envsubst < ./deploy/shared/shared-ingress-service.yml.tpl > ingress.yml
              envsubst < ./deploy/api/service.yml.tpl > service.yml
              envsubst < ./deploy/api/autoscaler.yml.tpl > autoscaler.yml
              envsubst < ./deploy/shared/nginx-service.yml.tpl > nginx-service.yml

              # Authenticate for ansible-playbook
              echo ${SERVICE_ACCOUNT} > /home/circleci/activo-infra/ansible/account.json
              export GOOGLE_APPLICATION_CREDENTIALS=/home/circleci/activo-infra/ansible/account.json

              ansible-playbook ./ansible/playbook.main.yml 
              
              # notify slack
              bash ./deploy/notify_slack.sh
            fi

  # production deployment
  deploy_production:
    #  docker environment for running k8s deployment
    docker:
      - image: gcr.io/${GCLOUD_ACTIVO_PROJECT}/activo-api-ci-ansible
        auth:
          username: _json_key
          password: '${SERVICE_ACCOUNT}'
    steps:
      - checkout
      - *cmd_clone_infra_repo
      - *cmd_set_up_k8s_cluster_prod
      - attach_workspace:
          at: ~/activo-api
      - run:
          name: install envsubst
          command: sudo apt-get install -y gettext
      - deploy:
          name: Deploy to production
          command: |
            if [[ "${CIRCLE_BRANCH}" == "master" ]]; then
               # use envsubst to run run variable substitution 
              export DEPLOYMENT_ENVIRONMENT=${DEPLOY_PROD_ENV}
              export K8S_INGRESS_IP=${K8S_PROD_STATIC_IP}
              export WEB_HOST_DOMAIN=${ACTIVO_WEB_PROD_URL} 
              export API_HOST_DOMAIN=${ACTIVO_API_PROD_URL} 
              cd /home/circleci/activo-infra

              envsubst < ./deploy/api/deployment.yml.tpl > deployment.yml
              envsubst < ./deploy/shared/shared-ingress-service.yml.tpl > ingress.yml
              envsubst < ./deploy/api/service.yml.tpl > service.yml
              envsubst < ./deploy/api/autoscaler.yml.tpl > autoscaler.yml
              envsubst < ./deploy/api/cronjob.yml.tpl > cronjob.yml
              envsubst < ./deploy/shared/nginx-service.yml.tpl > nginx-service.yml

              # Authenticate for ansible-playbook
              echo ${SERVICE_ACCOUNT} > /home/circleci/activo-infra/ansible/account.json
              export GOOGLE_APPLICATION_CREDENTIALS=/home/circleci/activo-infra/ansible/account.json

              ansible-playbook ./ansible/playbook.main.yml
              kubectl apply -f cronjob.yml
              
              # notify slack
              bash ./deploy/notify_slack.sh
            fi

# workflows
workflows:
  version: 2
  build_and_test:
    jobs:
      - build
      - run_tests:
          requires:
            - build
      - upload_coverage:
          requires:
            - run_tests
      - build_docker_image:
          filters:
            branches:
              only:
                - master
                - develop
            tags:
              only:
                - /v[0-9]+(\.[0-9]+)*/ 
      - release_to_staging:
          requires:
            - upload_coverage
            - build_docker_image
          filters:
            branches:
              only: 
                - develop
            tags:
              only:
                - /v[0-9]+(\.[0-9]+)*/
      - deploy_staging:
          requires:
            - release_to_staging
          filters:
            branches:
              only: 
                - develop
            tags:
              only:
                - /v[0-9]+(\.[0-9]+)*/
      - deploy_to_heroku:
          requires:
            - upload_coverage
          filters:
            branches:
              only: 
                - develop
                - sandbox
                - test-heroku-sandbox
            tags:
              only:
                - /v[0-9]+(\.[0-9]+)*/
      - release_to_production:
          requires:
            - upload_coverage
            - build_docker_image
          filters:
            branches:
              only: 
                - master
            tags:
              only:
                - /v[0-9]+(\.[0-9]+)*/
      - deploy_production:
          requires:
            - release_to_production
          filters:
            branches:
              only: 
                - master
            tags:
              only:
                - /v[0-9]+(\.[0-9]+)*/

 ```
 The circle ci script for the backend contains several workflows, when a pushed is made to the remote repository, the following process takes place
 - Build and test
 - Build docker image
 - Release to staging
 - Deploy staging
 - Deploy to heroku
 - Release to production
 - Deploy production
 
## The frontend Circle CI config
[Frontend Circle CI](https://github.com/andela/activo-web/blob/develop/.circleci/config.yml)
The config is slightly different from the back end, but same process takes place as the backend
