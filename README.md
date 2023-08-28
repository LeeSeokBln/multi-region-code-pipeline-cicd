# multi-region-code-pipeline-cicd

![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/101256150/3a2f6275-6acf-4e3d-aee6-a860922b9532)

### Description
당신은 현재 공공 클라우드 기업인 아마존 웹 서비스(이하 AWS)에서 데브옵스 엔지니어로 근무 중입니다. 하지만 최근 여러 리전 단위로 운영되는 웹 서비스를 배포하고자 하는데, 일일이 배포하기에는 너무 힘들어서 개발자가 코드를 업로드 하면 자동으로 여러 리전에 배포가 되도록 구성하고자 합니다. 하지만 해당 소스코드는 CodeBuild → CodeDeploy 간 소스코드가 이동 될 때  의 과정에서 소스코드가 유출되면 안되므로 이제 유의하여 구성합니다.

### Create Code Commit & Push Application
```
$ aws codecommit create-repository --repository-name seokbin-repo --region ap-northeast-2
```
buildspec.yaml을 파일을 작성
spec/buildspec.yaml
```
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 16

  pre_build:
    commands:
      - mkdir -p $CODEBUILD_SRC_DIR/build/public
      - mkdir -p $CODEBUILD_SRC_DIR/build/spec/scripts
      - mkdir -p $CODEBUILD_SRC/spec/scripts
      - cp $CODEBUILD_SRC_DIR/index.js $CODEBUILD_SRC_DIR/build/index.js
      - cp $CODEBUILD_SRC_DIR/package.json $CODEBUILD_SRC_DIR/build/package.json
      - cp $CODEBUILD_SRC_DIR/public/index.html $CODEBUILD_SRC_DIR/build/public/index.html
      - cd $CODEBUILD_SRC_DIR/build/
      - npm install
      - pip3 install aws-encryption-sdk-cli

  build:
    commands:
      - cd $CODEBUILD_SRC_DIR
      - mkdir spec/scripts
      - tar -cvf build.tar build/
      - ls -al
      - ls -al ./spec
      - aws-encryption-cli --encrypt --input build.tar -w key=$KeyArn --metadata-output ~/metadata --encryption-context purpose=test --commitment-policy require-encrypt-require-decrypt --output .
      - |
        cat << EOF > ./spec/appspec.yml
        version: 0.0
        os: linux
        files:
          - source: /
            destination: /apps/
        hooks:
          BeforeInstall:
            - location: scripts/install_dependencies.sh
              timeout: 300
              runas: root
          AfterInstall:
            - location: scripts/change_permissions.sh
              timeout: 300
              runas: root
          ApplicationStart:
            - location: scripts/start_server.sh
              timeout: 300
              runas: root
          ApplicationStop:
            - location: scripts/stop_server.sh
              timeout: 300
              runas: root
        EOF
      - |
        cat << EOF > ./spec/scripts/install_dependencies.sh
        #!/bin/bash
        cd /apps/
        aws-encryption-cli --decrypt --input build.tar.encrypted --wrapping-keys key=$KeyArn --commitment-policy require-encrypt-require-decrypt --encryption-context purpose=test --metadata-output /apps/metadata --max-encrypted-data-keys 1 --buffer --output .
        EOF
      - |
        cat << EOF > ./spec/scripts/change_permissions.sh
        #!/bin/bash
        chmod 777 index.js
        EOF

      - |       
        cat << EOF > ./spec/scripts/start_server.sh
        #!/bin/bash
        nohup index.js > nohup.pid 2>&1 & echo $! > /apps/app.pid
        EOF

      - |        
        cat << EOF > ./spec/scripts/stop_server.sh
        #!/bin/bash
        if [ -f /apps/app.pid ]; then
          kill -15 `cat /apps/app.pid`
        fi
        EOF
      - ls -al

  post_build:
    commands:
      - echo Finished

artifacts:
  files:
    - build.tar.encrypted
    - spec/**/*
  discard-paths: no
```
파일 구조 확인

![261183424-40f0b9cc-9e23-4da5-8184-45bb3c60b90d](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/795f6130-dfb7-49c9-b149-0a754fa13448)


Code Commit에 Push
```
$ git init
$ git remote add origin https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/seokbin-repo
$ git add .
$ git commit -m "my first commit"
$ git push origin master
```
### ap-northeast-1 & ap-northeast-2 KMS 키 생성

<img src="https://github-production-user-asset-6210df.s3.amazonaws.com/86287920/263580119-6002bd1c-dafa-4339-b24e-eb334a46a8f8.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230828%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230828T024208Z&X-Amz-Expires=300&X-Amz-Signature=05483ca2853e6cbd8dab47bb0abb98a3eaf1ac7ff866e6d7e04a71f83149bc78&X-Amz-SignedHeaders=host&actor_id=86287920&key_id=0&repo_id=679512807"/>

