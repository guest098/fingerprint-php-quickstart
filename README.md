
# Fingerprint PHP Quickstart

## Overview

In this quickstart, you'll implement a basic PHP backend using the [Fingerprint PHP SDK](https://github.com/fingerprintjs/fingerprint-pro-server-api-php-sdk) to prevent fraudulent account creation. You'll learn how to integrate with Fingerprint's identification API, store visitor information, and block duplicate account signups from the same device.

The goal is to walk through the steps in a developer-friendly and readable way—this is not intended to be a production-ready app, but a solid starting point for integrating Fingerprint fraud detection.

> Estimated time to complete: under 10 minutes

---

## Prerequisites

Before you begin, you should have:

- PHP 8.0 or later
- Composer installed
- SQLite (or similar lightweight DB)
- A front-end implementation that sends the `requestId` to the backend (see [Fingerprint JS Quickstart](https://dev.fingerprint.com/docs/quickstart))
- A Fingerprint Pro account (free trial available)

---

## Step 1: Get your Fingerprint secret API key

1. Log in to the [Fingerprint Dashboard](https://dashboard.fingerprint.com)
2. Go to **API Keys**
3. Copy your **Secret API Key** (you’ll need this on the server)

---

## Step 2: Set up your PHP project

Start with a minimal PHP project structure.

```bash
mkdir fingerprint-php && cd fingerprint-php
composer init --name="yourname/fingerprint-demo" --require=fingerprintjs/fingerprint-pro-server-api-php-sdk
composer require vlucas/phpdotenv
```

Then, create the following files:

```bash
touch index.php .env database.sqlite
```

Update `.env`:

```env
FINGERPRINT_SECRET_API_KEY=your_secret_key_here
```

---

## Step 3: Initialize the database

We'll use SQLite to store created accounts and track `visitorId`.

Create `setup.php`:

```php
<?php
$db = new PDO('sqlite:database.sqlite');
$db->exec("CREATE TABLE IF NOT EXISTS accounts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT NOT NULL,
  password TEXT NOT NULL,
  visitorId TEXT NOT NULL
)");
echo "Database setup complete.";
```

Run it once:

```bash
php setup.php
```

---

## Step 4: Set up a basic PHP server

Now we’ll write a simple PHP server to handle account creation requests.

Start by editing `index.php`:

```php
<?php
require 'vendor/autoload.php';

use Dotenv\Dotenv;

header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Methods: POST");
header("Access-Control-Allow-Headers: Content-Type");

$dotenv = Dotenv::createImmutable(__DIR__);
$dotenv->load();

$db = new PDO('sqlite:database.sqlite');

// Basic routing
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/api/create-account') {
    // We'll add Fingerprint integration here next
    echo json_encode(['message' => 'Account endpoint reached']);
}
```

---

## Step 5: Install and configure the Fingerprint PHP SDK

Instead of calling the HTTP API manually, use the official SDK.

1. Require the SDK (already done via Composer):

```bash
composer require fingerprintjs/fingerprint-pro-server-api-php-sdk
```

2. Initialize the Fingerprint client in your script:

```php
use Fingerprint\ServerAPI\Client;
use Fingerprint\ServerAPI\Region;

$client = new Client($_ENV['FINGERPRINT_SECRET_API_KEY'], Region::GLOBAL);
```

---

## Step 6: Handle account creation and Fingerprint verification

Now let’s connect everything together inside your `/api/create-account` handler.

Here’s how it works:

1. Receive the `requestId`, `username`, and `password` from the frontend
2. Call `getEvent()` from the SDK
3. Get the `visitorId` and check bot status
4. Check your DB for an existing `visitorId`
5. Block or insert accordingly

Update your `index.php`:

```php
<?php
require 'vendor/autoload.php';

use Dotenv\Dotenv;
use GuzzleHttp\Exception\RequestException;
use Fingerprint\ServerAPI\Client;
use Fingerprint\ServerAPI\Region;

header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Methods: POST");
header("Access-Control-Allow-Headers: Content-Type");

$dotenv = Dotenv::createImmutable(__DIR__);
$dotenv->load();

$db = new PDO('sqlite:database.sqlite');

if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/api/create-account') {
    $input = json_decode(file_get_contents("php://input"), true);

    $username = $input['username'] ?? null;
    $password = $input['password'] ?? null;
    $requestId = $input['requestId'] ?? null;

    if (!$username || !$password || !$requestId) {
        http_response_code(400);
        echo json_encode(["error" => "Missing fields"]);
        exit;
    }

    $client = new Client($_ENV['FINGERPRINT_SECRET_API_KEY'], Region::GLOBAL);

    try {
        $event = $client->getEvent($requestId);
        $visitorId = $event->products->identification->data->visitorId ?? null;
        $botResult = $event->products->botd->data->bot->result ?? 'notDetected';

        if ($botResult !== 'notDetected') {
            http_response_code(403);
            echo json_encode(['error' => 'Bot detected']);
            exit;
        }

        $check = $db->prepare("SELECT COUNT(*) FROM accounts WHERE visitorId = ?");
        $check->execute([$visitorId]);

        if ($check->fetchColumn() > 0) {
            http_response_code(429);
            echo json_encode(['error' => 'Device has already registered']);
            exit;
        }

        $insert = $db->prepare("INSERT INTO accounts (username, password, visitorId) VALUES (?, ?, ?)");
        $insert->execute([$username, $password, $visitorId]);

        echo json_encode(['status' => 'Account created']);
    } catch (Exception $e) {
        http_response_code(500);
        echo json_encode(['error' => 'Fingerprint verification failed']);
    }
}
```

---

## Step 7: Test the API

Start the PHP server:

```bash
php -S localhost:8000
```

Use Postman, cURL, or your front-end to test:

**POST** `http://localhost:8000/api/create-account`

```json
{
  "username": "alice",
  "password": "1234",
  "requestId": "fp_request_id_here"
}
```

If you submit again from the same device, it should return an error.

---

## Notes

- This example stores passwords as plain text – do **not** do this in production. Use proper hashing (e.g., bcrypt).
- CORS headers are added for testing. Lock these down in production.
- The SDK gives you access to more signals like VPN detection, incognito mode, etc.

---

## References

- [Fingerprint PHP SDK on GitHub](https://github.com/fingerprintjs/fingerprint-pro-server-api-php-sdk)
- [API Docs – Events](https://dev.fingerprint.com/reference/server-api-get-event)
- [Front-End Quickstart](https://dev.fingerprint.com/docs/quickstart)
