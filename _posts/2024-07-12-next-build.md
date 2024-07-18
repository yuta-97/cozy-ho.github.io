---
layout : post
title : Next build의 결과물
date : 2024-07-12 10:59:23
tags :
- nextjs
category: [Frontend, nextjs]
---

# Intro
Next.js를 사용 해 개발을 하고, 배포를 하려면 build 과정을 거쳐야 한다.
[공식문서에서 안내](https://nextjs.org/docs/pages/building-your-application/deploying#self-hosting)하는 배포 방법에는 3가지가 있었다.

![image](https://github.com/user-attachments/assets/4c1fdf17-50ba-4b4e-9d26-2f8888bad613)

1. 배포된 Node.js 환경에서 실행
2. Docker 환경으로 배포
3. Static 파일로 배포

1, 2번 방식은 모두 Node 환경에서 실행하기 때문에 배포환경으로 보자면 두가지 방법인 셈이다.

# Node 환경
기본으로 제공되는 설정에서 어떤것도 수정 할 필요 없이 `next build`를 하게되면 최상위에 `.next` 라는 폴더 하위로 빌드 결과물이 생성된다.

해당 디렉토리 안의 내용을 좀 살펴보면 다음과 같다

![image](https://github.com/user-attachments/assets/894f5c68-e7a0-4890-97dd-6284248bd958)

각각 이름들은 직관적으로 cache, server, static 이다.

### cache
해당 폴더에는 빌드에 사용된 캐시와 이미지 최적화, 페이지 정보 등에 사용되는 캐시들이 관리된다.
이름이 캐시인 만큼 이후 빌드 과정에서 변경사항이 없다면 여기에 있는 정보들이 재 사용 된다.

### server
Nextjs 는 SSR에 최적화 되어있는 프레임워크인 만큼 관련 기능을 위한 파일들이 여기에 구성된다.
서버사이드 렌더링, 미들웨어, API 엔드포인트 등 서버에서 처리되어야 할 기능들의 코드가 들어간다.

### static
여기에는 정적인 데이터들을 관리한다. 이미지 파일, 폰트, CSS, JS 파일 등 으로 구성된다. Next는 `code-spliting`을 자체적으로 지원하기 때문에 각 파일들을 적당하게 나눠서 최적화 작업을 거친 후 규칙에 맞게 분류해 관리한다.

### etc
그 이외에도 `build-manifest.json` 등의 파일도 있는데, 해당 파일은 사용자가 어떤 페이지를 요청했을 때 필요한 staic-file 들의 정보가 정의되어 있다.
Next가 최적화를 위해 조각 내 놓은 js 청크들을 페이지 요청시 알맞게 제공하기 위한 설정파일인 것.

# Static export
node.js 의 도움 없이 렌더링 할 수 있도록 한다.
`next.config.json` 파일에 `output: "export"` 로 설정하면 빌드 결과물이 달라진다.

정적 파일로 만들어서 배포하는 방식이다. 정통적인 배포 방식으로 `Nginx` 등의 별도의 웹 서버를 통해 제공한다.
앱의 각 페이지별로 HTML, CSS, JS 파일을 번들링한다.

기본적으로 빌드 결과물은 호스트의 root 경로 `/` 에서 serve 된다고 가정하기 때문에 모든 경로가 절대경로 기준이다. 때문에 다른 경로로 호스팅이 필요하다면 해당 설정을 수정 해야 한다. 예를들면 `S3`나 `CDN`을 통해 배포하는 경우 별도 URL 을 입력해야 한다.

해당 설정은 `next.config.json` 내에 `assetPrefix` 필드에 입력하면 된다. 경로를 입력하거나 URL 형식의 입력이 가능하다.
