
## AI 함수 프로그램 소개

이 프로그램은 Google Apps Script를 사용하여 **Google 스프레드시트에서 직접 OpenAI GPT 모델에 요청하고 응답을 받아오는 사용자 정의 함수**입니다.

예를 들어 아래와 같이 사용할 수 있습니다:

```
=GPT("이 문장을 요약해줘", "간단하게 요약해줘")
```

---

## 실행 방법

1. Google 스프레드시트 열기
   [https://sheets.new](https://sheets.new) 에 접속해 새 시트를 엽니다.

2. Apps Script 열기
   상단 메뉴 → **확장 프로그램 > Apps Script** 클릭

3. 아래 코드 붙여넣기

```javascript
/**
 * GPT 함수를 통해 OpenAI로 텍스트를 보냅니다.
 *
 * @param {string} text 요약할 텍스트
 * @param {string} instruction 지시어(예: "요약해줘")
 * @return 응답 텍스트
 * @customfunction
 */
function GPT(text, instruction) {
  return callGPT(text, instruction);
}

// OpenAI API 호출 함수
function callGPT(prompt, instruction) {
  var api_key = 'YOUR_OPENAI_API_KEY'; // 여기에 본인의 OpenAI API Key 입력
  var url = 'https://api.openai.com/v1/chat/completions';

  var payload = {
    "model": "gpt-4o",
    "messages": [
      { "role": "system", "content": instruction },
      { "role": "user", "content": prompt }
    ],
    "temperature": 0.2
  };

  var options = {
    "method": "post",
    "contentType": "application/json",
    "headers": {
      "Authorization": "Bearer " + api_key
    },
    "payload": JSON.stringify(payload),
    "muteHttpExceptions": true
  };

  var response = UrlFetchApp.fetch(url, options);
  var json = JSON.parse(response.getContentText());

  try {
    return json.choices[0].message.content.trim();
  } catch (e) {
    return "오류 발생: " + response.getContentText();
  }
}
```

4. OpenAI API 키 입력
   아래 가이드 문서를 참고하여 API 키를 발급받고, `'YOUR_OPENAI_API_KEY'` 부분에 붙여넣습니다.
   → [API 키 발급 가이드](https://github.com/dabidstudio/dabidstudio_guides/blob/main/get-openai-api-key.md)

5. 저장하기
   상단 메뉴 또는 `Ctrl + S`를 눌러 스크립트를 저장하면 준비 완료입니다.
   별도의 실행이나 권한 승인은 필요하지 않습니다.

6. 스프레드시트에서 사용하기
   예시:

   ```
   =GPT("기후 변화의 원인을 설명해줘", "초등학생 눈높이에 맞춰 설명해줘")
   ```

---

## 팁

1. 셀 참조를 활용하면 다양한 입력 조합을 자동화할 수 있습니다.
   예:

   ```
   =GPT(A2, B2)
   ```

2. **절대참조/상대참조**를 활용하면 반복 작업을 효율화할 수 있습니다.
   예:

   ```
   =GPT(A2, $B$1)  // A열 텍스트에 B1 셀 지시어를 고정 적용
   ```

3. 자주 쓰는 예시

   * 요약: `=GPT(A2, "한 문장으로 요약해줘")`
   * 문장 다듬기: `=GPT(A2, "더 자연스럽게 바꿔줘")`
   * 번역: `=GPT(A2, "영어로 번역해줘")`
   * 해설: `=GPT(A2, "전문가처럼 쉽게 풀어서 설명해줘")`

---

필요 시 예제 스프레드시트나 PDF 버전도 제작해드릴 수 있습니다. 원하시면 알려주세요.
