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
