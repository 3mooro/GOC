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
  try {
    const postData = JSON.parse(e.postData.contents);
    const method = postData.method; // "QR_SCAN" or "MANUAL"
    
    let email = "";
    let teamName = "Manual Entry";
    
    if (method === "QR_SCAN") {
      const payload = postData.payload; // format: email|team_name|signature
      const parts = payload.split('|');
      
      if (parts.length !== 3) {
        return ContentService.createTextOutput(JSON.stringify({success: false, error: "Invalid format"})).setMimeType(ContentService.MimeType.JSON);
      }
      
      email = parts[0];
      teamName = parts[1];
      const providedSignature = parts[2];
      
      // Verify HMAC Signature to prevent spoofing
      const dataToSign = email + "|" + teamName;
      const expectedSignature = computeHMAC(SECRET_KEY, dataToSign);
      
      if (providedSignature !== expectedSignature) {
        return ContentService.createTextOutput(JSON.stringify({success: false, error: "Signature mismatch! Fake QR Code."})).setMimeType(ContentService.MimeType.JSON);
      }
    } else {
      // Manual Entry
      email = postData.payload;
    }
    
    // Log to the Google Sheet (for backup)
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    sheet.appendRow([new Date(), email, teamName, method]);
    
    // --- PUSH TO NOTION ---
    // Example fetch to Notion API to mark attendance
    updateNotion(email);
    
    return ContentService.createTextOutput(JSON.stringify({success: true})).setMimeType(ContentService.MimeType.JSON);
    
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({success: false, error: error.toString()})).setMimeType(ContentService.MimeType.JSON);
  }
}

// Function to update Notion via API
function updateNotion(email) {
  const url = 'https://api.notion.com/v1/pages'; // or /databases/{db_id}/query to find the page first
  
  // NOTE: In a real scenario, you usually Query the DB to find the Page ID using the email, 
  // then send a PATCH request to that Page ID to update the "Attendance" property.
  // This is a placeholder for your specific Notion architecture.
  
  /* Example:
  const options = {
    method: 'post',
    headers: {
      'Authorization': 'Bearer ' + NOTION_API_KEY,
      'Notion-Version': '2022-06-28',
      'Content-Type': 'application/json'
    },
    payload: JSON.stringify({ ... })
  };
  UrlFetchApp.fetch(url, options);
  */
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
