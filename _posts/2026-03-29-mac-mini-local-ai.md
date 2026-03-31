---
title: "[Infra] Mac Mini M4에 Ollama(Qwen 2.5) & OpenClaw 로컬 AI 비서 구축기"
date: 2026-03-29 21:42:00 +0900
categories: [🤖 AI, Openclaw]
tags: [mac-mini, m4, ollama, qwen-2.5, openclaw, telegram-bot, local-ai]
---

보안 업무를 수행하다 보면 보안 로그나 취약점 분석 데이터를 외부 API(ChatGPT, Claude 등)에 전송하는 것이 망설여질 때가 많습니다. 민감한 데이터 유출(Data Leakage) 위험 때문입니다. 

이를 해결하기 위한 가장 확실한 방법은 폐쇄망에서도 돌아가는 완벽한 **로컬 LLM(Large Language Model) 환경**을 구축하는 것입니다. 최근 도입한 **Mac Mini(M4, 32Gb 메모리)**의 강력한 통합 메모리 환경을 활용하여, 가벼우면서도 한국어 성능이 뛰어난 **Qwen 2.5 (14B)** 모델과 **OpenClaw**를 연동해 나만의 로컬 AI 비서를 구축한 과정을 공유합니다.

---

## 1. 환경 준비 및 Ollama 설치
---
Apple Silicon(M시리즈) 맥은 통합 메모리 구조 덕분에 VRAM 부족 현상 없이 꽤 큰 규모의 모델을 원활하게 구동할 수 있습니다. LLM 구동을 위해 가장 직관적이고 가벼운 프레임워크인 `Ollama`를 사용했습니다.

### Ollama 설치 및 Qwen 2.5 다운로드

먼저 터미널을 열고 Homebrew를 통해 Ollama를 설치합니다.

```bash
brew install ollama
# 또는 공식 홈페이지에서 Mac 버전을 직접 다운로드하여 설치
````

설치가 완료되면 터미널에서 아래 명령어를 통해 Qwen 2.5 14B 파라미터 모델을 내려받고 실행합니다. 14B 모델은 32GB 메모리에서 속도와 추론 능력의 밸런스가 가장 훌륭했습니다.
![QWEN 2.5 설치 사진](/assets/img/posts/20260329/qwen 설치.png)
_ollama에서 qwen 2.5:14B 로컬 ai 설치_

```bash
ollama run qwen2.5:14b
```	

> **💡 Note:** 첫 실행 시 모델의 용량(약 8\~9GB)만큼 다운로드가 진행됩니다. 프롬프트 창이 뜨면 한국어로 질문을 던져 정상적으로 추론이 이루어지는지 확인합니다.
> {: .prompt-info }

![QWEN 2.5 확인 사진](/assets/img/posts/20260329/ollama 질문.png)

-----

## 2\. OpenClaw 설치 및 사전 요구 사항(Prerequisites)
---
### ⚙️ 사전 요구 사항 (Node.js 및 Python)
OpenClaw는 패키지 관리를 Python 환경에서 진행하지만, 내부적으로 에이전트 구동 및 비동기 이벤트 처리를 위해 Node.js 런타임을 강하게 요구합니다. 따라서 설치 전 필수 패키지들을 먼저 세팅해야 합니다.

Mac 환경에서는 Homebrew를 이용해 간단히 설치할 수 있습니다.
```
# Node.js 설치 (OpenClaw 런타임 요구사항)
brew install node

# 정상 설치 확인 (버전이 출력되면 성공)
node -v
npm -v

