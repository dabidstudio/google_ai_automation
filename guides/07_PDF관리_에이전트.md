# PDF 파일을 요약하고 링크로 보내주는 PDF 관리 에이전트

## 프로그램 소개

이 프로그램은 **텔레그램에서 PDF 파일을 전송하면, PDF 텍스트를 추출하고 GPT로 요약한 후 구글드라이브에 저장하고 요약 문서 링크와 원본 PDF 링크를 자동 응답하는 에이전트입니다.
특정 문서 요약, 보고서 전달, 자료 검토 등에 활용할 수 있습니다.

---

## 실행 방법

### 1. 각종 키 및 PDF 준비

* **TELEGRAM\_TOKEN**: [봇 토큰 발급 가이드](https://github.com/dabidstudio/dabidstudio_guides/blob/main/get_telegram_token.md)를 참고해 발급 후 입력
* **OPENAI\_API\_KEY**: [OpenAI 키 발급 가이드](https://github.com/dabidstudio/dabidstudio_guides/blob/main/get-openai-api-key.md)를 참고해 발급 후 입력
* 샘플 PDF 파일 : [Wikipedia의 Google Apps Script 문서](https://en.wikipedia.org/w/index.php?title=Special:DownloadAsPdf&page=Google_Apps_Script&action=show-download-screen)
---

### 2. Google Apps Script 열기

1. [https://script.new](https://script.new) 에 접속해 새 프로젝트 생성
2. 아래 코드를 붙여넣기

```javascript
const TELEGRAM_TOKEN = 'YOUR_TELEGRAM_BOT_TOKEN';
const TELEGRAM_API_URL = 'https://api.telegram.org/bot' + TELEGRAM_TOKEN;
const OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY';

function doPost(e) {
  const contents = JSON.parse(e.postData.contents);
  const message = contents.message;

  if (!message || !message.document) {
    sendMessage(message.chat.id, 'PDF 파일을 업로드해주세요.');
    return;
  }

  const fileId = message.document.file_id;
  const chatId = message.chat.id;

  const filePath = getTelegramFilePath(fileId);
  if (!filePath) return;

  const pdfBlob = downloadTelegramFile(filePath, message.document.file_name);
  if (!pdfBlob) return;

  const text = extractTextFromPDF(pdfBlob);
  if (!text || text.startsWith('오류')) return;

  const summary = summarizeText(text);
  if (!summary || summary.startsWith('요약하는 중 오류')) return;

  const docUrl = saveSummaryToGoogleDoc(summary);
  const pdfUrl = getShareableLink(pdfBlob);

  sendMessage(chatId, `요약이 완료되었습니다.\n\n📄 요약 문서: ${docUrl}\n📎 원본 PDF: ${pdfUrl}`);
}

function getTelegramFilePath(fileId) {
  try {
    const url = `${TELEGRAM_API_URL}/getFile?file_id=${fileId}`;
    const response = UrlFetchApp.fetch(url);
    const json = JSON.parse(response.getContentText());
    return json.result?.file_path || null;
  } catch (error) {
    return null;
  }
}

function downloadTelegramFile(filePath, fileName) {
  try {
    const fileUrl = `https://api.telegram.org/file/bot${TELEGRAM_TOKEN}/${filePath}`;
    const response = UrlFetchApp.fetch(fileUrl);
    return response.getBlob().setName(fileName);
  } catch (error) {
    return null;
  }
}

function extractTextFromPDF(pdfBlob) {
  try {
    const resource = {
      name: pdfBlob.getName().replace(/\.pdf$/i, ''),
      mimeType: MimeType.GOOGLE_DOCS
    };
    const options = {
      ocr: true,
      ocrLanguage: 'ko',
      fields: 'id,name'
    };

    const converted = Drive.Files.create(resource, pdfBlob, options);
    const doc = DocumentApp.openById(converted.id);
    const text = doc.getBody().getText();
    DriveApp.getFileById(converted.id).setTrashed(true);
    return text || 'PDF에서 텍스트를 찾을 수 없습니다.';
  } catch (error) {
    return '오류로 인해 텍스트를 추출할 수 없습니다.';
  }
}

function summarizeText(text) {
  try {
    const prompt = `다음 내용을 한국어로 요약해 주세요:\n\n${text}`;
    const url = 'https://api.openai.com/v1/chat/completions';
    const payload = {
      model: 'gpt-4',
      messages: [
        { role: 'system', content: '당신은 유능한 요약가입니다.' },
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
      payload: JSON.stringify(payload),
      muteHttpExceptions: true
    };

    const response = UrlFetchApp.fetch(url, options);
    const data = JSON.parse(response.getContentText());
    return data.choices[0].message.content.trim();
  } catch (error) {
    return '요약하는 중 오류가 발생했습니다.';
  }
}

function saveSummaryToGoogleDoc(summary) {
  try {
    const doc = DocumentApp.create('PDF 요약 결과');
    doc.getBody().setText(summary);
    doc.saveAndClose();
    const file = DriveApp.getFileById(doc.getId());
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
    return doc.getUrl();
  } catch (error) {
    return '요약 문서를 저장하는 중 오류가 발생했습니다.';
  }
}

function getShareableLink(fileBlob) {
  try {
    const file = DriveApp.createFile(fileBlob);
    file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
    return file.getUrl();
  } catch (error) {
    return 'PDF 공유 링크 생성 중 오류가 발생했습니다.';
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

   * **설명**: PDF 요약 봇
   * **다음 사용자로 앱 실행**: 본인 계정
   * **앱에 액세스할 수 있는 사용자**: *익명 사용자(Anyone)*
3. **"새 버전으로 배포"** 클릭 → **웹 앱 URL 복사**

**중요: 코드를 수정할 경우, 반드시 새 버전으로 다시 배포해야 변경 사항이 적용됩니다.**

---

### 4. 텔레그램 웹훅 설정

웹 앱 URL을 아래 형식으로 등록합니다:

```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<웹앱_URL>
```

**예시**
텔레그램 봇 토큰이 `123456789:ABCxyz` 이고,
웹 앱 URL이 `https://script.google.com/macros/s/AKfycb123456789/exec` 라면:

```
https://api.telegram.org/bot123456789:ABCxyz/setWebhook?url=https://script.google.com/macros/s/AKfycb123456789/exec
```

웹훅이 정상적으로 설정되면, 텔레그램에서 PDF를 전송하면 자동으로 요약 결과가 전송됩니다.

---

## 기타

* PDF 텍스트 추출은 Google Drive OCR 기능을 사용하며, 이미지 기반 PDF도 처리 가능합니다.
* Google 문서 및 PDF 파일은 모두 **링크로 공유 가능**하도록 설정되며, 받는 사람이 열람만 할 수 있습니다.
* GPT 요약의 품질은 텍스트 길이 및 난이도에 따라 달라질 수 있으며, 필요 시 prompt를 조정해 더욱 구체적인 요약을 요청할 수 있습니다.
* 앱 URL이 변경되거나 새로 배포되면 웹훅도 반드시 다시 등록해야 합니다.
