# Livelinen Card Scanner (GitHub Pages + n8n)

No Telegram. Browser UI on GitHub Pages. OCR heavy work in n8n.

## Live app

https://development928.github.io/Livelinen-OCR-/

## Flow

1. User opens the page
2. **Upload & OCR** → image sent to n8n webhook → Gemini → form filled
3. Or **Manual entry** → type fields directly
4. **Save card** → inserts into Supabase `visiting_cards`

## Setup n8n OCR webhook (one workflow)

1. **Webhook** node  
   - Method: `POST`  
   - Path: `card-ocr`  
   - Response mode: **Using Respond to Webhook node**

2. **Code** node — turn base64 into binary for Gemini:

```javascript
const item = $input.first().json;
const binary = Buffer.from(item.imageBase64, 'base64');

return [{
  json: {
    filename: item.filename || 'card.jpg',
    mimeType: item.mimeType || 'image/jpeg',
  },
  binary: {
    data: {
      data: binary.toString('base64'),
      mimeType: item.mimeType || 'image/jpeg',
      fileName: item.filename || 'card.jpg',
    },
  },
}];
```

3. **Google Gemini — Analyze image** (same prompt as before), binary input from `data`

4. **Code** — parse Gemini JSON (same parser as before, without Telegram fields)

5. **Respond to Webhook**  
   - Respond With: JSON  
   - Body: the parsed object (`first_name`, `last_name`, `company_name`, `emails`, `phone_numbers`)  
   - Add response header: `Access-Control-Allow-Origin` = `*`  
   - Also handle OPTIONS / CORS if your n8n version needs it

6. Copy the Webhook **Production URL** into `index.html`:

```js
const N8N_OCR_WEBHOOK_URL = "https://YOUR-N8N-HOST/webhook/card-ocr";
```

7. Commit & push `index.html`, wait for GitHub Pages to update

## Notes

- Keep Gemini API key only in n8n — never in GitHub Pages
- Supabase anon key in the page is expected; RLS controls access
