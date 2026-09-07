# Livelinen Card Scanner (GitHub Pages + n8n)

No Telegram. Browser UI on GitHub Pages. OCR heavy work in n8n.

## Live app

https://development928.github.io/Livelinen-OCR-/

## Flow

1. User opens the page
2. Upload **front** (required) + **back** (optional)
3. **Run OCR** → n8n webhook → Gemini merges both sides → form filled
4. Or **Manual entry**
5. **Save card** → Supabase `visiting_cards`

## Webhook body from the UI

```json
{
  "imageBase64Front": "...",
  "mimeTypeFront": "image/jpeg",
  "filenameFront": "front.jpg",
  "imageBase64Back": "...",
  "mimeTypeBack": "image/jpeg",
  "filenameBack": "back.jpg",
  "hasBack": true,
  "imageBase64": "...",
  "mimeType": "image/jpeg",
  "filename": "front.jpg"
}
```

(`imageBase64` is the front image, kept for compatibility.)

## n8n OCR webhook setup

1. **Webhook** (POST)  
   Response mode: **Using Respond to Webhook node**

2. **Code** — build front (+ optional back) binary:

```javascript
const item = $input.first().json;

const frontB64 = item.imageBase64Front || item.imageBase64;
if (!frontB64) throw new Error('Front image missing');

const out = {
  json: {
    hasBack: !!(item.imageBase64Back || item.hasBack),
    filenameFront: item.filenameFront || item.filename || 'front.jpg',
    filenameBack: item.filenameBack || 'back.jpg',
  },
  binary: {
    front: {
      data: frontB64,
      mimeType: item.mimeTypeFront || item.mimeType || 'image/jpeg',
      fileName: item.filenameFront || item.filename || 'front.jpg',
    },
  },
};

if (item.imageBase64Back) {
  out.binary.back = {
    data: item.imageBase64Back,
    mimeType: item.mimeTypeBack || 'image/jpeg',
    fileName: item.filenameBack || 'back.jpg',
  };
}

return [out];
```

3. **Google Gemini — Analyze image**  
   - Attach **both** binaries if available (`front` and `back`)  
   - Prompt:

```text
You are given business card image(s).
Image labeled FRONT is the front side.
If a BACK image is also provided, merge info from both sides into ONE contact.

Return ONLY valid JSON (no markdown):
{
  "first_name": "",
  "last_name": "",
  "company_name": "",
  "emails": [""],
  "phone_numbers": [{"type":"mobile|office|fax|other","number":""}]
}

Rules:
- Combine fields from front and back.
- Prefer clearer/more complete values on conflict.
- Do not invent data.
- Missing fields: "" or [].
```

4. **Code** — parse Gemini JSON

5. **Respond to Webhook**  
   - JSON body = parsed card  
   - Header: `Access-Control-Allow-Origin: *`

## Notes

- Keep Gemini API key only in n8n
- Front-only still works if back is empty
