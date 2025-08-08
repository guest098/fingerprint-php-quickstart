
# Fingerprint PHP Server Quickstart

## Overview

In this quickstart, you’ll add [Fingerprint](https://fingerprint.com) to a PHP server to prevent fraudulent account creation.

The example use case is **new account fraud** — when attackers create multiple fake accounts from the same device to exploit systems. You’ll identify the device for each signup and block duplicates or suspicious activity.

You’ll learn how to:

- Set up a PHP server with Fingerprint integration
- Retrieve visitor identification data using the PHP Server SDK
- Block bots and suspicious devices
- Prevent multiple signups from the same device

**Before starting**  
This guide focuses on the backend. You should already have a front-end or mobile Fingerprint implementation that generates a `requestId` and sends it to the server.

> Estimated time: < 10 minutes

---

## Prerequisites

- PHP 8.0+
- Composer
- SQLite3
- Basic PHP knowledge
- A completed front-end Fingerprint integration

---

## 1. Get your secret API key

1. Sign in to the [Fingerprint Dashboard](https://dashboard.fingerprint.com/api-keys)  
2. Create a **secret API key**  
3. Save it — you’ll use it to retrieve visitor data

---

## 2. Set up the project

```bash
mkdir fingerprint-php-quickstart && cd fingerprint-php-quickstart
composer init --name="demo/fp-php"
composer require fingerprintjs/fingerprint-pro-server-api-php-sdk vlucas/phpdotenv
```

Create the required files:

```bash
touch index.php setup.php .env database.sqlite
```

`.env`:

```env
FINGERPRINT_SECRET_API_KEY=your_secret_key_here
```

---

## 3. Initialize the database

`setup.php`:

```php
<?php
$db = new PDO('sqlite:database.sqlite');
$db->exec("
CREATE TABLE IF NOT EXISTS accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL,
    password TEXT NOT NULL,
    visitorId TEXT NOT NULL
)");
echo "Database ready.";
```

Run once:

```bash
php setup.php
```

---

## 4. Create a basic PHP server

`index.php`:

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

if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/api/create-account') {
    echo json_encode(['message' => 'Server is running']);
}
``

Run:

```bash
php -S localhost:8000
```

---

## 5. Fingerprint integration steps

We’ll now integrate the Fingerprint PHP SDK step-by-step.

### Step 5.1 – Import the SDK

At the top of `index.php`:

```php
use Fingerprint\ServerAPI\Client;
use Fingerprint\ServerAPI\Region;
```

---

### Step 5.2 – Set up the client

Still at the top of `index.php`, after loading environment variables:

```php
$client = new Client($_ENV['FINGERPRINT_SECRET_API_KEY'], Region::GLOBAL);
```

---

### Step 5.3 – Update `/api/create-account` to use Fingerprint

Replace the route handler:

```php
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/api/create-account') {
    $input = json_decode(file_get_contents('php://input'), true);
    $username = $input['username'] ?? null;
    $password = $input['password'] ?? null;
    $requestId = $input['requestId'] ?? null;

    if (!$username || !$password || !$requestId) {
        http_response_code(400);
        echo json_encode(['error' => 'Missing required fields']);
        exit;
    }

    // Step 5.4 – Fetch the Fingerprint event
    try {
        $event = $client->getEvent($requestId);
    } catch (Exception $e) {
        http_response_code(500);
        echo json_encode(['error' => 'Could not retrieve event']);
        exit;
    }

    // Step 5.5 – Use the event data
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
        echo json_encode(['error' => 'Duplicate account detected']);
        exit;
    }

    $insert = $db->prepare("INSERT INTO accounts (username, password, visitorId) VALUES (?, ?, ?)");
    $insert->execute([$username, $password, $visitorId]);

    echo json_encode(['status' => 'Account created']);
}
```

---

## 6. Test the implementation

Start the server:

```bash
php -S localhost:8000
```

From your frontend, send:

```json
{
  "username": "alice",
  "password": "secret",
  "requestId": "fp_request_id_here"
}
```

- First attempt → should return “Account created”  
- Second attempt from same device → should return “Duplicate account detected”  

---

## Notes

- Passwords are stored as plain text here for simplicity. Always hash in production.
- This uses SQLite for easy setup — use a production database in real apps.
- CORS is open for testing — restrict in production.

---

## References

- [Fingerprint PHP SDK](https://github.com/fingerprintjs/fingerprint-pro-server-api-php-sdk)  
- [Events API Reference](https://dev.fingerprint.com/reference/server-api-get-event)  
- [Node Server Quickstart](https://fingerprintjs.notion.site/Node-Server-Quickstart-Example-20102f125ebd8094ad2ec19a9b76cbe1)  
