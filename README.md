# 프론트엔드 배포 파이프라인

## 개요

- 다이어그램 작성 (Draw.io)

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다.

1. 저장소를 체크아웃합니다.
2. Node.js 18.x 버전을 설정합니다.
3. 프로젝트 의존성을 설치합니다.
4. Next.js 프로젝트를 빌드합니다.
5. AWS 자격 증명을 구성합니다.
6. 빌드된 파일을 S3 버킷에 동기화합니다.
7. CloudFront 캐시를 무효화합니다.

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghae99.s3-website-ap-southeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름: https://d2xvtvxlifkv9.cloudfront.net

## 주요 개념

- GitHub Actions과 CI/CD 도구:
- S3와 스토리지:
- CloudFront와 CDN:
- 캐시 무효화(Cache Invalidation):
- Repository secret과 환경변수:

> 과제 팁1: 다이어그램 작성엔 Draw.io, Lucidchart 등을 이용합니다.

> 과제 팁2: 새로운 프로젝트 진행시, 프론트엔드팀 리더는 예시에 있는 다이어그램을 준비한 후, 전사 회의에 들어가 발표하게 됩니다. 미리 팀장이 되었다 생각하고 아키텍쳐를 도식화 하는걸 연습해봅시다.

> 과제 팁3: 캐시 무효화는 배포와 장애 대응에 중요한 개념입니다. .github/workflows/deployment.yml 에서 캐시 무효화가 어떤 시점에 동작하는지 보고, 추가 리서치를 통해 반드시 개념을 이해하고 넘어갑시다.

> 과제 팁4: 상용 프로젝트에선 DNS 서비스가 필요합니다. 도메인 구입이 필요해 본 과제에선 ‘Route53’을 붙이는걸 하지 않았지만, 실무에선 다음과 같은 인프라 구성이 필요하다는걸 알아둡시다

## CDN과 성능최적화

(CDN 도입 전과 도입 후의 성능 개선 보고서 작성)

> 과제 팁1 : CDN 도입후 성능 개선 보고서 영역은 [프론트엔드 개발자를 위한 CloudFront](https://sprout-log-68d.notion.site/CloudFront-2c0653cb130f42b2b21078389511cca2) 에서 네트워크 탭을 비교한 영역을 참고해주세요. 이미지와 수치등을 표로 정리해서 보여주면 가독성이 높은 보고서가 됩니다.

> 과제 팁2 : 저장소 → 스토리지 → CDN을 통해 정적파일을 배포하는 방식을 이해하지 못하면 다양한 기술 문서를 이해하지 못합니다. 링크로 첨부한 문서를 보고 실무에서 이런 네트워크 지식이 어떻게 쓰이는지 맛보기 해보세요.
