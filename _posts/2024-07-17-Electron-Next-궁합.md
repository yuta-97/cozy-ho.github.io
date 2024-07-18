---
layout : post
title : Electron 과 Nextjs
description: Electron 과 Next의 궁합은 좋지 않다
date : 2024-07-17 10:59:23
tags :
- electron
- nextjs
category: [Frontend]
---

# Intro
Electron과 함께 사용할 renderer 쪽 프레임워크로 Next.js를 사용하는 것은 그리 좋지 않은 선택이었다는 것을 알게된 과정을 따라가며 Next와 Electron에 대한 소소한 정보들을 챙겨보자.

Electron 으로 만들어보고 싶은 것이 생겨서 사용할 기술들을 고르고 있었다.
처음 생각한 기술스택은 다음과 같다.

> Next.js + Electron + Typescript
{: .prompt-info }

가장 친근하기도 하고 자주 사용하던 프레임워크들이라 선택했다.

앱 패키징은 `electron-builder` 를 사용했다. 이런저런 패키지 관련 설정과 Next 설정파일 등등을 구성하고 로컬 실행시에는 아무 문제가 없었다.

Electron 프로세스 따로, 로컬에서 도는 Next 따로 실행 시켰을 경우에는 괜찮았던 것이 Package 과정을 거쳐 앱으로 만들고 실행 시키면 정상적으로 화면이 보이지 않았다.

![image](https://github.com/user-attachments/assets/a519437a-7076-43e6-bbdb-ea8ba4cd7abb)
> 뭐야 내 화면 돌려줘요

# 의문점

## Electron 문제인가?
`next.config.js`에 설정한 항목은 다음과 같다.
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
	output: "export",
	distDir: "build",
	swcMinify: true
};

export default nextConfig;
```

![image](https://github.com/user-attachments/assets/805d51c8-49f6-4191-9483-9b4b003dc1d6)

배포 된 앱의 로그를 보면 소스파일 경로를 찾지 못하는 문제가 있다.

![image](https://github.com/user-attachments/assets/fbd25bd6-af4a-485d-8e9e-81eeb08ee35e)

음.. 하지만 참조하는 js, css 등의 파일 경로는 올바르게 들어 간 것으로 보인다.
그렇다면 패키징 과정에서 혹시 `index.html`을 제외한 나머지 asset들이 누락이 된 걸까?

정말 그런지 확인 해 보려면 패키징 된 앱을 까봐야 알 것 같다.
Electron 앱은 패키징 과정에서 resource 들이 하나의 `asar`포맷의 파일로 합쳐져서 들어가기 때문에(일종의 압축파일이다), 확인 하려면 해당 파일을 찾아서 unpack하는 과정을 거쳐야 한다.

방법은 다음과 같다.

```shell
// 설치
npm install -g @electron/asar
// test 경로에 unpack
npx @electron/asar extract <asar 파일 경로> ./test
```

![image](https://github.com/user-attachments/assets/5123ee18-3709-4660-a901-1b9f06320fa2)

확인한 결과 `asar`파일에도 정상적으로 모든 데이터가 들어 간 것을 확인했다.

파일은 문제가 없었다. 그렇다면 대체 뭐가 문제일까?

## 원인을 찾다
개발 환경에서는 `https://localhost:port`형식으로 넘겨주겠지만, 배포환경에서는 빌드된 결과의 index.html 파일을 넘겨준다.

위에 첨부한 소스인 Next에서 빌드한 결과 index.html 파일을 보면 파일 경로가 모두 `/_next/`로 시작한다는 걸 알 수 있다. 패키지 된 앱에서의 절대경로를 인식하지 못하는 문제였다.

빌드타임 에서의 root 경로와 패키지 된 app 에서의 root 경로가 다르기 때문에, 해당 경로로 파일을 가져올 수 없는 것.

~~경로 문제라는 것을 알았으니 쉽게 해결 될 줄 알았다.~~