### Create S3 Bucket & Code Build
S3 BUcket을 생성
```
$ aws s3 mb s3://skills-s3-bucket-seokbin
```
Code Build Project를 생성

![263579187-5b75621c-fef3-4f41-b29e-6b41cb43dc97](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/d0aa8055-4b2e-4342-afbf-d53d376d7458)

![263579311-21b00abb-11e7-495f-8127-1d33392697e5](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/57e6e884-abcc-456e-9990-bd0a7781e52d)

<img src="https://github-production-user-asset-6210df.s3.amazonaws.com/86287920/263579493-9543b92d-861b-4906-8956-7425910a0b68.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230828%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230828T024229Z&X-Amz-Expires=300&X-Amz-Signature=3a32e921595762fd6aa5e096bfcb10ee69c9f8be22cc0cf16735785b75864ef1&X-Amz-SignedHeaders=host&actor_id=86287920&key_id=0&repo_id=679512807" />

<img src="https://github-production-user-asset-6210df.s3.amazonaws.com/86287920/263579535-78a56652-339a-4705-93bf-1a6fc8b07ab9.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230828%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230828T024234Z&X-Amz-Expires=300&X-Amz-Signature=ca1539d6344295bafdae742aa7e4cb5ecb167367c0b25453c034af47d125b37b&X-Amz-SignedHeaders=host&actor_id=86287920&key_id=0&repo_id=679512807" />

<img src="https://github-production-user-asset-6210df.s3.amazonaws.com/86287920/263579795-c2c67e5e-03cc-43d6-892a-ff2449423f61.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230828%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230828T024241Z&X-Amz-Expires=300&X-Amz-Signature=c0ffbbf4387cbd8bebc853a355afb3e02a559de3005d3c3eb4caf12debd669be&X-Amz-SignedHeaders=host&actor_id=86287920&key_id=0&repo_id=679512807" />

<img src="https://github-production-user-asset-6210df.s3.amazonaws.com/86287920/263580297-4e535ffc-492e-43e2-bf9a-29019fa91a8b.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230828%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230828T024310Z&X-Amz-Expires=300&X-Amz-Signature=d0334282eba81ff37740b4343dfcef10f7f7f9922b7f389197d664005a33226a&X-Amz-SignedHeaders=host&actor_id=86287920&key_id=0&repo_id=679512807" />

![261183640-bdb21206-5ed8-4b4e-9bb9-5b54e968aa6d](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/68ffbaf5-cd8b-4a7d-8926-eb803eae21aa)

![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/d00455f7-85c8-47ad-882b-3eed7dd93527)

Code Build Project를 생성 그 후 해당 Code Build의 IAM Role에 아래와 같은 권한을 부여
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kms:Encrypt",
                "kms:GenerateDataKey"
            ],
            "Resource": "arn:aws:kms:ap-northeast-2:**********:key/*******************"
        }
    ]
}
```
### Create Code Deploy
IAM Role을 하나 생성

![Untitled](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/101256150/c6f8014a-916b-4233-8180-a56bf75d2684)
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "kms:Decrypt",
            "Resource": "arn:aws:kms:ap-northeast-2:**********:key/*"
        }
    ]
}
```
ap-northeast-1 리전으로 이동해서 Code Deploy를 생성

![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/c9357d11-be18-43d5-9895-93f855e66ef6)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/2e65a4f2-5938-4ed1-b2e3-3e27f2aba1a9)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/38f2cd05-c9a3-45b1-be06-ea1411effcff)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/93f8c35f-8ad3-4521-af93-bf0e10c6ae0e)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/6bc7524c-0880-4b18-aa8b-e1dedcdc2790)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/83d2f05b-3617-4245-936e-1f90c26dee8f)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/a7d9cc1d-9a47-494c-9e60-fe8393ec5e23)


EC2 생성

![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/da4e8dbc-1265-4b0d-86e3-30e227422adf)

IAM Role을 Attach

![Untitled](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/101256150/cb2e2687-8a5d-4ec9-843f-4bcd6e37f12c)
![Untitled](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/101256150/3ceffc46-b879-4122-8b9f-8d8dbcfdf958)

인스턴스를 생성

### Create CodePipeline

서울 리전으로 옮기고 Code Pipeline을 생성

![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/5208bc4a-1387-44f9-9735-d8157badd6bf)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/95708c6c-8f68-4dee-b351-ba8817d8d7dc)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/12f232f6-243a-40be-b1ab-41ee7090c075)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/edea061f-93f6-4be2-b814-771bfd2ec1c8)
![image](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/86287920/7f7bde6c-d26a-46d5-8e4d-7d3fae442516)

Pipeline을 생성

만약 CodeBuild에서 Access Denied가 뜰 경우 아래와 같이 해당 Build의 IAM Role의 정책을 수정해준다

![Untitled](https://github.com/LeeSeokBln/multi-region-code-pipeline-cicd/assets/101256150/940b975b-059d-4d1a-8216-429eaca14902)
