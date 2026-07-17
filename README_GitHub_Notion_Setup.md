# 🏴‍☠️ Game of Coders 5.0 - Attendance System (GitHub Pages + Google Sheets + Notion)

This guide explains how to host your beautiful, static Pirate-themed QR Scanner on GitHub Pages, and how to use Google Apps Script as a secure "Proxy" to verify the QR codes and update Notion.

## Step 1: Host on GitHub Pages
1. Initialize a Git repository in this folder (`/home/omar/Desktop/GameOfCoders5-Attendance-Static/`).
2. Push the code (which includes `index.html` and the `images/` folder) to a new GitHub repository.
3. Go to the repository **Settings** -> **Pages**.
4. Select the `main` branch and click **Save**.
5. Your scanner will be live at `https://<your-username>.github.io/<repo-name>/`.
6. **Passkey Note:** The scanner has a simple passkey (`goc2026`). You can change this in `index.html` on line 170.

---

## Step 2: Set up the Google Sheet Proxy
To hide your Notion API Key and verify the secure signatures on the QR codes, we use a Google Sheet + Google Apps Script as a proxy.

1. Go to [Google Sheets](https://sheets.google.com) and create a new blank spreadsheet. Name it "GOC 5.0 Attendance Proxy".
2. Click on **Extensions** -> **Apps Script**.
3. Delete the default `myFunction()` code and paste the following code exactly as is:

```javascript
// --- CONFIGURATION ---
// This SECRET_KEY MUST EXACTLY MATCH the one in your generate_theme_qrs.py file!
const SECRET_KEY = "GOC_5_SECRET_TREASURE_KEY_2026"; 
const NOTION_API_KEY = "YOUR_NOTION_API_KEY_HERE";
const NOTION_DATABASE_ID = "YOUR_NOTION_DATABASE_ID_HERE";

// This function receives the POST request from your GitHub Pages scanner
function doPost(e) {
  // Setup CORS headers to allow GitHub Pages to read the JSON response
  const headers = {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "POST, GET, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type"
  };

  try {
    const postData = JSON.parse(e.postData.contents);
    const method = postData.method; // "QR_SCAN" or "MANUAL"
    
    let personName = "Unknown";
    let email = "";
    let teamName = "Manual Entry";
    
    if (method === "QR_SCAN") {
      const payload = postData.payload; // format: person_name|email|team_name|signature
      const parts = payload.split('|');
      
      if (parts.length !== 4) {
        return ContentService.createTextOutput(JSON.stringify({success: false, error: "QR غير صالح! تأكد من تحديث الكيو ار."})).setMimeType(ContentService.MimeType.JSON);
      }
      
      personName = parts[0];
      email = parts[1];
      teamName = parts[2];
      const providedSignature = parts[3];
      
      // Verify HMAC Signature to prevent spoofing
      const dataToSign = personName + "|" + email + "|" + teamName;
      const expectedSignature = computeHMAC(SECRET_KEY, dataToSign);
      
      if (providedSignature !== expectedSignature) {
        return ContentService.createTextOutput(JSON.stringify({success: false, error: "توقيع غير مطابق! QR مزيف."})).setMimeType(ContentService.MimeType.JSON);
      }
    } else {
      // Manual Entry
      email = postData.payload;
    }
    
    // Log to the Google Sheet (for backup)
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    sheet.appendRow([new Date(), personName, email, teamName, method]);
    
    // --- PUSH TO NOTION ---
    // Make sure updateNotion returns the success object
    const notionResult = updateNotion(email);
    
    if (notionResult.success) {
      return ContentService.createTextOutput(JSON.stringify({success: true, personName: personName, teamName: teamName})).setMimeType(ContentService.MimeType.JSON);
    } else {
      return ContentService.createTextOutput(JSON.stringify({success: false, error: notionResult.error})).setMimeType(ContentService.MimeType.JSON);
    }
    
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({success: false, error: error.toString()})).setMimeType(ContentService.MimeType.JSON);
  }
}

// Function to update Notion via API
function updateNotion(userEmail) {
  const queryUrl = 'https://api.notion.com/v1/databases/' + NOTION_DATABASE_ID + '/query';
  
  const headers = {
    'Authorization': 'Bearer ' + NOTION_API_KEY,
    'Notion-Version': '2022-06-28',
    'Content-Type': 'application/json'
  };

  const queryPayload = {
    "filter": {
      "property": "Email", 
      "email": {
        "equals": userEmail
      }
    }
  };

  const queryOptions = {
    'method': 'post',
    'headers': headers,
    'payload': JSON.stringify(queryPayload),
    'muteHttpExceptions': true
  };
  
  const queryResponse = UrlFetchApp.fetch(queryUrl, queryOptions);
  const queryData = JSON.parse(queryResponse.getContentText());
  
  // 1. لو الإيميل مش موجود أصلاً في نوشن
  if (!queryData.results || queryData.results.length === 0) {
    return { success: false, error: "الإيميل ده مش متسجل في نوشن!" };
  }
  
  const page = queryData.results[0];
  const pageId = page.id;
  
  // 2. لو هو حضر قبل كده متعلم عليه (Done)
  // تأكد إن اسم عمود الحالة عندك Status بالظبط أو غيره في الكود تحت
  let currentStatus = "";
  try {
    currentStatus = page.properties["Status"].status.name;
  } catch(e) {
    // In case the status is empty or property name is wrong
  }

  if (currentStatus === "Done") {
    return { success: false, error: "الشخص ده سجل حضور قبل كده بالفعل!" };
  }
    
  // 3. تحديث خانة الـ Status عشان نأكد حضوره
  const updateUrl = 'https://api.notion.com/v1/pages/' + pageId;
  const updatePayload = {
    "properties": {
      "Status": {
        "status": {
          "name": "Done" 
        }
      }
    }
  };
  
  const updateOptions = {
    'method': 'patch',
    'headers': headers,
    'payload': JSON.stringify(updatePayload),
    'muteHttpExceptions': true
  };
  
  const updateResponse = UrlFetchApp.fetch(updateUrl, updateOptions);
  const updateData = JSON.parse(updateResponse.getContentText());
  
  if (updateData.id) {
    return { success: true };
  } else {
    return { success: false, error: "خطأ في تعديل نوشن: " + updateData.message };
  }
}

// Utility function to compute HMAC SHA256 in Apps Script
function computeHMAC(secret, data) {
  const signatureBytes = Utilities.computeHmacSha256Signature(data, secret);
  let hexString = '';
  for (let i = 0; i < signatureBytes.length; i++) {
    let hex = (signatureBytes[i] & 0xFF).toString(16);
    if (hex.length === 1) hex = '0' + hex;
    hexString += hex;
  }
  return hexString;
}
```

## Step 3: Deploy the Proxy and Link it
1. In the Apps Script editor, click the **Deploy** button (top right) -> **New deployment**.
2. Select type: **Web app**.
3. Description: `GOC Attendance Proxy`
4. Execute as: **Me** (Your email).
5. Who has access: **Anyone** (This is crucial, otherwise GitHub Pages can't send data to it).
6. Click **Deploy**.
7. Copy the **Web app URL**.
8. Open your `/home/omar/Desktop/GameOfCoders5-Attendance-Static/index.html` file.
9. Go to line `167` and paste the URL: `const GOOGLE_SCRIPT_URL = 'YOUR_GOOGLE_SCRIPT_WEB_APP_URL_HERE';`
10. Push the updated `index.html` to GitHub!

Now, when you scan a QR code from GitHub Pages, it sends the data securely to your Google Sheet (acting as a proxy), which verifies the secret code, logs it as a backup, and pushes it to Notion via API securely!