검색을 해 보니 나와 비슷한 문제를 겪는 사람들이 몇몇 보였다.
- [Static CSS import fails with ERR_FILE_NOT_FOUND in production build](https://github.com/vercel/next.js/discussions/13578)
- [Generated static files html files have wrong assets paths](https://github.com/vercel/next.js/issues/8158)

해당 issue, discussion 에서는 모두 `next.config.js` 의 `assetPrefix` 를 "./" 등 상대경로로 사용하여 해결하라는 답변이 있었다.

하지만 어쨰서인지 현재 Next버전(v14.x) 에서는 해당 필드의 값으로 받아주지 않았다..
![image](https://github.com/user-attachments/assets/6f03fb59-82f3-4d76-a62e-82ff4ac2dc6a)

다시 검색해 보니 나름 최신(2023)에 등록된 이슈가 있었다.

- [Could not use relative URLs with "next/font"](https://github.com/vercel/next.js/issues/52050#issuecomment-1813323300)

안타깝게도 유일해 보이는 해결책은 index.html 내에 `/_next/` 로 되어있는 부분들을 수동으로, 또는 스크립트를 작성하여 직접 변경(...) 하는 방법 이었다.

![image](https://github.com/user-attachments/assets/953d03bf-8242-43d9-9726-e9a2a90499d2)
> 잘 나오긴 하네..

> static export 로 빌드한 결과물에서는 Next 서버가 없기 때문에 서버사이드의 기능들은 사용 할 수 없다. 그래서 위의 이미지에서도 Next/Image 가 제대로 보이지 않는 것.
{: .prompt-info }

## 이게맞나?
아무리 생각해도 임시방편에 불과한 이런 방법이 마음에 들지 않았다. 역시 프레임워크를 사용하면 편한만큼 자유도는 줄어드는구나.. 대체 왜 `assetPrefix`에 상대경로를 설정하지 못하도록 변경한걸까..

경로문제를 깔끔하게 해결하려면 renderer 프로세스도 Node 위에서 돌리면 된다. 그런데 Electron 도 node 런타임에서 돌아가는데 Next를 실행시키기 위해 불필요한 node 프로세스가 추가 된다. 빌드된 앱의 크기가 너무 커지기도 하고, 속도도 느릴 것이 뻔했다.

Next의 장점은 web 환경에서의 퍼포먼스이지 (cache, ssr ..) 이게 로컬에서 도는 native-app 에서는 의미없다 라는 생각이 들었고, 굳이 Next를 고집할 이유가 있을까? 하는 고민에 이르렀다.

# 해결? 결론
그냥 기본 React 를 사용하기로 했다. 내가 만드려는 앱의 기능과 동작이 중요하지, 그걸 만드는데 필요한 도구들이 일으키는 문제들에 집중하고 싶지 않았다.

Electron 앱을 만들 때 생각보다 Electron 자체 설정과 패키징을 위한 electron-builder 관련 설정 등 자잘하게 신경써야 할 부분들이 많다. 뭐 하나 잘못되면 익숙하지 않은 탓에 디버깅 하기 쉽지 않기도 하고, 빠른 속도감이 없으면 금새 흥미가 떨어져 흐지부지 되어 버리는게 싫었다.

그래서 사용한 scaffold
```console
pnpm create @quick-start/electron . --template react-ts
```

`Electron`, `React.js`, `Typescript` 스택으로 완벽하게 기본 세팅을 해준다. 우리는 이제 무엇을 만들 것인가에 집중하면 된다.
> 속이 뻥..!

# Outro
Next 로 패키징 했을떄는 앱의 크기가 약 650MB 정도였지만 vite + react 가 확실히 가볍긴 한가보다. 크기가 250MB 로 거의 1/3 크기로 줄었다.

처음에 직접 만들었을 때의 디렉토리 구조와 다른 점들이 있다. 물론 이것도 모범 답안은 아니지만 구조를 잡아둔 방식을 보면서 배운것들도 있었다.

어떤 기술이나 툴을 처음 배울 때 다들 프레임워크나 자동화 툴 등을 사용하지 않고 직접 구현해 보는게 좋다고 한다. 나도 처음부터 휘리릭 생성해주는 구조에 익숙해져서 편하게 사용했다면 고민의 깊이가 달랐을 것이다. 역시 삽질한 만큼 보이는구나.
