# 🚀 OSL IAP Backend - Infrastructure Setup Guide

Welcome to the **OSL IAP Backend** integration guide. This document provides step-by-step instructions on configuring the necessary cloud infrastructure, developer consoles, and webhooks to connect your backend with Google Play and the Apple App Store.

---

## 🟢 Part 1: Google Play Store (Android) Setup

To verify purchases and receive real-time subscription updates, you must link Google Cloud Platform (GCP) with your Google Play Console.

### 1.1 Google Cloud Platform (GCP) & Service Account
1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project (or select an existing one).
3. Navigate to **APIs & Services > Library** and enable the **Google Play Android Developer API**.
4. Go to **IAM & Admin > Service Accounts** and click **Create Service Account**.
5. Give it a name (e.g., `iap-backend-service`) and grant it the **Viewer** role (or no role is fine for just API access, as Play Console permissions matter more).
6. Click on the created Service Account, go to the **Keys** tab, click **Add Key > Create new key**, and choose **JSON**.
7. Download the `.json` file. Save it safely and place it in your server directory (e.g., `Keys/google-service-account.json`).

### 1.2 Google Play Console (API Access)
1. Go to the [Google Play Console](https://play.google.com/console).
2. Navigate to **Setup > API access**.
3. Link the GCP project you created in step 1.1.
4. Scroll down to **Service Accounts**, find the account you just created, and click **Manage Play Console permissions**.
5. Grant the following permissions to the service account:
   * View app information and download bulk reports (Read-only)
   * View financial data, orders, and cancellation survey responses
   * Manage orders and subscriptions

### 1.3 Google Webhook (Real-Time Developer Notifications - RTDN)
1. In the **Google Cloud Console**, go to **Pub/Sub > Topics** and click **Create Topic** (e.g., `play-store-notifications`).
2. Add a permission to this topic:
   * **Principal:** `google-play-developer-notifications@system.gserviceaccount.com`
   * **Role:** Pub/Sub Publisher
3. Go to **Pub/Sub > Subscriptions** and create a subscription for your topic.
   * **Delivery Type:** Push
   * **Endpoint URL:** `https://your-domain.com/api/webhook/google`
   * **Authentication:** Enable **OIDC token** for the push subscription.
   * **Audience:** Set this to the exact value you will place in `Iap:Google:PubSubAudience`.
   * **Service Account:** Use a dedicated push invoker service account and place its email in `Iap:Google:PubSubServiceAccountEmail`.
4. Go back to the **Google Play Console**, navigate to **Monetize > Setup > Monetization setup**.
5. In the **Real-time developer notifications** section, enter your Pub/Sub Topic name and save.

> The backend rejects Google webhook calls that do not include a valid Pub/Sub OIDC bearer token. A plain unauthenticated push subscription will not work.

---

## 🍎 Part 2: Apple App Store (iOS) Setup

Apple uses the App Store Server API V2 for verification and App Store Server Notifications V2 for webhooks.

### 2.1 Apple Developer Portal (Generating Keys)
1. Go to the [Apple Developer Portal](https://developer.apple.com/) and log in.
2. Navigate to **Certificates, Identifiers & Profiles > Keys**.
3. Click the **+** button to create a new key.
4. Name the key (e.g., `OSL IAP Server Key`) and check the **In-App Purchase** box.
5. Click **Continue** and then **Register**.
6. Download the `.p8` file. **(Warning: You can only download this file once. Do not lose it!)**
7. Place the `.p8` file in your server directory (e.g., `Keys/AuthKey.p8`).
8. Take note of the **Key ID** (from the Keys page) and your **Issuer ID** (found in App Store Connect -> Users and Access -> Keys -> App Store Connect API).

### 2.2 App Store Connect (App Setup)
1. Go to [App Store Connect](https://appstoreconnect.apple.com/).
2. Select your app under **My Apps**.
3. Make sure your **Bundle ID** matches the one configured in your Unity project and backend `appsettings.json`.
4. Set up your In-App Purchases (Consumables, Auto-Renewable Subscriptions, etc.) under the **Features > In-App Purchases** section.

### 2.3 Apple Webhook (App Store Server Notifications V2)
1. In App Store Connect, go to your app and navigate to **App Information** (under the General section).
2. Scroll down to **App Store Server Notifications**.
3. In both the **Production Server URL** and **Sandbox Server URL**, enter your webhook endpoint:
   * `https://your-domain.com/api/webhook/apple`
4. Select **Version 2** for the notification format.
5. Save your changes.

### 2.4 Apple Root Certificate for JWS Verification
1. Download the Apple Root CA certificate used for App Store JWS verification from Apple's certificate authority page.
2. Store the certificate file on the server (example: `Keys/AppleRootCA-G3.cer`).
3. Set `Iap:Apple:RootCertPath` to that file path.

> The backend does not trust Apple webhook payloads unless the JWS certificate chain validates against this pinned root certificate.

---

## ⚙️ Part 3: Backend Configuration (`appsettings.json`)

Update your `appsettings.json` file on the server with the credentials gathered from the steps above:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=iap_database.db"
  },
  "Iap": {
    "Google": {
      "PackageName": "com.yourcompany.yourgame",
      "ServiceAccountKeyPath": "Keys/google-service-account.json",
      "PubSubAudience": "https://your-domain.com/api/webhook/google",
      "PubSubServiceAccountEmail": "pubsub-invoker@your-project.iam.gserviceaccount.com"
    },
    "Apple": {
      "BundleId": "com.yourcompany.yourgame",
      "IssuerId": "YOUR_APPLE_ISSUER_ID",
      "KeyId": "YOUR_APPLE_KEY_ID",
      "PrivateKeyPath": "Keys/AuthKey.p8",
      "RootCertPath": "Keys/AppleRootCA-G3.cer"
    }
  }
}
```

## 🔌 Feature-to-Configuration Mapping

| Feature | Required configuration |
|---|---|
| `GET /api/iap/history/{userId}` | Database connection only |
| `GET /api/iap/subscription/{userId}` | Database connection only |
| `POST /api/iap/google/verify` | `Iap:Google:PackageName`, `Iap:Google:ServiceAccountKeyPath` |
| `POST /api/iap/apple/verify` | `Iap:Apple:BundleId`, `Iap:Apple:IssuerId`, `Iap:Apple:KeyId`, `Iap:Apple:PrivateKeyPath`, `Iap:Apple:RootCertPath` |
| `POST /api/webhook/google` | `Iap:Google:PubSubAudience` and optionally `Iap:Google:PubSubServiceAccountEmail` |
| `POST /api/webhook/apple` | `Iap:Apple:RootCertPath` |

This means the read-only history/subscription APIs can run without Google/Apple key files, as long as the database is configured.

## ✅ What can be omitted in `appsettings.json` for simple verification

"Simple verification" here means: using only `POST /api/iap/google/verify` and/or `POST /api/iap/apple/verify`, without webhook endpoints.

### Commonly optional when webhooks are not used
- `Iap:Google:PubSubAudience`
- `Iap:Google:PubSubServiceAccountEmail`

### If you use Google verification only
Required:
- `Iap:Google:PackageName`
- `Iap:Google:ServiceAccountKeyPath`

Can be omitted:
- Entire Apple section (`BundleId`, `IssuerId`, `KeyId`, `PrivateKeyPath`, `RootCertPath`)
- Google Pub/Sub webhook fields above

### If you use Apple verification only
Required:
- `Iap:Apple:BundleId`
- `Iap:Apple:IssuerId`
- `Iap:Apple:KeyId`
- `Iap:Apple:PrivateKeyPath`
- `Iap:Apple:RootCertPath`

Can be omitted:
- Google `PackageName` and `ServiceAccountKeyPath`
- Google Pub/Sub webhook fields

### About `ConnectionStrings:DefaultConnection`
- It can be omitted for local default behavior.
- If omitted, backend falls back to `Data Source=iap_database.db`.

## ⚠️ Important Warnings & Pro Tips

### General Tips
* **HTTPS is Mandatory:** Both Google Pub/Sub and Apple Server Notifications require your webhook endpoints to be secured with a valid SSL/TLS certificate (HTTPS). Local testing requires tools like [ngrok](https://ngrok.com/) to expose your `localhost` via HTTPS.
* **Idempotency:** Webhooks can sometimes be delivered more than once due to network retries. The backend is designed to handle this, but never remove the database checks that prevent duplicate processing.
* **Duplicate Purchase Handling:** The verify API now returns a semantic success for `AlreadyProcessed`. The client must confirm the store order but must not grant the reward again when that result code is returned.

### Google Play Specifics
* **Propagation Delay:** When you grant permissions to your Service Account in the Play Console, it can take up to **24 to 48 hours** for the permissions to fully propagate. If you get a "Purchase not found" or `401 Unauthorized` error immediately after setup, wait a day and try again.
* **Testing Sandbox:** To test Google IAP without actual charges, add your testing Google accounts to the **License Testing** section in the Play Console. 

### Apple Specifics
* **The OriginalTransactionId Rule:** For Apple subscriptions, a new `TransactionId` is generated every time a subscription renews. However, webhooks only send the `OriginalTransactionId`. This backend uses the `OriginalTransactionId` as the Primary Key for subscriptions to ensure renewals are tracked correctly.
* **Sandbox vs Production:** When testing via Unity/Xcode using a Sandbox Tester account, the client **must** pass `IsSandbox = true` in the API request. Otherwise, the server will attempt to verify a test receipt on Apple's production servers, which will fail.
* **Sandbox Webhooks:** Apple's sandbox environment simulates time differently (e.g., a 1-month subscription renews every 5 minutes in sandbox). Use this to test your webhook's `DID_RENEW` and `EXPIRED` logic rapidly.

## 🗄️ Part 4: Database Management & Scaling

This backend uses **SQLite** by default for zero-configuration, out-of-the-box testing. However, because it is built on **Dapper** (a lightning-fast Micro-ORM), it is incredibly easy to scale up to enterprise databases like MySQL or PostgreSQL when your game grows.

### 4.1 Default SQLite Setup (Zero-Config)
* **Automatic Creation:** You do not need to set up a database server. When you run the API backend for the first time, the `InitializeDatabase()` method automatically creates a local file named `iap_database.db` in the root directory of the application.
* **How to view your data:** To view or edit the raw purchase records, download a free database viewer such as [DB Browser for SQLite](https://sqlitebrowser.org/) or [DBeaver](https://dbeaver.io/), and open the `iap_database.db` file.

### 4.2 Migrating to MySQL / SQL Server (Production Ready)
Since this project uses raw SQL and DAO patterns instead of a heavy ORM like Entity Framework, migrating to a larger database takes less than 5 minutes.

1. **Install the Database Provider:** Open your terminal and run: `dotnet add package MySql.Data` (for MySQL) or `Npgsql` (for PostgreSQL).
2. **Update the DAO (`SqlitePurchaseDao.cs`):** Simply replace `new SqliteConnection(_connectionString)` with `new MySqlConnection(_connectionString)`.
3. **Update `appsettings.json`:** Replace the SQLite file path with your actual database connection string.

*(Note: The SQL syntax used in this backend is standard ANSI SQL and is fully compatible with MySQL, PostgreSQL, and SQL Server with minimal to no changes.)*

---

## 🎮 Part 5: Unity Client First-Success Quickstart

This section is for getting the **first successful purchase verification** quickly, without chasing settings across multiple scripts.

### 5.0 Required Unity packages
For a clean Unity project or Asset Store validation project, install these packages before using the runtime demo flow:
* `com.unity.ugui` for the demo UI
* `com.unity.purchasing` for the runtime IAP flow and product catalog
* `com.unity.services.core` for Unity Services initialization

If these packages are missing, the package will still import, but the demo/runtime scripts stay in compatibility mode until the required packages are installed.

### 5.1 Use one shared config asset (`IapSetupConfig`)
In Unity:
1. Open `Tools > OSL > IAP API Tester`.
2. Click **Create Setup Asset**.
3. Fill these fields in `IapSetupConfig`:
   * `Server Url`
   * `Timeout Seconds`
   * `Environment Name` (`production` by default)
   * `Default User Id` (optional, but useful for the demo scene)
   * `Demo Consumable Product Id`
   * `Demo Subscription Product Id`
   * `Product Catalog`
   * `Apple Sandbox` policy (`UseDebugBuildForAppleSandbox` / `ForceAppleSandbox`)
   * Retry options

### 5.2 Scene wiring checklist
1. Assign the same `IapSetupConfig` asset to:
   * `IapManager`
   * `BackendApiClient`
2. Assign references in `IapManager`:
   * `BackendApiClient`
3. Assign references in `IapDemoUI`:
   * `iapManager`
   * `logText`
   * `goldText`
   * `vipText`
   * `logScrollRect`
   * `userIdInput`
   * `loginStatusText` (optional)
4. Bind UI buttons:
   * Login button → `OnClickLogin`
   * Buy Gold button → `OnClickBuyGold`
   * Buy VIP button → `OnClickBuyVIP`

### 5.3 Verify API response contract (required)
`PurchaseResponse` includes:
* `ResultCode` (`Verified`, `AlreadyProcessed`, `VerificationFailed`)
* `ShouldConfirmPurchase`
* `ShouldGrantReward`

Client must behave exactly as below:
1. If `ShouldConfirmPurchase == true`, call `ConfirmPurchase`.
2. If `ShouldGrantReward == true`, grant item **exactly once**.
3. If `ResultCode == AlreadyProcessed`, confirm the pending order, but **do not grant reward again**.

### 5.4 First-success test path
1. Enter User ID and click login.
2. Trigger a store test purchase.
   * For Apple, this means a sandbox purchase.
   * For Google Play, use the store's test purchase flow.
3. Confirm client log shows server verification result.
4. Confirm reward is granted once.
5. Confirm store order is confirmed (`ConfirmPurchase`).
6. Repeat same transaction recovery scenario and confirm no duplicate reward.
