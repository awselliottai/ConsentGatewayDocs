# Consent Gateway Data Lineage 1.0

### This is an example document to show the way timestamps could be implemented throughout the entirety of the Consent Gateway framework, from user preferences, to device storage, and server retrieval via the Camara API.

## Part 1: Time Stamping from the CMP UI to the Device (Mobile is shown, but could easily be implemented for a non-mobile device)

### 1. User interacts with CMP to generate their consent string:

### Example Javascript showing the user consent data capture along with a timestamp generated

// JavaScript (React/Vanilla) - Capturing user interactions

<code>const logEvent = (eventName, metadata = {}) => {


  const timestamp = new Date().toISOString(); // Capture current timestamp
  console.log(`[${timestamp}] Event: ${eventName}`, metadata);</code>

### Optional: Example - Send to a separate server to track/store data lineage
<code>
  fetch('/api/log', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ event: eventName, timestamp, metadata })
  });
};
</code>

### Example generate and store Consent String to Device (not official TypeScript/JavaScript Code from repo's)

<code>
const generateConsentString = (preferences) => {
  const consentString = btoa(JSON.stringify(preferences)); // Base64 encoding for storage
  const timestamp = new Date().toISOString();
  
  return { consentString, timestamp };
};
</code>
<code>
// Example: User selects preferences in CMP user interface, which generates consent string with timestamp
const { consentString, timestamp } = generateConsentString({
  consentType1: true,
  consentType2: false,
  analyticsType1: true,
  trackingType1: false,
  trackingType2: true,
  personalization: true
});

console.log(`[${timestamp}] Consent String Generated:`, consentString);
</code>
## 2. Consent string is stored to the user's mobile device:

### Example kotlin code to store consent string to Android EncryptedSharedPreferences w/ Timestamp:
<code>
val sharedPreferences = EncryptedSharedPreferences.create(
    "consent_prefs",
    MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
    context,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

val editor = sharedPreferences.edit()
val timestamp = System.currentTimeMillis().toString()

// Store consent string with timestamp
editor.putString("consent_string", consentString)
editor.putString("timestamp", timestamp)
editor.apply()
</code>

### Example swift code to store consent string to iOS Keychain w/ Timestamp:
<code>
val sharedPreferences = EncryptedSharedPreferences.create(
    "consent_prefs",
    MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
    context,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

val editor = sharedPreferences.edit()
val timestamp = System.currentTimeMillis().toString()

// Store consent string with timestamp
editor.putString("consent_string", consentString)
editor.putString("timestamp", timestamp)
editor.apply()
</code>

## Part 2 - Time stamps are also implemented for valid logging per every API call and to enforce the security matrix.

## 3. Backend - API Endpoint Example with Timestamps:

<code>
const express = require('express');
const app = express();
const bodyParser = require('body-parser');

// Middleware to parse JSON request body - part of GPP/TABCF development suite
app.use(bodyParser.json());

// Endpoint to receive consent data - Camara Endpoints
app.post('/api/consent/sync', (req, res) => {
  const { consentString, timestamp, deviceId } = req.body;

  if (!consentString || !timestamp || !deviceId) {
    return res.status(400).json({ error: 'Missing required fields.' });
  }

  console.log(`[${timestamp}] Consent Synced for Device: ${deviceId}`);
  
  // Store the consent data in a database (or log) - displays saving to memory for example purposes.
  const consentLog = { consentString, timestamp, deviceId };
  console.log('Consent Log:', consentLog);

  res.status(200).json({ status: 'Success', message: 'Consent synced.' });
});
</code>

### Example of exposing associated server port

<code>
// Start the server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
</code>

## Part 3: Syncing will also need to take place in order to keep the consent states valid and concentric.

### Example syncing Logic in kotlin for Android Consent State retrieval w/ Timestamps:

<code>
import okhttp3.*
import org.json.JSONObject
import java.io.IOException

val client = OkHttpClient()

fun syncConsentData(serverUrl: String) {
    val sharedPreferences = EncryptedSharedPreferences.create(
        "consent_prefs",
        MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
        context,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    val consentString = sharedPreferences.getString("consent_string", "") ?: ""
    val timestamp = sharedPreferences.getString("timestamp", "") ?: ""
    val deviceId = "1234-5678-ABCD"  // Example device ID

    // Prepare JSON request body
    val jsonBody = JSONObject()
    jsonBody.put("consentString", consentString)
    jsonBody.put("timestamp", timestamp)
    jsonBody.put("deviceId", deviceId)

    val requestBody = RequestBody.create(
        MediaType.get("application/json; charset=utf-8"),
        jsonBody.toString()
    )

    val request = Request.Builder()
        .url(serverUrl)
        .post(requestBody)
        .build()

    // Execute the request
    client.newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            println("Failed to sync consent: ${e.message}")
        }

        override fun onResponse(call: Call, response: Response) {
            if (response.isSuccessful) {
                println("Consent synced successfully: ${response.body()?.string()}")
            } else {
                println("Failed to sync: ${response.code()}")
            }
        }
    })
}

