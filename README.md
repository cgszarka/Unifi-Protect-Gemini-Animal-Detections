# Unifi-Protect-Gemini-Animal-Detections
This project uses Google Apps Script as a webhook receiver to connect UniFi Protect security cameras to Google's Gemini AI. It automatically analyzes animal detection events, uses AI to identify the specific animal, filters out unwanted alerts (like neighborhood cats or squirrels), and emails you a notification with the thumbnail attached.
# UniFi Protect to Gemini AI Smart Webhook

This project uses Google Apps Script as a webhook receiver to connect UniFi Protect security cameras to Google's Gemini AI. It automatically analyzes animal detection events, uses AI to identify the specific animal, filters out unwanted alerts (like neighborhood cats or squirrels), and emails you a notification with the thumbnail attached.

## Features
* **Zero-Cost Infrastructure:** Runs entirely on Google Apps Script (free) and the Gemini API (free tier).
* **Smart Filtering:** Only alerts you for target animals you actually care about (e.g., bears, coyotes, raccoons, dogs).
* **Robust Payload Parsing:** Extracts image thumbnails directly from the UniFi Protect webhook payload.
* **Context-Aware AI:** Uses a customizable prompt to differentiate between household pets and wildlife, even at night.

---

## 1. Get a Gemini API Key

