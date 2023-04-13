# GITLAB-CI-COMMANDS
mvn_sonar_merge_request:
  stage: check
  allow_failure: false
  script:
    - mvn clean install -U compile -DskipTests=true
    - mvn package -B -DskipTests=true
    - mvn sonar:sonar -Dsonar.host.url=https://sonarq.hmb.gov.tr  -Dsonar.login=yazilimAltyapi  -Dsonar.password=yazilimAltyapi
  only:
   refs :
     - merge_requests
   variables :
      -  $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "test"
  tags:
    - deploy-runner


include:
  - project: 'operasyon/uygulama-veri/devops/deployment'
    file: 'variables.yml'
  - project: 'operasyon/uygulama-veri/devops/deployment'
    file: 'global-jobs.yml'

stages:
  - check
  - build
  - deploy

mvn_sonar_merge_request:
  stage: check
  allow_failure: false
  before_script:
    - export JAVA_HOME=/usr/lib/jvm/jdk-17.0.1
    - export JAVA_17_HOME=/usr/lib/jvm/jdk-17.0.1
    - export PATH=$PATH:/usr/lib/jvm/jdk-17.0.1/bin
  script:
    - mvn -v
    - mvn clean install -U compile -DskipTests=true
    - mvn package -B -DskipTests=true
    - mvn sonar:sonar -Dsonar.host.url=https://sonarq.hmb.gov.tr  -Dsonar.login=yazilimAltyapi  -Dsonar.password=yazilimAltyapi
  only:
   refs :
     - merge_requests
   variables :
      -  $CI_MERGE_REQUEST_TARGET_BRANCH_NAME == "test"
  tags:
    - deploy-runner

build_dev_image:
  stage: build
  before_script:
    - docker login -u $GITLAB_REGISTRY_USER -p $GITLAB_REGISTRY_TOKEN git-registry.hmb.gov.tr
    - export JAVA_HOME=/usr/lib/jvm/jdk-17.0.1
    - export JAVA_17_HOME=/usr/lib/jvm/jdk-17.0.1
    - export PATH=$PATH:/usr/lib/jvm/jdk-17.0.1/bin
  script:
    - mvn -v
    - mvn clean install -U compile -DskipTests=true
    - mvn package -B -DskipTests=true
    - cp -r /opt/pinpoint-agent-2.3.3 agent/
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-dev .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-dev
  only:
    - dev
  tags:
    - deploy-runner

k8s-dev-oecd-deploy:
  stage: deploy
  script:
    - export IMAGE=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-dev
    - export ENVIRONMENT=infra-dev,dev
    - sed -i --expression s@'$TAG'@$IMAGE@g deployment.yml
    - sed -i --expression s@'$PROFILE'@$ENVIRONMENT@g deployment.yml
    - kubectl config use-context kaps
    - kubectl apply -f deployment.yml
    - kubectl apply -f service.yml
  tags:
    - k8s-dev-oecd-runner
  only:
    - dev

k8s-dev-emek-deploy:
  stage: deploy
  script:
    - export IMAGE=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-dev
    - export ENVIRONMENT=infra-dev,dev
    - sed -i --expression s@'$TAG'@$IMAGE@g deployment.yml
    - sed -i --expression s@'$PROFILE'@$ENVIRONMENT@g deployment.yml
    - kubectl config use-context kaps
    - kubectl apply -f deployment.yml
    - kubectl apply -f service.yml
  tags:
    - k8s-dev-emek-runner
  only:
    - dev

build_test_image:
  stage: build
  before_script:
    - docker login -u $GITLAB_REGISTRY_USER -p $GITLAB_REGISTRY_TOKEN git-registry.hmb.gov.tr
    - export JAVA_HOME=/usr/lib/jvm/jdk-17.0.1
    - export JAVA_17_HOME=/usr/lib/jvm/jdk-17.0.1
    - export PATH=$PATH:/usr/lib/jvm/jdk-17.0.1/bin
  script:
    - mvn -v
    - mvn clean install -U compile -DskipTests=true
    - mvn package -B -DskipTests=true
    - cp -r /opt/pinpoint-agent-2.3.3 agent/
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-test .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-test
  only:
    - test
  tags:
    - deploy-runner

k8s-test-deploy:
  stage: deploy
  script:
    - export IMAGE=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA-test    
    - export ENVIRONMENT=infra-prod,test
    - sed -i --expression s@'$TAG'@$IMAGE@g deployment.yml
    - sed -i --expression s@'$PROFILE'@$ENVIRONMENT@g deployment.yml
    - kubectl config use-context kaps
    - kubectl apply -f deployment.yml
    - kubectl apply -f service.yml
  tags:
    - k8s-test-deploy
  only:
    - test

build_prod_image:
  stage: build
  before_script:
    - docker login -u $GITLAB_REGISTRY_USER -p $GITLAB_REGISTRY_TOKEN git-registry.hmb.gov.tr
    - export JAVA_HOME=/usr/lib/jvm/jdk-17.0.1
    - export JAVA_17_HOME=/usr/lib/jvm/jdk-17.0.1
    - export PATH=$PATH:/usr/lib/jvm/jdk-17.0.1/bin
  script:
    - export DEPLOY_VERSION=$(grep -m1 '<version>' pom.xml | grep -oP  '(?<=>).*(?=<)' | awk '{print tolower($0)}')
    - echo $DEPLOY_VERSION >> variables-prod
    - mvn clean install -U compile -DskipTests=true
    - mvn package -B -DskipTests=true
    - cp -r /opt/pinpoint-agent-2.3.0 agent/
    - docker build -f Dockerfile.prod -t $CI_REGISTRY_IMAGE:$DEPLOY_VERSION .
    - docker push $CI_REGISTRY_IMAGE:$DEPLOY_VERSION
  only:
    - master
  tags:
    - deploy-runner
  artifacts:
    paths:
    - variables-prod

deploy_prod_11:
  stage: deploy
  variables:
    GIT_STRATEGY: none
  before_script:
    - docker login -u $GITLAB_REGISTRY_USER -p $GITLAB_REGISTRY_TOKEN git-registry.hmb.gov.tr
  script: 
    - export DEPLOY_VERSION=$(cat variables-prod)
    - docker pull $CI_REGISTRY_IMAGE:$DEPLOY_VERSION
    - docker stop $(docker ps -f "name=kaps" -a -q) || true
    - docker rm $(docker ps -f "name=kaps" -a -q) || true
    - docker run --name kaps -d --restart=always -v /var/log/mics:/var/log/mics -e "TZ=Europe/Istanbul" $PROD_REDIS_CLUSTER -e JAVA_OPTS='-Xmx2g' -e "SPRING_PROFILES_ACTIVE=production" -p 9210:9500 -t $CI_REGISTRY_IMAGE:$DEPLOY_VERSION

  only:
    - master
  tags:
    - kaps-prod-11