// Example usage (for illustrative purposes):
syncConsentData("http://localhost:3000/api/consent/sync")

</code>

### Example syncing logic in Swift for iOS Consent State Retrieval w/ Time Stamps:

<code>
import Foundation

func getConsentDataFromKeychain(account: String) -> String? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne
    ]

    var item: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &item)

    guard status == errSecSuccess, let data = item as? Data else {
        return nil
    }

    return String(data: data, encoding: .utf8)
}

func syncConsentData() {
    guard
        let consentString = getConsentDataFromKeychain(account: "consent_string"),
        let timestamp = getConsentDataFromKeychain(account: "timestamp")
    else {
        print("Failed to retrieve consent data.")
        return
    }

    let deviceId = "5678-1234-XYZ" // Example device ID
    let json: [String: Any] = [
        "consentString": consentString,
        "timestamp": timestamp,
        "deviceId": deviceId
    ]

    let jsonData = try? JSONSerialization.data(withJSONObject: json)

    let url = URL(string: "http://localhost:3000/api/consent/sync")!
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = jsonData

    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            print("Failed to sync consent: \(error.localizedDescription)")
            return
        }

        if let response = response as? HTTPURLResponse, response.statusCode == 200 {
            print("Consent synced successfully.")
        } else {
            print("Failed to sync consent.")
        }
    }

    task.resume()
}

// Example usage (for illustrative purposes)
syncConsentData()

</code>

## Part 4: Logging is a critical portion of the overall framework, which should also necessarily include the proper time stamping for retrieval and storage:

### Data Lineage Sync Log in Server Console Example:

<code>
[2024-10-24T15:22:10Z] Consent Synced for Device: 1234-5678-ABCD
Consent Log: { consentString: "eyJhZHMiOnRydWUsICJhbmFseXRpY3MiOmZhbHNlLCAicGVyc29uYWxpemF0aW9uIjp0cnVlfQ==", timestamp: "2024-10-24T15:21:05Z", deviceId: "1234-5678-ABCD" }
</code>

## Part 5: Example overview of data lineage with Timestamping:

### 1. Consent State Creation and related device - Expiration is included but may need to be handled on api and server side for true time-bound validation & verification:

<code>
{
  "consent_creation": {
    "user_id": 123,
    "created_at": "2024-10-24T10:30:12Z",
    "expires_at": 2025-01-01T00:00:00Z",
    "consent_string": "abc123...",
    "platform": "iOS"
  },
</code>

### 2. Consent is retreived as part of security & scoping matrix (not necessarily for ad bidding):
<code>
  "consent_retrieval": {
    "retrieved_at": "2024-10-24T11:00:05Z",
    "retrieved_by": "browser_app"
  },

  "server_request": {
    "request_at": "2024-10-24T11:01:00Z",
    "validity_checked_at": "2024-10-24T11:01:01Z",
    "result": "Access granted",
    "scope_matrix": ["scopes", "purposes", "authorization"]
  }
}

</code>

### 3. The server produces and stores the time-stamped logs per user or authenticator request:

<code>
{
  "user_id": 123,
  "consent_string": "abc123...",
  "request_at": "2024-10-24T11:01:00Z",
  "validity_checked_at": "2024-10-24T11:01:01Z",
  "result": "Access granted"
}

</code>