1. Go to [Google AI Studio](https://aistudio.google.com).
2. Sign in with your Google account.
3. Click **Get API key** and create a new key.
4. Save this key somewhere safe.

## 2. Set Up the Google Apps Script

1. Go to [script.google.com](https://script.google.com) and click **New Project**.
2. Name the project (e.g., `UniFi Protect Webhook`).
3. Delete the default code in the editor and paste the complete `Code.gs` provided below.
4. Update the `GEMINI_API_KEY`, `NOTIFICATION_EMAIL` (can be comma-separated for multiple), and `TARGET_ANIMALS` constants at the top of the script.

### Deploy the Script
1. Click **Deploy** > **New deployment** in the top right corner.
2. Click the gear icon next to "Select type" and choose **Web app**.
3. Set the following configuration:
   * **Execute as:** Me
   * **Who has access:** Anyone
4. Click **Deploy**.
5. Authorize the required Google permissions (to allow the script to send emails).
6. **Copy the Web app URL** (it should end in `/exec`).

## 3. Set Up UniFi Protect

1. Log into your UniFi Protect dashboard.
2. Navigate to **Alarm Manager**.
3. Create a new alarm or edit an existing one triggered by **Animal Detection**.
4. Under **Actions**, select **Webhook** > **Custom Webhook**.
5. Paste your Apps Script Web app URL into the Delivery URL field.
6. Click **Advanced Settings**.
7. Set the Method to **HTTP POST**.
8. Ensure **"Use Thumbnails"** is toggled **ON**.
9. Click **Save**.

## 4. Updating and Fine-Tuning

* **Adding Target Animals:** Simply add new lowercase strings to the `TARGET_ANIMALS` array.
* **Tuning the Prompt:** If the AI consistently misidentifies your dog as a coyote or cat, edit the prompt inside the `identifyAnimalWithGemini()` function to include a physical description of your pet.
* **Applying Code Changes:** Whenever you edit the script, you **must** redeploy it. Click **Deploy** > **Manage deployments**, edit the active deployment, set the version to **New version**, and click **Deploy**.

---

## The Code (`Code.gs`)

```javascript
/**
 * ============================================================================
 * UniFi Protect to Gemini AI Smart Notifications
 * ============================================================================
 * 
 * CHANGELOG & DEVELOPMENT HISTORY:
 * 
 * v1.0 - Initial Webhook Setup
 *  - Established doPost(e) receiver for UniFi Protect custom webhooks.
 *  - Basic parsing of base64 thumbnail data.
 *  - Configured MailApp for basic email delivery.
 * 
 * v1.1 - Gemini AI Integration
 *  - Replaced standard vision APIs with Google Gemini Flash for faster, 
 *    more accurate image classification.
 * 
 * v1.2 - Target Animal Filtering
 *  - Added TARGET_ANIMALS array to prevent notification spam.
 *  - Script now evaluates Gemini's response and quietly exits if the 
 *    detected subject (e.g., squirrel, neighborhood cat) is not on the list.
 * 
 * v1.3 - Payload Extraction Robustness
 *  - Updated JSON parsing to hunt for the base64 image across multiple 
 *    potential UniFi payload structures (payload.thumbnail, 
 *    payload.alarm.thumbnail, and payload.event.thumbnail).
 *  - Handled UniFi's "data:image/jpeg;base64," prefix stripping.
 * 
 * v1.4 - Diagnostic & Debug Engine (Temporary)
 *  - Added a self-emailing diagnostic logger to bypass Google Apps Script's 
 *    hidden execution logs for anonymous webhooks.
 *  - Captured raw JSON payloads and Gemini HTTP status codes to troubleshoot 
 *    missing thumbnails and API 404 endpoint errors.
 * 
 * v1.5 - Prompt Engineering Fix
 *  - Fixed an issue where Gemini returned "None" when a human triggered 
 *    the camera. 
 *  - Adjusted the prompt from "specific animal" to "primary subject" 
 *    (explicitly including Person/Vehicle) to correctly classify humans.
 * 
 * v1.6 - Clean Production Release (Current)
 *  - Removed all diagnostic/debugging email steps for silent background execution.
 *  - Added support for multiple comma-separated email addresses in the 
 *    NOTIFICATION_EMAIL constant.
 * 
 * v1.7 - Prompt Fine-Tuning for Pet Identification
 *  - Updated the Gemini prompt to include specific context about household pets.
 *  - Added physical descriptions (size, breed, snout/tail features) to prevent 
 *    the model from misclassifying the dogs as cats.
 *  - Added specific instructions for handling distorted infrared/night-vision 
 *    images.
 * ============================================================================
 */

// ==========================================
// 1. CONFIGURATION
// ==========================================
const GEMINI_API_KEY = "YOUR_GEMINI_API_KEY";
const NOTIFICATION_EMAIL = "your-email@example.com";

// Target alert list (lowercase)
const TARGET_ANIMALS = ["bear", "coyote", "raccoon", "fox", "dog", "person"]; 

// ==========================================
// 2. MAIN WEBHOOK HANDLER
// ==========================================
function doPost(e) {
  try {
    if (!e || !e.postData) return ContentService.createTextOutput("No data");

    const payload = JSON.parse(e.postData.contents);

    let base64Image = payload.thumbnail || 
                      (payload.alarm && payload.alarm.thumbnail) || 
                      (payload.event && payload.event.thumbnail);
                      
    if (!base64Image) return ContentService.createTextOutput("No thumbnail");
    
    if (base64Image.startsWith("data:image")) {
      base64Image = base64Image.split(",")[1];
    }

    const animalType = identifyAnimalWithGemini(base64Image);
    
    if (animalType === "API_ERROR" || animalType === "Unknown") {
       return ContentService.createTextOutput("Identification failed");
    }

    const animalLower = animalType.toLowerCase();
    const isTargetAnimal = TARGET_ANIMALS.some(target => animalLower.includes(target));

    if (isTargetAnimal) {
      sendEmailNotification(animalType, base64Image);
      return ContentService.createTextOutput(`Success: ${animalType}`);
    } else {
      return ContentService.createTextOutput(`Ignored: ${animalType}`);
    }

  } catch (error) {
    console.error("Webhook Error: " + error.message);
    return ContentService.createTextOutput("Error");
  }
}

// ==========================================
// 3. HELPER FUNCTIONS
// ==========================================
function identifyAnimalWithGemini(base64Image) {
  const url = `[https://generativelanguage.googleapis.com/v1beta/models/gemini-3.8-flash:generateContent?key=$](https://generativelanguage.googleapis.com/v1beta/models/gemini-3.8-flash:generateContent?key=$){GEMINI_API_KEY}`;
  
  const requestBody = {
    "contents": [{
      "parts": [
        { 
          "text": "Analyze this security camera thumbnail. What is the primary subject in the image? Context: This is a residential camera. We own dogs that are [insert breed/size]. Look very closely at the snout, tail, and body shape before classifying a pet as a 'Cat'. Note: At night under infrared light, our dogs' eyes glow brightly, but they are still dogs, not cats or raccoons. Keep your response extremely concise, ideally just a single word (e.g., 'Person', 'Raccoon', 'Coyote', 'Cat', 'Bear', 'Dog', 'Squirrel', 'Vehicle'). If the frame is mostly empty, reply 'None'." 
        },
        { "inlineData": { "mimeType": "image/jpeg", "data": base64Image } }
      ]
    }]
  };
  
  const options = {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(requestBody),
    muteHttpExceptions: true 
  };
  
  try {
    const response = UrlFetchApp.fetch(url, options);
    if (response.getResponseCode() !== 200) return "API_ERROR";
    
    const data = JSON.parse(response.getContentText());
    if (data.candidates && data.candidates.length > 0) {
      return data.candidates[0].content.parts[0].text.trim();
    }
    return "Unknown";
  } catch (e) {
    return "API_ERROR";
  }
}

function sendEmailNotification(animalType, base64Image) {
  const imageBlob = Utilities.newBlob(Utilities.base64Decode(base64Image), "image/jpeg", "thumbnail.jpg");
  MailApp.sendEmail({
    to: NOTIFICATION_EMAIL,
    subject: `UniFi Protect Alert: ${animalType} Detected`,
    body: `Your camera detected a ${animalType}. See attached thumbnail.`,
    attachments: [imageBlob]
  });
}
