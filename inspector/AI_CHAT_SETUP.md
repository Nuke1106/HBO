# การติดตั้ง AI Chat Assistant (Google Apps Script)

ระบบ AI Chat ใช้ Google Apps Script (GAS) เดิมเป็น proxy ไปยัง Anthropic API
**API Key อยู่ที่ GAS เท่านั้น ไม่ปรากฏใน frontend**

---

## ขั้นที่ 1: เปิดโปรเจกต์ GAS เดิม

เปิด: https://script.google.com → เปิดโปรเจกต์ที่ deploy URL เดียวกับ
`https://script.google.com/macros/s/AKfycbzHDj38XpjPvjfPGUz_fA564r5XiefTYr4rdzPVzMEfMQHNWkuQNUr_R5GE-73E87Tk2A/exec`

---

## ขั้นที่ 2: เก็บ API Key ใน Script Properties

1. ไปที่ ⚙ **Project Settings**
2. เลื่อนลงไปที่ **Script Properties** → กด **Add script property**
3. ใส่:
   - **Property:** `ANTHROPIC_API_KEY`
   - **Value:** `sk-ant-api03-...` (paste API key ตรงนี้)
4. กด **Save script properties**

---

## ขั้นที่ 3: เพิ่ม Function `handleAiChat` ใน Code.gs

วาง code ด้านล่างนี้ลงท้ายไฟล์ Code.gs ของโปรเจกต์ GAS:

```javascript
function handleAiChat(data) {
  const apiKey = PropertiesService.getScriptProperties().getProperty('ANTHROPIC_API_KEY');
  if (!apiKey) {
    return ContentService.createTextOutput(JSON.stringify({
      ok: false,
      error: 'ANTHROPIC_API_KEY ไม่ได้ตั้งใน Script Properties'
    })).setMimeType(ContentService.MimeType.JSON);
  }

  const messages = data.messages || [];
  if (!messages.length) {
    return ContentService.createTextOutput(JSON.stringify({
      ok: false, error: 'messages ว่าง'
    })).setMimeType(ContentService.MimeType.JSON);
  }

  const systemPrompt = 'คุณเป็นผู้ช่วยด้านเวชศาสตร์ความดันบรรยากาศสูง (Hyperbaric Oxygen Therapy - HBO) ' +
    'สำหรับบุคลากรทางการแพทย์\n' +
    '- ตอบเป็นภาษาไทย กระชับ ชัดเจน\n' +
    '- ใช้ web search ค้นข้อมูลปัจจุบันจาก UHMS, ECHM, วารสารทางการแพทย์\n' +
    '- อ้างอิงแหล่งที่มาทุกครั้ง\n' +
    '- ถ้าไม่แน่ใจ ให้บอกตรงๆ ไม่แต่งคำตอบเอง';

  let workingMessages = messages.slice();
  const MAX_ITER = 6;

  try {
    for (let i = 0; i < MAX_ITER; i++) {
      const resp = UrlFetchApp.fetch('https://api.anthropic.com/v1/messages', {
        method: 'post',
        contentType: 'application/json',
        headers: {
          'x-api-key': apiKey,
          'anthropic-version': '2023-06-01'
        },
        payload: JSON.stringify({
          model: 'claude-sonnet-4-6',
          max_tokens: 2048,
          system: systemPrompt,
          tools: [{ type: 'web_search_20250305', name: 'web_search', max_uses: 5 }],
          messages: workingMessages
        }),
        muteHttpExceptions: true
      });

      const code = resp.getResponseCode();
      const body = resp.getContentText();

      if (code !== 200) {
        return ContentService.createTextOutput(JSON.stringify({
          ok: false,
          error: 'Anthropic API error: HTTP ' + code,
          detail: body.slice(0, 500)
        })).setMimeType(ContentService.MimeType.JSON);
      }

      const parsed = JSON.parse(body);

      if (parsed.stop_reason !== 'tool_use') {
        const answer = (parsed.content || [])
          .filter(function(b){ return b.type === 'text'; })
          .map(function(b){ return b.text; })
          .join('\n\n');
        return ContentService.createTextOutput(JSON.stringify({
          ok: true,
          answer: answer,
          stop_reason: parsed.stop_reason
        })).setMimeType(ContentService.MimeType.JSON);
      }

      workingMessages.push({ role: 'assistant', content: parsed.content });
      const toolResults = (parsed.content || [])
        .filter(function(b){ return b.type === 'tool_use'; })
        .map(function(b){
          return { type: 'tool_result', tool_use_id: b.id, content: '' };
        });
      if (!toolResults.length) break;
      workingMessages.push({ role: 'user', content: toolResults });
    }

    return ContentService.createTextOutput(JSON.stringify({
      ok: false, error: 'AI ทำงานเกินจำนวนรอบที่กำหนด'
    })).setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({
      ok: false, error: 'GAS Error: ' + err.message
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

---

## ขั้นที่ 4: เพิ่ม Route ใน doPost()

หา function `doPost(e)` ในโปรเจกต์ GAS แล้วเพิ่ม `if` ใหม่ก่อนการ return:

```javascript
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    const action = data.action;

    if (action === 'ai_chat') return handleAiChat(data);   // <-- เพิ่มบรรทัดนี้
    if (action === 'upload_photo') return handleUploadPhoto(data);
    if (action === 'bulk_submit')  return handleBulkSubmit(data);
    // ... อื่นๆ ของเดิม
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({
      ok: false, error: err.message
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

---

## ขั้นที่ 5: Deploy เวอร์ชันใหม่

1. กด **Deploy** → **Manage deployments**
2. กด ✏ **Edit** ของ deployment เดิม
3. **Version** → เลือก **New version**
4. **Description:** `Add AI chat endpoint`
5. กด **Deploy**

URL จะเป็นตัวเดิม ไม่ต้องเปลี่ยนใน HTML

---

## ขั้นที่ 6: ทดสอบ

1. เปิด `inspector/index.html`
2. กดปุ่ม 💬 มุมล่างซ้าย
3. พิมพ์คำถามเช่น "อธิบาย UHMS HBO indication"
4. รอ AI ตอบ (ประมาณ 5–30 วินาที — มี web search)

---

## หมายเหตุด้านโควต้า

- GAS UrlFetch: 20,000 calls/วัน (ฟรี)
- Execution timeout: 6 นาที/call (เกินพอสำหรับ AI 1 รอบ)
- Anthropic API: คิดเงินตาม token ที่ใช้

หากเปลี่ยน API key ภายหลัง → แก้ใน **Script Properties** ของ GAS เท่านั้น
ไม่ต้องแก้ HTML ไม่ต้อง deploy ใหม่
