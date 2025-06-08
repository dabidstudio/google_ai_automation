# 링크를 보내면 슬라이드를 생성해주는 텔레그램 봇 만들기

## 프로그램 소개

이 프로그램은 **텔레그램으로 웹사이트 링크를 보내면 해당 내용을 분석하여 Google 슬라이드 형식의 발표 자료를 자동 생성하는 Google Apps Script 기반 텔레그램 봇**입니다.
GPT 모델을 활용해 웹페이지 내용을 요약하고, Google Slides API를 사용하여 테마가 적용된 발표 자료를 자동으로 구성합니다.

---

## 실행 방법

### 1. 각종 키 입력

* **TELEGRAM\_TOKEN**: [봇 토큰 발급 가이드](https://github.com/dabidstudio/dabidstudio_guides/blob/main/get_telegram_token.md)를 참고해 발급 후 입력
* **OPENAI\_API\_KEY**: [OpenAI 키 발급 가이드](https://github.com/dabidstudio/dabidstudio_guides/blob/main/get-openai-api-key.md)를 참고해 입력
* **TEMPLATE\_SLIDE\_ID**: Google Slides에서 사용할 **슬라이드 템플릿 파일의 ID**를 복사해 입력

  * 템플릿은 원하는 테마를 설정한 후 Slides에서 파일 열기 → URL 중 `.../d/파일ID/edit` 형태의 `파일ID` 사용

---

### 2. Google Apps Script 열기

1. [https://script.new](https://script.new) 접속
2. 아래 코드 전체를 붙여넣기

```javascript
const TELEGRAM_TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN';
const OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY';
const TELEGRAM_API_URL = 'https://api.telegram.org/bot' + TELEGRAM_TOKEN;
const TEMPLATE_SLIDE_ID = 'YOUR_TEMPLATE_SLIDE_ID';  // 슬라이드 템플릿 ID 입력

function doPost(e) {
  const contents = JSON.parse(e.postData.contents);
  const message = contents.message;
  const chatId = message.chat.id;
  const text = message.text;

  const urlRegex = /(https?:\/\/[^\s]+)/;
  const match = text.match(urlRegex);

  if (match && match[1]) {
    const websiteUrl = match[1];
    const pageText = fetchWebsiteText(websiteUrl);
    const structuredSlides = generateSlideStructure(pageText);
    const slideUrl = createGoogleSlides(structuredSlides, chatId);
    sendMessage(chatId, `슬라이드가 생성되었습니다: ${slideUrl}`);
  } else {
    sendMessage(chatId, '홈페이지 링크를 보내주세요.');
  }
}

function fetchWebsiteText(url) {
  try {
    const response = UrlFetchApp.fetch(url);
    const html = response.getContentText();
    const text = html.replace(/<[^>]*>/g, ' ').replace(/\s+/g, ' ').trim();
    return text.slice(0, 4000); // GPT 입력 제한
  } catch (error) {
    return '웹사이트 내용을 가져오는 중 오류가 발생했습니다.';
  }
}

function generateSlideStructure(text) {
  const prompt = `
다음 내용을 기반으로 5개의 슬라이드 구성안을 만들어 주세요.
각 슬라이드는 다음 형식을 따릅니다:
- 핵심 주제 (1문장)
- 그 주제를 뒷받침하는 근거 또는 설명 (3문장)

출력 예시:
슬라이드 1:
제목: 슬라이드 주제
내용:
1. 근거1
2. 근거2
3. 근거3

내용:
${text}`;

  const payload = {
    model: 'gpt-4o',
    messages: [
      { role: 'system', content: '당신은 슬라이드 구조를 잘 잡는 요약가입니다.' },
      { role: 'user', content: prompt }
    ],
    temperature: 0.7
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    headers: {
      'Authorization': 'Bearer ' + OPENAI_API_KEY
    },
    payload: JSON.stringify(payload)
  };

  const response = UrlFetchApp.fetch('https://api.openai.com/v1/chat/completions', options);
  const data = JSON.parse(response.getContentText());
  return data.choices[0].message.content.trim();
}

function createGoogleSlides(slideText, chatId) {
  try {
    const copiedFile = DriveApp.getFileById(TEMPLATE_SLIDE_ID).makeCopy('자동 생성된 슬라이드');
    const presentation = SlidesApp.openById(copiedFile.getId());
    sendMessage(chatId, '슬라이드 프레젠테이션을 생성했습니다.');

    while (presentation.getSlides().length > 0) {
      presentation.getSlides()[0].remove();
    }

    const slideChunks = slideText.split(/슬라이드\s*\d+:/).filter(s => s.trim() !== '');
    slideChunks.forEach((slide, index) => {
      const titleMatch = slide.match(/제목:\s*(.+)/);
      const contentMatches = slide.match(/(?:\d\.\s*)(.+)/g);

      const title = titleMatch ? titleMatch[1].trim() : '제목 없음';
      const contents = contentMatches ? contentMatches.map(c => c.replace(/^\d\.\s*/, '')) : [];

      const newSlide = presentation.appendSlide(SlidesApp.PredefinedLayout.TITLE_AND_BODY);
      newSlide.getPlaceholder(SlidesApp.PlaceholderType.TITLE).asShape().getText().setText(title);
      newSlide.getPlaceholder(SlidesApp.PlaceholderType.BODY).asShape().getText().setText(contents.join('\n'));

      sendMessage(chatId, `슬라이드 ${index + 1} 생성 완료`);
    });

    const slideUrl = presentation.getUrl();
    sendMessage(chatId, `슬라이드가 모두 생성되었습니다! 링크: ${slideUrl}`);
    return slideUrl;

  } catch (error) {
    sendMessage(chatId, `슬라이드 생성 중 오류가 발생했습니다: ${error.message}`);
    throw error;
  }
}

function sendMessage(chatId, text) {
  const url = TELEGRAM_API_URL + '/sendMessage';
  const payload = {
    chat_id: chatId,
    text: text
  };

  const options = {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload)
  };

  UrlFetchApp.fetch(url, options);
}
```

---

### 3. 앱 배포 (웹훅용)

1. 상단 메뉴 → **"배포 > 웹 앱으로 배포"** 클릭
2. 설정

   * **설명**: 웹 링크 → 슬라이드 생성기
   * **앱 실행 권한**: 본인 계정
   * **액세스 대상**: *익명 사용자(Anyone)*
3. **"새 버전으로 배포"** 클릭 → **웹 앱 URL 복사**

**중요: 코드를 수정한 경우, 반드시 다시 배포(새 버전 만들기)를 해야 변경 내용이 적용됩니다.**

---

### 4. 텔레그램 웹훅 설정

웹 앱 URL을 아래 명령어에 붙여 넣어 실행

```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<웹앱_URL>
```

**예시**
텔레그램 봇 토큰이 `123456789:ABCxyz` 이고, 웹 앱 URL이
`https://script.google.com/macros/s/AKfycbXYZ123/exec` 라면:

```
https://api.telegram.org/bot123456789:ABCxyz/setWebhook?url=https://script.google.com/macros/s/AKfycbXYZ123/exec
```

---

## 기타

* 슬라이드 템플릿에 맞춰 다양한 테마를 직접 구성한 후 `TEMPLATE_SLIDE_ID`로 복사해 사용하면 디자인 품질을 높일 수 있습니다.
* 웹페이지의 구조가 복잡하거나 자바스크립트 기반일 경우, 본문 텍스트 파싱이 제대로 되지 않을 수 있습니다.
  이 경우 다음 형식으로 Jina AI Reader 서비스를 사용하면 대부분 해결됩니다:

```javascript
const modifiedLink = 'https://r.jina.ai/' + originalLink;
```

해당 서비스를 통해 텍스트만 추출하여 처리 가능: [https://jina.ai/reader/](https://jina.ai/reader/)

* 앱을 새로 배포하거나 템플릿을 바꾼 경우, 웹훅도 다시 설정해야 합니다.

