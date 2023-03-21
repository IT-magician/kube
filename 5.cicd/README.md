#### 1. 환경 설정

- 1.1 CloudBuild API 활성화
  ```bash
  gcloud services enable container.googleapis.com \
  cloudbuild.googleapis.com \
  sourcerepo.googleapis.com \
  containeranalysis.googleapis.com
  ```
- 1.2 도커 이미지가 저장될 저장소 생성 (Artifact Registry)
  ```bash
  gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=us-central1
  ```
- 1.3 클러스터 생성
  ```bash
  gcloud container clusters create hello-cloudbuild \
    --num-nodes 1 --region us-central1
  ```
- 1.4 git 설정
  ```bash
  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
  ```

#### 2. 소스가 저장될 저장소 생성 (Source Repository)

- 2.1 두 개의 리포지토리 생성
  ```bash
  gcloud source repos create hello-cloudbuild-app
  gcloud source repos create hello-cloudbuild-env
  ```
- 2.2 코드 복제
  ```bash
  cd ~
  git clone https://github.com/GoogleCloudPlatform/gke-gitops-tutorial-cloudbuild \
  hello-cloudbuild-app
  ```
- 2.3 원격 리포지토리 설정
  ```bash
  cd ~/hello-cloudbuild-app
  PROJECT_ID=$(gcloud config get-value project)
  git remote add google \
      "https://source.developers.google.com/p/${PROJECT_ID}/r/hello-cloudbuild-app"
  ```

#### 3. Cloud Build를 사용하여 컨테이너 이미지 생성

- 3.1 Dockerfile 확인
  ```bash
  cat > Dockerfile<<EOF
  FROM python:3.7-slim
  RUN pip install flask
  WORKDIR /app
  COPY app.py /app/app.py
  ENTRYPOINT ["python"]
  CMD ["/app/app.py"]
  # [END dockerfile]
  EOF
  ```
- 3.2 Cloud Build를 사용하여 컨테이너 이미지 생성 후, push
  ```bash
  # cd ../hello-cloudbuild-app
  COMMIT_ID="$(git rev-parse --short=7 HEAD)"
  gcloud builds submit --tag="us-central1-docker.pkg.dev/${PROJECT_ID}/my-repository/hello-cloudbuild:${COMMIT_ID}" .
  ```
- 3.3 Artifact Registry > Repositories에서 이미지 확인

#### 4. Countinuous Integration(CI) 파이프라인(Cloud Build)

- 4.1 Cloud Build의 Trigger로 이동하여 "Create Trigger" 클릭
- 4.2 Name: "hello-cloudbuild", Repository: "hello-cloudbuild-app", Branch: "^master$" 입력
- 4.3 실행되는 cloudbuild.yaml 파일의 내용

  ```yaml
  # [START cloudbuild]
  steps:
    # This step runs the unit tests on the app
    - name: "python:3.7-slim"
      id: Test
      entrypoint: /bin/sh
      args:
        - -c
        - "pip install flask && python test_app.py -v"

    # This step builds the container image.
    - name: "gcr.io/cloud-builders/docker"
      id: Build
      args:
        - "build"
        - "-t"
        - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/hello-cloudbuild:$SHORT_SHA"
        - "."

    # This step pushes the image to Artifact Registry
    # The PROJECT_ID and SHORT_SHA variables are automatically
    # replaced by Cloud Build.
    - name: "gcr.io/cloud-builders/docker"
      id: Push
      args:
        - "push"
        - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/hello-cloudbuild:$SHORT_SHA"
  # [END cloudbuild]
  ```

- 4.3 트리거 하기 위해 코드 업로드
  ```bash
  #cd ~/hello-cloudbuild-app
  git push google master
  ```

#### 5. test 환경과 CD 파이프라인 생성

- 5.1 Cloud Build가 GKE에 접속할 수 있도록 권한 부여
  ```bash
  PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')"
  gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
    --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
    --role=roles/container.developer
  ```
- 5.2 환경 리포지토리 복제
  ```bash
  cd ~
  gcloud source repos clone hello-cloudbuild-env
  cd ~/hello-cloudbuild-env
  git checkout -b production
  ```
- 5.3 CD를 위한 cloudbuild.yaml 파일 생성

  ```bash
  cd ~/hello-cloudbuild-env
  cp ~/hello-cloudbuild-app/cloudbuild-delivery.yaml ~/hello-cloudbuild-env/cloudbuild.yaml
  git add .
  git commit -m "Create cloudbuild.yaml for deployment"
  ```

  ```yaml
  # cloudbuild.yaml (cloudbuild-delivery.yaml)
  # [START cloudbuild-delivery]
  steps:
    # This step deploys the new version of our container image
    # in the hello-cloudbuild Kubernetes Engine cluster.
    - name: "gcr.io/cloud-builders/kubectl"
      id: Deploy
      args:
        - "apply"
        - "-f"
        - "kubernetes.yaml"
      env:
        - "CLOUDSDK_COMPUTE_REGION=us-central1"
        - "CLOUDSDK_CONTAINER_CLUSTER=hello-cloudbuild"

    # This step copies the applied manifest to the production branch
    - name: "gcr.io/cloud-builders/git"
      id: Copy to production branch
      entrypoint: /bin/sh
      args:
        - "-c"
        - |
          git config user.email $(gcloud auth list --filter=status:ACTIVE --format='value(account)') && \
          git fetch origin production && git checkout production && \
          git checkout $COMMIT_SHA kubernetes.yaml && \
          git commit -m "Manifest from commit $COMMIT_SHA
          $(git log --format=%B -n 1 $COMMIT_SHA)" && \
          git push origin production

  # [END cloudbuild-delivery]
  ```

