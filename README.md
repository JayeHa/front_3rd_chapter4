- [프론트엔드 배포 파이프라인](#프론트엔드-배포-파이프라인)
  - [1.주요 링크](#1주요-링크)
  - [2. 개요](#2-개요)
    - [2.1 배포 파이프라인](#21-배포-파이프라인)
    - [2.2 GitHub Actions 워크플로우](#22-github-actions-워크플로우)
  - [3. 주요 개념](#3-주요-개념)
    - [3.1 GitHub Actions과 CI/CD 도구](#31-github-actions과-cicd-도구)
    - [3.2 S3와 스토리지](#32-s3와-스토리지)
    - [3.3 CloudFront와 CDN](#33-cloudfront와-cdn)
    - [3.4 캐시 무효화(Cache Invalidation)](#34-캐시-무효화cache-invalidation)
    - [3.5 Repository secret과 환경변수](#35-repository-secret과-환경변수)
- [\[심화과제\] CDN과 성능최적화](#심화과제-cdn과-성능최적화)
  - [1. 개요](#1-개요)
  - [2. 테스트 환경](#2-테스트-환경)
  - [3. 성능 측정 결과](#3-성능-측정-결과)
    - [3.1 네트워크 요청 분석](#31-네트워크-요청-분석)
    - [3.2 주요 성능 지표 비교](#32-주요-성능-지표-비교)
    - [3.3 응답 헤더 분석](#33-응답-헤더-분석)
    - [3.4 상세 응답 시간 분석](#34-상세-응답-시간-분석)
  - [4. 결론](#4-결론)
- [참고 자료](#참고-자료)
  - [공식문서](#공식문서)
  - [블로그](#블로그)

# 프론트엔드 배포 파이프라인

> 과제 팁1: 다이어그램 작성엔 Draw.io, [Lucidchart](https://lucid.app/documents#/documents?folder_id=home) 등을 이용합니다.

> 과제 팁2: 새로운 프로젝트 진행시, 프론트엔드팀 리더는 예시에 있는 다이어그램을 준비한 후, 전사 회의에 들어가 발표하게 됩니다. 미리 팀장이 되었다 생각하고 아키텍쳐를 도식화 하는걸 연습해봅시다.

> 과제 팁3: 캐시 무효화는 배포와 장애 대응에 중요한 개념입니다. .github/workflows/deployment.yml 에서 캐시 무효화가 어떤 시점에 동작하는지 보고, 추가 리서치를 통해 반드시 개념을 이해하고 넘어갑시다.

> 과제 팁4: 상용 프로젝트에선 DNS 서비스가 필요합니다. 도메인 구입이 필요해 본 과제에선 ‘Route53’을 붙이는걸 하지 않았지만, 실무에선 다음과 같은 인프라 구성이 필요하다는걸 알아둡시다

## 1.주요 링크

- S3 버킷 웹사이트 엔드포인트: https://hanghae99.s3-website-ap-southeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름: https://d2xvtvxlifkv9.cloudfront.net

## 2. 개요

### 2.1 배포 파이프라인

<img src="/public/diagram.png" alt="프론트엔드 배포 파이프라인 다이어그램" />

1. **Git Repository**

   - 개발자가 GitHub 저장소에 코드 푸시

2. **CI/CD 파이프라인**

   - `🐱GitHub Actions`에서 자동 빌드 및 테스트 실행
   - `🔑 AWS IAM`을 통한 리소스 접근 권한 관리

3. **빌드 결과물 배포**

   - `🪣 S3 버킷` 에 빌드 산출물 업로드
   - `🔑 AWS IAM` 정책을 통한 접근 권한 제어
     - `🪣 S3 버킷`에 대한 쓰기 권한 제공
     - `🌎 CloudFront` 배포 설정에 대한 접근 권한 관리

4. **CDN 배포** 🌎
   - `🌎 CloudFront`를 통한 전역 배포
   - 캐시 전략을 통한 성능 최적화

### 2.2 GitHub Actions 워크플로우

[GitHub Actions 워크플로우](https://docs.github.com/ko/actions/writing-workflows/quickstart)를 작성해 배포가 진행되도록 했습니다.

1. 저장소를 체크아웃합니다.

   ```yaml
   - name: Checkout repository
           uses: actions/checkout@v2
   ```

2. Node.js 18.x 버전을 설정합니다.

   ```yaml
   - name: Setup Node.js and install pnpm
     uses: actions/setup-node@v3
     with:
       node-version: 18 # Node.js 버전을 18 이상으로 설정
   - run: corepack enable
   - run: corepack prepare pnpm@latest --activate
   ```

3. 프로젝트 의존성을 설치합니다.

   ```yaml
   - name: Install dependencies
     run: pnpm install --frozen-lockfile # or npm ci
   ```

4. Next.js 프로젝트를 빌드합니다.

   ```yaml
   - name: Build
     run: pnpm build # or npm run build
   ```

5. AWS 자격 증명을 구성합니다.

   ```yaml
   - name: Configure AWS credentials
     uses: aws-actions/configure-aws-credentials@v1
     with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }} # IAM 계정 생성시 발급받은 액세스 키
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }} # IAM 계정 생성시 발급받은 비밀 액세스 키
       aws-region: ${{ secrets.AWS_REGION }} # S3를 세팅한 리전(지역)의 코드
   ```

6. 빌드된 파일을 S3 버킷에 동기화합니다.

   ```yaml
   - name: Deploy to S3
     run: |
       aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
   ```

7. CloudFront 캐시를 무효화합니다.

   ```yaml
   - name: Invalidate CloudFront cache
     run: |
       aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
   ```

## 3. 주요 개념

### 3.1 GitHub Actions과 CI/CD 도구

GitHub Actions는 자동화된 워크플로우를 생성하여 CI/CD(Continuous Integration/Continuous Deployment)를 관리할 수 있는 도구입니다.
이를 통해 코드 변경 시마다 자동으로 빌드, 테스트, 배포를 수행합니다.
GitHub Actions는 배포 파이프라인을 관리하는 데 중요한 역할을 합니다.

### 3.2 S3와 스토리지

AWS S3는 정적 파일을 저장하고 제공하는 스토리지 서비스입니다.
이 서비스를 사용하여 빌드된 파일들을 안전하게 저장하고, 클라이언트에 제공할 수 있습니다.

### 3.3 CloudFront와 CDN

CloudFront는 AWS의 콘텐츠 전송 네트워크(CDN) 서비스로, 전 세계에 분산된 서버를 통해 파일을 빠르게 전달합니다.
CloudFront는 파일 캐시를 관리하고, 사용자가 파일을 더 빠르게 접근할 수 있도록 최적화합니다.

### 3.4 캐시 무효화(Cache Invalidation)

CloudFront의 캐시를 무효화하는 과정은 배포 후 반드시 수행되어야 합니다.
새로 배포된 파일이 캐시로 인해 사용자에게 전달되지 않는 문제를 방지하고, 최신 파일을 제공할 수 있도록 합니다.
이 과정은 자동화된 배포 프로세스의 중요한 부분입니다.

### 3.5 Repository secret과 환경변수

GitHub에서 제공하는 `repository secret`과 `환경 변수`를 사용하여 민감한 정보를 안전하게 관리할 수 있습니다.
이를 통해 AWS 자격 증명, API 키 등 중요한 정보를 코드에 포함시키지 않고 안전하게 저장하고 사용할 수 있습니다.

---

# [심화과제] CDN과 성능최적화

> 과제 팁1 : CDN 도입후 성능 개선 보고서 영역은 [프론트엔드 개발자를 위한 CloudFront](https://sprout-log-68d.notion.site/CloudFront-2c0653cb130f42b2b21078389511cca2) 에서 네트워크 탭을 비교한 영역을 참고해주세요. 이미지와 수치등을 표로 정리해서 보여주면 가독성이 높은 보고서가 됩니다.

> 과제 팁2 : 저장소 → 스토리지 → CDN을 통해 정적파일을 배포하는 방식을 이해하지 못하면 다양한 기술 문서를 이해하지 못합니다. 링크로 첨부한 문서를 보고 실무에서 이런 네트워크 지식이 어떻게 쓰이는지 맛보기 해보세요.

## 1. 개요

본 보고서는 정적 웹 사이트 배포 시 Amazon CloudFront 도입 전후의 성능을 비교 분석한 결과입니다.

## 2. 테스트 환경

- 테스트 도구: Chrome DevTools Network 탭
- 측정 방식: 시크릿 모드에서 4회 반복 측정
- 테스트 대상:
  - CDN 도입 전: S3 버킷 직접 호스팅
  - CDN 도입 후: CloudFront 배포

## 3. 성능 측정 결과

### 3.1 네트워크 요청 분석

네트워크 탭을 통해 컨텐츠의 전반적인 파일 사이즈와 응답속도를 비교했습니다.

<img src="/public/네트워크 탭 비교.png" />

→ CDN 도입 후(오른쪽)의 파일 사이즈와 응답속도가 현저히 개선된 것을 확인할 수 있습니다.

### 3.2 주요 성능 지표 비교

[네트워크 패널 하단의 상대표시줄](https://developer.chrome.com/docs/devtools/network/reference?utm_source=devtools&hl=ko#timing-explanation)을 통해 주요 성능 지표를 비교했습니다.

<img src="/public/네트워크 탭 하단 상태표시줄 비교.png" />

이를 표로 나타내면 다음과 같습니다.

| 지표                  | 도입 전 | 도입 후 | 개선율     |
| --------------------- | ------- | ------- | ---------- |
| 전송된 리소스 총 크기 | 564 kB  | 282 kB  | 50% 감소   |
| 완료 시간             | 7.02초  | 6.49초  | 7.5% 감소  |
| DOMContentLoaded      | 540ms   | 99ms    | 81.7% 감소 |
| 로드 시간             | 1.21초  | 210ms   | 82.6% 감소 |

### 3.3 응답 헤더 분석

<img src="/public/응답 헤더 비교.png" />

주요 개선사항:

1. **컨텐츠 압축 적용**: Content-Encoding 헤더를 통해 gzip 압축 확인
2. **캐시 적용**: X-Cache 헤더 존재로 캐시 동작 확인

### 3.4 상세 응답 시간 분석

<img src="/public/요청의 타이밍 분석 비교.png" />

시크릿 창에서 4회 반복 측정 결과:

**도입 전 측정 결과**
<img src="/public/요청의 타이밍 분석 - 도입 전.png" />

- 1차: 339.79ms
- 2차: 179.09ms
- 3차: 168.56ms
- 4차: 323.13ms
- **평균: 227.14ms**

**도입 후 측정 결과**
<img src="/public/요청의 타이밍 분석 - 도입 후.png" />

- 1차: 53.92ms
- 2차: 13.25ms
- 3차: 20.05ms
- 4차: 13.90ms
- **평균: 25.55ms**

→ 응답 시간 88.8% 개선

## 4. 결론

CloudFront CDN 도입으로 다음과 같은 주요 성능 개선을 달성했습니다:

1. 페이지 로드 시간 82.6% 감소
2. 전송 데이터 크기 50% 감소
3. 평균 응답 시간 88.8% 개선

특히 주목할 만한 점은 모든 요청에서 도입 전에는 최소 100ms 이상이 소요되었으나, 도입 후에는 모든 요청이 100ms 미만으로 처리되었다는 점입니다.

---

# 참고 자료

## 공식문서

- [Amazon S3 사용 설명서](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Amazon CloudFront 개발자 안내서](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [IAM이란 무엇입니까?](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/introduction.html)
- [GitHub Actions에 대한 워크플로 구문](https://docs.github.com/ko/actions/writing-workflows/workflow-syntax-for-github-actions#about-yaml-syntax-for-workflows)

## 블로그

- [웹프론트개발팀에서 배민 커머스 어드민을 개발하는 방법](https://techblog.woowahan.com/15084/)
- [프론트엔드 CI/CD 파이프라인 구축](https://velog.io/@taegon1998/%ED%94%84%EB%A1%A0%ED%8A%B8%EC%97%94%EB%93%9C-CICD-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EA%B5%AC%EC%B6%95)
- [팀에 맞는 프론트엔드 CI/CD 파이프라인 설계 및 배포 자동화 구축하기](https://medium.com/@dlxotjde_87064/%ED%94%84%EB%A1%A0%ED%8A%B8%EC%97%94%EB%93%9C-ci-cd-%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EC%84%A4%EA%B3%84-%EB%B0%8F-%EB%B0%B0%ED%8F%AC-%EC%9E%90%EB%8F%99%ED%99%94-c84c973ce45d)