# Python 설치 (이미 최신 버전이 있다면 생략 가능)
brew install python
```
Ollama 단독으로도 훌륭하지만, 이를 실제 '비서'처럼 활용하려면 메신저와 연동하고 다양한 도구(Tool)를 쥐여줄 수 있는 프레임워크가 필요합니다. 이를 위해 **OpenClaw**를 도입했습니다.

### OpenClaw 초기 세팅

Node.js가 준비되었다면, 터미널에서 npm을 통해 OpenClaw를 설치해 줍니다.

```bash
# OpenClaw 전역(global) 설치
npm install -g openclaw
```

설치 후, OpenClaw의 설정 파일에서 LLM 백엔드를 우리가 방금 설치한 로컬 Ollama(Qwen 2.5 14B)로 지정해 줍니다. API 키가 필요 없다는 것이 로컬 환경의 가장 큰 장점입니다.

![Openclaw Ollama 설정](/assets/img/posts/20260329/Openclaw Ollama 설정.png)
_LLM 백엔드를 로컬 Ollama로 설정_

-----

## 3\. Telegram 봇 연동 및 트러블슈팅

언제 어디서든 로컬 AI에 접근하여 질문을 던질 수 있도록 텔레그램(Telegram) 메신저와 연동을 진행했습니다.


1.  텔레그램에서 `BotFather`를 검색해 새로운 봇을 생성하고 **API Token**을 발급받습니다.

    ![Botfather 검색](/assets/img/posts/20260329/Botfather.jpg){: width="500"}
    _Telegram에서 Botfather 검색 후 open_

    ![Botfather 생성 1](/assets/img/posts/20260329/make bot.jpg){: width="500"}
    _Create a New Bot 선택으로 봇 생성_

    ![Botfather 생성 2](/assets/img/posts/20260329/name bot.jpg){: width="500"}
    _생성할 Bot 이름 설정 후 봇 생성_

    ![Telegrem 연동 Key](/assets/img/posts/20260329/copy key.jpg){: width="500"}
    _생성한 Bot의 Key 복사_

2. 생성한 내 텔레그램 봇 채팅방에 들어가 /start 명령어를 입력합니다.

3. 봇이 인가된 사용자인지 확인하기 위해 **페어링 코드(Pairing Code)**를 요구합니다. 이때 Mac 터미널(OpenClaw 실행 로그)에 출력되어 있는 고유 페어링 코드를 텔레그램 채팅창에 그대로 입력해 줍니다.

    ![Telegram 페어링 코드](/assets/img/posts/20260329/pairing code.jpg){: width="500"}
    _생성한 봇 채팅창에 /start 입력 시 페어링 코드 확인 가능_

### ⚠️ 트러블슈팅: OpenClaw Agent Binding 에러

Mac Mini 환경에서 텔레그램 연동 시, 봇이 응답하지 않거나 특정 에이전트 바인딩(Agent Binding) 명령어가 제대로 먹히지 않는 이슈를 겪었습니다.

확인해 보니 LLM의 컨텍스트를 유지하는 에이전트 세션과 텔레그램의 Chat ID가 매핑되는 과정에서 충돌이 발생한 것이 원인이었습니다. 이 경우 OpenClaw를 실행할 때 포그라운드(Foreground)가 아닌 백그라운드 서비스로 등록하고, 바인딩 명령어를 명시적으로 다시 전달해주어야 합니다.

```bash
# Agent 바인딩 초기화 및 텔레그램 리스너 재시작 명령어 예시
claw-cli agent unbind --all
claw-cli agent bind --model qwen2.5:14b --channel telegram
```

위 과정을 거치고 나니 텔레그램에서 질문했을 때 Mac Mini에서 추론을 거쳐 즉각적으로 답변이 오는 것을 확인할 수 있었습니다.

*(스마트폰이나 PC 텔레그램에서 구축한 AI 봇과 대화하는 화면 캡처)*

-----

## 🎯 마무리

이번 세팅을 통해 **외부 네트워크 의존 없이 안전하게 동작하는 든든한 로컬 AI 인프라**를 확보했습니다. M4 칩의 NPU를 활용한 Qwen 2.5 14B의 성능은 기대 이상으로 쾌적합니다.

이제 이 로컬 AI 비서에게 단순한 질의응답을 넘어, 24시간 비서의 역할을 할 수 있도록 수정해야 할 것 같습니다.

---