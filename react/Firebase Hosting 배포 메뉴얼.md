
Firebase CLI 설치
```shell
sudo npm install -g firebase-tools
```

Firebase CLI  로그아웃
```shell
firebase logout
```

Firebase CLI 로그인
```shell
firebase login
```

Firebase 프로젝트 리스트 출력
사용자 계정에 접근이 되는지 테스트하기 위해서 다음 명령어를 사용하여 프로젝트 리스트를 출력해봅니다.
```shell
firebase projects:list
```
![](../images/Pasted%20image%2020260305120817.png)

Firebase CLI 제거
Firbase CLI를 제거하고 싶다면 다음 명령어를 사용하면 됩니다.
```shell
npm uninstall -g firebase-tools
```


Firebase 프로젝트 생성
- Firbase Hosting 콘솔로 이동하여 프로젝트를 생성함
![](../images/Pasted%20image%2020260306112505.png)

Firebase 프로젝트 초기화
- Hosting
- Use an existing project 선택
```shell
firebase init
```

Hosting Setup은 다음 화면과 같이 입력하여 설정합니다.
![](../images/Pasted%20image%2020260305123118.png)

firebase.json
```json
{
  "hosting": {
    "public": "build",
    "ignore": [
      "firebase.json",
      "**/.*",
      "**/node_modules/**"
    ],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ]
  }
}

```

.firebaserc
```json
{
  "projects": {
    "default": "invest72"
  }
}
```

### CI 환경을 위한 '애플리케이션 기본 자격 증명(ADC)' 설정
Setup 1 서비스 계정 키 생성
1: Google Cloud 콘솔 접속
2: IAM 관리자 -> 서비스 계정 클릭
3: 서비스 계정 만들기 클릭 (이름: firebase-deploy-sa)
4: 역할 부여: Firebase Hosting 관리자 역할 추가
5: 생성된 계정을 클릭하고 key 탭 -> 새 키 만들기 -> json 선택후 다운로드

![](../images/Pasted%20image%2020260305123530.png)
![](../images/Pasted%20image%2020260305123547.png)


Setup 2 CI 환경에 등록 (Github Actions 기준)
1: Github 저장소의 Settings -> Secrets and variables > Actions 이동
2: New repository secret 클릭
- name: FIREBASE_SERVICE_ACCOUNT_JSON
- value: 다운로드 받은 JSON 파일의 내용을 통째로 복사해서 붙여넣기
![](../images/Pasted%20image%2020260305123845.png)

### CI 워크플로우 작성(.github/workflows/deploy.yaml)
```yaml
name: Deploy to Firebase Hosting
on:
  push:
    branches:
      - main

jobs:
  ci-cd:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repo
        uses: actions/checkout@v4
      - name: Install Dependencies
        run: npm install
      - name: Build Project
        run: npm run build
        env:
          REACT_APP_API_BASE_URL: ${{ secrets.REACT_APP_API_BASE_URL }}
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
          project_id: invest72
      - name: Deploy to Firebase Hosting
        run: |
          npx firebase-tools deploy --only hosting --project invest72
```


Github Secret에 시크릿 정보 저장
- REACT_APP_API_BASE_URL 환경변수를 추가하여 빌드시 주입하도록 합니다.
![](../images/Pasted%20image%2020260306112307.png)

푸시하여 CI 시스템 이용
코드를 수정한후 main 브랜치에 코드를 푸시하게 되면 Github Action이 수행하여 Firebase에 자동적으로 배포하게 됩니다.
![](../images/Pasted%20image%2020260306112710.png)


