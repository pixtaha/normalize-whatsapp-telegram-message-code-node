# 🤖 n8n Telegram & WhatsApp Message Normalizer

> طبقة تطبيع موحدة تستقبل الرسائل من **Telegram** و**WhatsApp** وتحولها لـ output متسق جاهز للمعالجة

[![n8n](https://img.shields.io/badge/n8n-workflow-orange)](https://n8n.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## 📌 ما هو هذا الـ Workflow؟

بدلاً من كتابة كود منفصل لكل قناة، هذا الـ workflow يوحّد الرسائل القادمة من Telegram وWhatsApp في **output واحد متسق**، مما يجعل بناء الـ chatbots أسهل وأسرع.

---

## ✨ المميزات

- ✅ دعم كامل لـ **Telegram** و**WhatsApp** (عبر Evolution API)
- ✅ تطبيع الأرقام العربية والفارسية تلقائياً `٣` → `3`
- ✅ كشف جميع أنواع الرسائل: نص، صور، فيديو، صوت، ردود، commands، callbacks
- ✅ دعم **Reply** مع معرفة نوع الرسالة المردود عليها
- ✅ تحميل الملفات من الاتنين وتحويلها لـ binary جاهز للمعالجة
- ✅ تحويل الصوت لنص بـ **OpenAI Whisper**
- ✅ فصل بيئات dev/prod

---

## 🏗️ معمارية الـ Workflow

```
TG • Trigger! ──┐
                ├──→ ENV • Config ──→ Normalize • Patient Message ──→ Switch • Action Type
WA • Trigger! ──┘
```

### Nodes الرئيسية

| Node | الوظيفة |
|------|----------|
| `TG • Trigger!` | استقبال رسائل Telegram |
| `WA • Trigger!` | استقبال رسائل WhatsApp عبر Webhook |
| `Check Sender • Not Me` | فلترة رسائل البوت نفسه |
| `ENV • Config` | إعدادات البيئة + جلب بيانات الـ trigger |
| `Normalize • Patient Message` | **القلب** — تطبيع الرسالة من الاتنين |
| `Switch • Action Type` | توجيه حسب نوع الرسالة |
| `Switch • Source Channel` | تفريق WhatsApp عن Telegram للـ assets |
| `WA • Download Asset` | تحميل الملفات من WhatsApp |
| `TG • Download Asset` | تحميل الملفات من Telegram |
| `Convert • Base64 to Binary` | تحويل base64 لـ binary |
| `Convert • File Id to Binary` | تحميل ملف Telegram كـ binary |
| `Merge` | دمج الـ binary في مسار واحد |
| `Switch • Asset Type` | توجيه Photo / Video / Audio |
| `Transcribe a recording` | تحويل الصوت لنص بـ OpenAI Whisper |

---

## 📤 الـ Output الموحد

```json
{
  "env": "dev",
  "source_channel": "telegram | whatsapp",
  "user_id": 123456789,
  "chat_id": 123456789,
  "message_text": "نص الرسالة",
  "message_type": "message | command | callback_query | photo | video | audio | reply | document | sticker | from_me | unknown",
  "is_command": false,
  "command": "/start",
  "callback_data": "context:action:sid",
  "callback_context": "context",
  "callback_action": "action",
  "callback_sid": "sid",
  "has_photo": false,
  "photo_file_id": "file_id أو url",
  "has_video": false,
  "video_file_id": "file_id أو url",
  "audio_file_id": "file_id أو url",
  "caption": "كابشن الصورة أو الفيديو",
  "is_reply": false,
  "replied_to_type": "text | photo | video | audio",
  "replied_to_text": "نص الرسالة المردود عليها",
  "replied_to_photo_id": "file_id أو url",
  "replied_to_video_id": "file_id أو url",
  "replied_to_audio_id": "file_id أو url",
  "raw": {}
}
```

---

## ⚙️ الإعداد

### المتطلبات

- n8n (self-hosted)
- Telegram Bot Token
- Evolution API (لـ WhatsApp)
- OpenAI API Key (للـ Transcription — اختياري)

### خطوات الإعداد

**1. إعداد البيئة في `ENV • Config`**

```javascript
const ENV = 'dev'; // غير لـ 'prod' في الإنتاج

const CONFIG = {
  dev: {
    tgToken: 'YOUR-BOT-TOKEN',
    numbersTable: '---'
  },
  prod: {
    tgToken: 'YOUR-BOT-TOKEN',
    numbersTable: '---'
  }
};
```

**2. إعداد WhatsApp Webhook في Evolution API**

```
POST https://your-n8n-domain/webhook/YOUR-SECURE-PATH
```

> ⚠️ غيّر الـ webhook path لنص عشوائي معقد لأمان أكبر

**3. إعداد الـ Credentials في n8n**

- Telegram Bot Token في `TG • Trigger!`
- OpenAI API Key في `Transcribe a recording`
- Evolution API Key في headers الـ `WA • Download Asset`

---

## 🔀 منطق التوجيه

```
Switch • Action Type
├── Callback  → SW • CallBack Type → TG • Answer Callback Query
├── Command   → SW • Command Type  → (command handlers)
├── Asset     → Switch • Source Channel
│               ├── WhatsApp → WA • Download Asset → Convert • Base64 to Binary ──┐
│               └── Telegram → TG • Download Asset → Convert • File Id to Binary ──┤
│                                                                                   ↓
│                                                                                 Merge
│                                                                                   ↓
│                                                                       Switch • Asset Type
│                                                                       ├── Photo
│                                                                       ├── Video
│                                                                       └── Audio → OpenAI Whisper
├── Text      → (text handlers)
└── Reply     → (reply handlers)
```

---

## 🔧 تفاصيل تقنية

### لماذا `ENV • Config` يجيب بيانات الـ Trigger؟

n8n لا يسمح بالإشارة لـ node لم يتم تنفيذه في نفس الـ execution. الـ `ENV • Config` يستخدم `try/catch` لجلب بيانات الـ trigger الشغال ويمررها كـ `triggerData`.

### تحميل ملفات WhatsApp

WhatsApp يرسل الملفات مشفرة. نستخدم Evolution API endpoint:
```
POST /chat/getBase64FromMediaMessage/{instance}
```

### تحميل ملفات Telegram

Telegram يستخدم `file_id` وليس URL مباشر — خطوتان:
1. `getFile` API للحصول على `file_path`
2. تحميل الملف من `https://api.telegram.org/file/bot{TOKEN}/{file_path}`

---

## 📄 الترخيص

MIT License — استخدم وعدّل بحرية مع ذكر المصدر.

---

*صُنع بـ ❤️ لمساعدة مطوري n8n العرب*