- 5.4 배포 테스트를 위한 브랜치 생성
  ```bash
  git checkout -b candidate
  git push origin production
  git push origin candidate
  ```
- 5.5 Cloud Build가 소스 저장소에 접근하기 위한 권한 부여

  ```bash
  PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} \
   --format='get(projectNumber)')"
  cat >/tmp/hello-cloudbuild-env-policy.yaml <<EOF
  bindings:
  - members:
    - serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com
    role: roles/source.writer
  EOF
  gcloud source repos set-iam-policy \
    hello-cloudbuild-env /tmp/hello-cloudbuild-env-policy.yaml
  ```

- 5.6 CD 파이프라인의 트리거 생성, Cloud Build > Triggers
- 5.7 Name: "hello-cloudbuild-deploy", Source: "hello-cloudbuild-env", Repository: "^candidate$"

- 5.8 CD 파이프라인을 트리거 하기 위해 CI 파이프라인 변경

  ```bash
  cd ~/hello-cloudbuild-app
  cp cloudbuild-trigger-cd.yaml cloudbuild.yaml
  ```

  ```yaml
  # cloudbuild.yaml (cloudbuild-trigger-cd.yaml)
  # [START cloudbuild]
  steps:
    # This step runs the unit tests on the app
    - name: "python:3.7-slim"
      id: Test
      entrypoint: /bin/sh
      args:
        - -c
        - "pip install flask && python test_app.py -v"

    # This step builds the container image.
    - name: "gcr.io/cloud-builders/docker"
      id: Build
      args:
        - "build"
        - "-t"
        - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/hello-cloudbuild:$SHORT_SHA"
        - "."

    # This step pushes the image to Artifact Registry
    - name: "gcr.io/cloud-builders/docker"
      id: Push
      args:
        - "push"
        - "us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/hello-cloudbuild:$SHORT_SHA"
    # [END cloudbuild]

    # [START cloudbuild-trigger-cd]
    # This step clones the hello-cloudbuild-env repository
    - name: "gcr.io/cloud-builders/gcloud"
      id: Clone env repository
      entrypoint: /bin/sh
      args:
        - "-c"
        - |
          gcloud source repos clone hello-cloudbuild-env && \
          cd hello-cloudbuild-env && \
          git checkout candidate && \
          git config user.email $(gcloud auth list --filter=status:ACTIVE --format='value(account)')

    # This step generates the new manifest
    - name: "gcr.io/cloud-builders/gcloud"
      id: Generate manifest
      entrypoint: /bin/sh
      args:
        - "-c"
        - |
          sed "s/GOOGLE_CLOUD_PROJECT/${PROJECT_ID}/g" kubernetes.yaml.tpl | \
          sed "s/COMMIT_SHA/${SHORT_SHA}/g" > hello-cloudbuild-env/kubernetes.yaml

    # This step pushes the manifest back to hello-cloudbuild-env
    - name: "gcr.io/cloud-builders/gcloud"
      id: Push manifest
      entrypoint: /bin/sh
      args:
        - "-c"
        - |
          set -x && \
          cd hello-cloudbuild-env && \
          git add kubernetes.yaml && \
          git commit -m "Deploying image us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/hello-cloudbuild:${SHORT_SHA}
          Built from commit ${COMMIT_SHA} of repository hello-cloudbuild-app
          Author: $(git log --format='%an <%ae>' -n 1 HEAD)" && \
          git push origin candidate

  # [END cloudbuild-trigger-cd]
  ```

- 5.9 변경된 CI 파이프라인의 cloudbuild.yaml 파일을 소스 저장소에 푸시
  ```bash
  cd ~/hello-cloudbuild-app
  git add cloudbuild.yaml
  git commit -m "Trigger CD pipeline"
  git push google master
  ```

#### 6. Cloud Build 파이프라인 확인

- 6.1 Cloud Build > Dashboard에서 확인

#### 7. 파이프라인의 실행 결과 확인

- 7.1 Kubernetes Engine > Services & Ingress에서 External IP 확인 후, 브라우저에서 확인
- 7.2 출력되는 메시지를 변경 후 파이프라인이 자동으로 실행되는지 확인

  ```bash
  cd ~/hello-cloudbuild-app
  sed -i 's/Hello World/Hello Cloud Build/g' app.py
  sed -i 's/Hello World/Hello Cloud Build/g' test_app.py

  git add app.py test_app.py
  git commit -m "Hello Cloud Build"
  git push google master
  ```

#### 8. 테스트 환경 롤백

- 8.1 Cloud Build > Dashboard
- 8.2 "History"에서 "hello-cloudbuild-env" 빌드 정보를 확인
- 8.3 가장 최근에서 바로 이전의 build 정보를 클릭
- 8.4 "Rebuild" 클릭
