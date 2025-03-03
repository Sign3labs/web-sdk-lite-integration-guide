# Sign3 Web SDK Integration Guide

The Sign3 Web SDK is a JavaScript-based fraud prevention toolkit designed to assess browser security, detecting potential risks such as proxy usage, VPN connections, user agent spoofing, TOR connections, and more. By providing insights into the browser's security, it enhances protection against fraudulent activities and ensures robust security measures.

**Note:** The SDK only collects signals from the browser environment. It does not store or process them internally. The integrating party is responsible for encrypting and sending the collected signals to the backend for risk analysis.

---

## Adding Sign3 Web SDK to Your Project

1. Download the JavaScript agent from the CDN link and include it in your codebase as `sign3-web-sdk.js` (provided separately).
2. Call `sign3.initialize(config)` to initialize the JavaScript client for signal collection.
3. `sign3.initialize(config)` returns an object that includes the `get` method.
4. Wrap initialization in a try-catch block to handle errors in case of an invalid configuration.
5. After initialization, retrieve browser fingerprinting signals before triggering specific actions (e.g., Login, Payment, or Registration) using the `get` method.
6. Call `result.get(successCallback, errorCallback)`, passing two callbacks: one for success and one for error handling.
7. On success, the callback receives an object containing security signals that must be sent to the backend in encrypted form. Encryption must be implemented by the integrating party—an example is provided later in the document.
8. Contact the Sign3 team to download the [latest version](https://github.com/Sign3labs/web-sdk-lite-integration-guide/tree/main?tab=readme-ov-file#change-log) for integration.

---

## Bundling Into Your Code

Assuming you have the SDK inside a file named `sign3-web-sdk.js`:

### Initialization

To use the SDK, initialize it with the required parameters.

```javascript
import sdk from './sign3-web-sdk.js';
try {
  sdk.initialize({
    sessionId: 'your-unique-session-id', // Required: A unique identifier to track the user session.
    apiKey: 'your-api-key', // Required: API key for authentication (shared separately by Sign3).
    apiSecret: 'your-api-secret', // Required: Secret key for authentication (shared separately by Sign3).
  }).get(
  function(result) {
    console.log(result);
  },
  function(error) {
    console.error("Error:", error.message); // Example: "Something missing from required fields"
  }
);
} catch (error) {
  console.log("Initialization failed: ", error);
}
```

### Parameters

| Parameter   | Type     | Required | Description                                             |
|------------|----------|----------|---------------------------------------------------------|
| `sessionId` | `string` | Yes      | A unique session identifier to track the user session.  |
| `apiKey`    | `string` | Yes      | API key used for authentication (shared separately).    |
| `apiSecret` | `string` | Yes      | Secret key used for authentication (shared separately). |

---

## Fetching Browser Data

Once initialized successfully, use the `get` method to retrieve browser information.

```javascript
result.get(function(response) {
  console.log(response);
}, function(error) {
  console.log('Get error:', error.message);
});
```

---

## `get` Method Response Structure

The `get` method returns an object containing various security signals that must be sent to the backend in an encrypted format. Below is an example response structure:

```json
{
  "f": "refscfgg453ccx",
  "a": {"a": "", "b": "", "c": ""},
  "c": {"a": "cdddd-vddss", "b": null, "c": null, ...},
  "ze": {},
  "v": "lite",
  "p": null || "wdfddsx",
  "additionalParams": null
}
```

---

## Error Handling

Any error during a `get` call is handled using the `errorCallback` parameter in the `get` method.

### Example Error Handling

```javascript
result.get(function(response) {}, function(error) {
  console.log('Get error:', error.message);
});
```

---

## Encryption Example

The integrating party must encrypt the data returned by the `get` method before sending it to the backend. Below is an example of how encryption can be implemented using AES-CBC:

```javascript
const crypLib = window.crypto;
const { subtle: sbtl } = crypLib || {};

const keySize = 128; // In bits
const iterationCount = 1000;

const generateKey = async (salt, passPhrase) => {
  const encoder = new TextEncoder();
  const keyMaterial = await sbtl.importKey(
    'raw',
    encoder.encode(passPhrase),
    { name: 'PBKDF2' },
    false,
    ['deriveKey']
  );
  return await sbtl.deriveKey(
    {
      name: 'PBKDF2',
      salt: encoder.encode(salt),
      iterations: iterationCount,
      hash: 'SHA-256',
    },
    keyMaterial,
    { name: 'AES-CBC', length: keySize },
    false,
    ['encrypt', 'decrypt']
  );
};

export const encrypt = async (salt, passPhrase, data) => {
  const key = await generateKey(salt, passPhrase);
  const iv = crypLib.getRandomValues(new Uint8Array(16)); // 128-bit IV
  const encoder = new TextEncoder();
  const encrypted = await sbtl.encrypt(
    {
      name: 'AES-CBC',
      iv: iv,
    },
    key,
    encoder.encode(data)
  );
  const encData = btoa(String.fromCharCode(...new Uint8Array(encrypted)));
  const backendIv = Array.from(iv).map((b) => b.toString(16).padStart(2, '0')).join('');
  return {
    encodedData: encData,
    iv: backendIv,
  };
};
```

## Sending Data to Backend

```javascript
const authHeader = `${apiKey}:${apiSecret}`;
const headers = {
  Authorization: `Basic ${window.btoa(authHeader)}`,
  'TENANT-ID': iv,
  'Content-Type': 'text/plain',
  'client-ip-forwarded': "3.25.124.56",
  'client-ts-millis': Date.now(),
};
```

### curl

```
curl 'https://intelligence-b.sign3.in/v1/userInsights/web?cstate=true' \
  -H 'accept: */*' \
  -H 'accept-language: en-GB,en;q=0.9' \
  -H 'authorization: Basic U2lnbjNUZXN0QXBwOlNpZ24zbGFic0BpbnRlcm5hbA==' \
  -H 'client-ip-forwarded: 182.74.33.58' \
  -H 'client-ts-millis: 1740991469037' \
  -H 'content-type: text/plain' \
  -H 'origin: http://localhost:3000' \
  -H 'priority: u=1, i' \
  -H 'referer: http://localhost:3000/' \
  -H 'sdk-version-code: 1' \
  -H 'sdk-version-name: 1.0.0' \
  -H 'sec-ch-ua: "Not(A:Brand";v="99", "Google Chrome";v="133", "Chromium";v="133"' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'sec-ch-ua-platform: "macOS"' \
  -H 'sec-fetch-dest: empty' \
  -H 'sec-fetch-mode: cors' \
  -H 'sec-fetch-site: cross-site' \
  -H 'tenant-id: 933c43f3528f5d813a8197da7ad9119e' \
  -H 'user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36' \
  --data-raw '+HL9FSxzTYwiVa5EIYjKMYq4hOHolRXmEm40NtOsRQTDf5iENq7D+ROSWk+3w4Dy4twTw8PM+6snVfoHfEpiwb4hQgaQUBz2j9561WxVl9claxlTN87N3HBoR9KhBN0PQxAu8l/axev1jT2HwfL4cbFB7/4q9BuoSV7qWf6LQ7ODeODhV5vvLIp6+zUBiM0VIcqqlv43zM18FfUuuK91/dzpoiU8SFGgEcmaKsha7V6ID9pjFLkT7ij+zE+R+pP0BhvSc9FGuuvrHr8h5gyoXrlK4X3++oq22pMpBHZhkgnP15KcPA/DMN0maR691W6hafjrYBPOr4kekmYlNsqPMmOv1w67l/9TLdxlklKERFetd2kFE5861IqIKVIK0U9/63RNhkPkTWuEPTcXb40DUc1pU2DhbpBd7Uv01C+E3okgklKQBJnFxTLasjfuvoYyGHKX/O52GrQrsNLL016EJ99o+zPP1AVV69bn6uztrGlQsL7rN0Qij+Z/nnLf/WsFbZUno24Qy2lNaP7WirdCgyUGWXitRNg78/ApQe/JF6yDYQ71+mgZ/JHVcKARNpK3cp/iqwZ3kGs4a6HmRc9GGtzjjdWZEYtA7aU87HS+5O7Pp3aEpBAuDZTPxXarbV+5AiFI68ERCjuWJktmpkA5EMcgaDDHVKHRv4Ged9QubOjD0+nPLIOa/lku+btAwipcw26xQRjEo/EAIjQsMDXIzqt11FU86P48/ReNnUVN+7leVALeMkyvDR9iK2DyAdNy5QM9J/PnJ6IA8qJkgZ9JTgCaqX44b5GlcdousV45yJyg5Dltib1Ib+RSGnb67DLrR8CdJFpkjqrRvnZk2fpzlYVBo0try56uFtvOxR2Z89woAfFO9PO7zPVFAHoxqbxEUfqgJAxgu5zWAXugXgKiAxYtfPoV44TSAQLHZaWTxTx7BzCL9eSDP7hg3t8bGRocRNHk4TL4O4sTrH97cXUgQXkBaJrtugCMwyHMQaOh8cLD1e/y6wf7AB6qUokdcUQcVsnCSi1m0jVZ7O4g97Qb5iJ601vKZtxhvukr8P42sgnHXLk+vqr1ikpSZiQvS9Xb+5EZKdQ9HHVEaqIUBNNhX9hNLGMhT+xUNstjyYwkdXgw6ZQNdzlQ+7RvBHgWAifIC++pLXtrCWJWC3ql/gXivPH/e7cVrtl4WBZrJfloly7/zIvalmT8XL6kYxYRZSJ0MkK1WVFhrXK/usehMMWGSwpUx6UYxUqcXkVfObMw5CH4btoS0kOEjnKqYx95dx6Zb3XKHTqAXh27UjQ6bT4DtDS3W8f/BQc0ZrKVaMDXKbnWtadkTaJy41mDOiX329oa23G6Gdyb7UWN7MbpivFMgiEvXHwA38nXmWGJicDgUwSte1qzxZ0YI8u3X/dFgn7xM+0rgZI5/AcMoL+ZTSkmXmrlZA+sYy8GVc1vjDW0ROAE7BzgXW7gEPe3DoBPTx1u5qpShdM8nkzqCxYN/sdw2ARTTG4BmdLkvJbFMDQerpo1B0oBthqc56X2PVYax577G5vwnQIU8Re9MXRpefA7ImeIKmCTDv5tve7I5sPtiTQzQ4i7bLoZ6StQbZqWNo1gFfICDAgz4Jq03JN+EBSbB5C7sNOs6jJd4YLWn1rljkA4fLFDD2XwXuUpH/zpw/wi7YzQNgj9si6VU8YT1FQLtJvQOyqoKWrNPFocpznQP4MGTv/KktPzMWo4FcR974uM'
```

**Note:** The `client-ip-forwarded` header is used to pass the client IP address received at the integrating party's backend to the Sign3 backend to retrieve IP intelligence from Sign3.

---

## Browser Fingerprint Response Structure

This section provides a detailed explanation of the fields included in the browser fingerprint response JSON.

### JSON Structure

```json
{
  "requestId": "f60f48bf-890f-4274-a050-2cfa9d3ad33d",
  "newDevice": false,
  "fingerprint": "018fff6b-d4a3-4c67-ba53-b8bdce2cac3d",
  "sessionId": "17dnu81f-890f-adff-a050-2cfa9d3ad33d",
  "createdAt": 1738141661,
  "riskScore": "Medium",
  "firstSeenDays": 10,
  "ipIntelligence": {
    "city": "New Delhi",
    "region": "National Capital Territory of Delhi",
    "country": "IN",
    "latitude": 28.60000038,
    "longitude": 77.19999695,
    "isVPN": false,
    "isTor": false,
    "isProxy": false,
    "ip": "14.140.38.186"
  },
  "browserDetections": {
    "isIncognito": false,
    "isPrivacyFocused": true,
    "isBotDetected": false,
    "isAdBlockerEnabled": null, 
    "isUserAgentSpoofed": true
  }
}
```

### Field Descriptions

### General Information
- **requestId** *(string)*: A unique identifier for this fingerprint request.
- **newDevice** *(boolean)*: Indicates whether this is a newly detected device (`true`) or a previously seen device (`false`).
- **fingerprint** *(string)*: A unique identifier generated for the browser and device combination based on various fingerprinting techniques.
- **sessionId** *(string)*: A unique session identifier for tracking user activity during a session.
- **createdAt** *(integer)*: Timestamp (Unix format) representing when this fingerprint was generated.
- **riskScore** *(string)*: Indicates the risk level of the detected fingerprint (e.g., "Low", "Medium", "High") triangulated with ip data and other malicious signals 
- **firstSeenDays** *(integer)*: The number of days since this device was first seen.

### IP Intelligence
- **city** *(string)*: The city where the detected IP is located.
- **region** *(string)*: The administrative region associated with the IP.
- **country** *(string)*: The country code (ISO 3166-1 alpha-2) where the IP originates.
- **latitude** *(float)*: Approximate latitude coordinate of the detected IP.
- **longitude** *(float)*: Approximate longitude coordinate of the detected IP.
- **isVPN** *(boolean)*: Indicates whether the detected IP is associated with a VPN service.
- **isTor** *(boolean)*: Indicates whether the detected IP is part of the Tor network.
- **isProxy** *(boolean)*: Indicates whether the detected IP is using a proxy server.
- **ip** *(string)*: The IP address of the detected user.

### Browser Detections
- **isIncognito** *(Boolean)*: Indicates whether the browser is in incognito or private browsing mode.
- **isPrivacyFocused** *(Boolean)*: Indicates whether the browser is privacy focused. It has built-in ad blocking, fingerprinting protection, or other privacy-focused modifications.
- **isBotDetected** *(Boolean)*: Identifies if the user is a bot or an automated script.
- **isAdBlockerEnabled** *(Boolean)*: Determines if an ad blocker is active in the browser.
- **isUserAgentSpoofed** *(Boolean)*: Detects if the user agent string has been modified to disguise the browser's identity.

**Note:** All signals are collected in real-time when the `get` call is made.

---

## Change Log

### Version 1.0.0

- Initial release of Sign3 Web SDK integration guide.
- Added `initialize` and `get` method details.
- Included browser fingerprint response structure and field explanations.
- Provided error handling examples.

