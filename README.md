
# Fingerprint PHP Quickstart

## 🧠 Overview

In this quickstart, you’ll build a PHP server that prevents fraudulent account creation using the Fingerprint Server API and SQLite.

The example use case is stopping **new account fraud**, where attackers create multiple fake accounts from the same device. You’ll use Fingerprint to identify each signup attempt and block duplicate signups or suspicious devices.

> Estimated time: < 10 minutes

## ✅ Prerequisites

Before starting, make sure you have:

- PHP 8.0 or later
- Composer (PHP package manager)
- SQLite3 installed
- A completed front-end Fingerprint implementation that sends the `requestId` to your server

## 🔑 1. Get your secret API key

1. Sign in to the [Fingerprint Dashboard](https://dashboard.fingerprint.com/api-keys)
2. Create a new **secret API key**
3. Save this for use in your `.env` file later

## 🛠️ 2. Project Setup

1. Create a new project folder and initialize with Composer:

```bash
mkdir fingerprint-php && cd fingerprint-php
composer init
composer require vlucas/phpdotenv guzzlehttp/guzzle
```

2. Create your project files:

```bash
touch index.php database.sqlite .env
```

3. Create a `.env` file to store your API key:

```env
FINGERPRINT_SECRET_API_KEY=your_api_key_here
```

## 🧱 3. Database Setup

We’ll use SQLite to store accounts and prevent duplicate signups.

```php
<?php
// setup.php
$db = new PDO('sqlite:database.sqlite');

$db->exec("CREATE TABLE IF NOT EXISTS accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT,
    password TEXT,
    visitorId TEXT
)");
echo "Database initialized.";
```

Run once:

```bash
php setup.php
```


## 🚀 4. Set up the PHP server

We'll create a minimal backend using PHP's built-in development server. This will handle `POST` requests to `/create-account`.

Create a file named `index.php` with the following initial setup:

```php
<?php
require 'vendor/autoload.php';

use Dotenv\Dotenv;

header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Headers: Content-Type");
header("Access-Control-Allow-Methods: POST");

$dotenv = Dotenv::createImmutable(__DIR__);
$dotenv->safeLoad();

// Database connection
$db = new PDO('sqlite:database.sqlite');

// Listen for POST requests at /create-account
if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/create-account') {
    echo json_encode(['message' => 'Server is working. Ready to handle account creation.']);
}
?>
```

Now run the server:

```bash
php -S localhost:8000
```

You should see a success message when you send a POST request to `/create-account`.

---

## 🔗 5. Integrate Fingerprint into the server

Next, update the `/create-account` route to fetch and validate the `requestId` from the Fingerprint API and check for fraud.

Update your `index.php` like so:

```php
<?php
require 'vendor/autoload.php';

use Dotenv\Dotenv;
use GuzzleHttp\Client;

header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Headers: Content-Type");
header("Access-Control-Allow-Methods: POST");

$dotenv = Dotenv::createImmutable(__DIR__);
$dotenv->safeLoad();

$apiKey = $_ENV['FINGERPRINT_SECRET_API_KEY'];
$db = new PDO('sqlite:database.sqlite');

if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/create-account') {
    $input = json_decode(file_get_contents('php://input'), true);
    $requestId = $input['requestId'] ?? null;
    $username = $input['username'] ?? null;
    $password = $input['password'] ?? null;

    if (!$requestId || !$username || !$password) {
        http_response_code(400);
        echo json_encode(['error' => 'Missing required fields']);
        exit;
    }

    $client = new Client([
        'base_uri' => 'https://api.fpjs.io',
        'headers' => ['Authorization' => "Bearer $apiKey"]
    ]);

    try {
        $response = $client->get("/events/$requestId");
        $event = json_decode($response->getBody(), true);
        $visitorId = $event['products']['identification']['data']['visitorId'];
        $botResult = $event['products']['botd']['data']['bot']['result'] ?? 'notDetected';

        if ($botResult !== 'notDetected') {
            http_response_code(403);
            echo json_encode(['error' => 'Bot detected. Account creation blocked.']);
            exit;
        }

        $stmt = $db->prepare("SELECT COUNT(*) FROM accounts WHERE visitorId = ?");
        $stmt->execute([$visitorId]);
        $count = $stmt->fetchColumn();

        if ($count > 0) {
            http_response_code(429);
            echo json_encode(['error' => 'Duplicate account from same device detected.']);
            exit;
        }

        $insert = $db->prepare("INSERT INTO accounts (username, password, visitorId) VALUES (?, ?, ?)");
        $insert->execute([$username, $password, $visitorId]);

        echo json_encode(['status' => 'Account created successfully']);
    } catch (Exception $e) {
        http_response_code(500);
        echo json_encode(['error' => 'Failed to verify requestId', 'details' => $e->getMessage()]);
    }
}
?>
```



Edit `index.php`:

```php
<?php
require 'vendor/autoload.php';

use Dotenv\Dotenv;
use GuzzleHttp\Client;

header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Headers: Content-Type");
header("Access-Control-Allow-Methods: POST");

$dotenv = Dotenv::createImmutable(__DIR__);
$dotenv->safeLoad();

$apiKey = $_ENV['FINGERPRINT_SECRET_API_KEY'];
$db = new PDO('sqlite:database.sqlite');

if ($_SERVER['REQUEST_METHOD'] === 'POST' && $_SERVER['REQUEST_URI'] === '/create-account') {
    $input = json_decode(file_get_contents('php://input'), true);
    $requestId = $input['requestId'] ?? null;
    $username = $input['username'] ?? null;
    $password = $input['password'] ?? null;

    if (!$requestId || !$username || !$password) {
        http_response_code(400);
        echo json_encode(['error' => 'Missing required fields']);
        exit;
    }

    $client = new Client([
        'base_uri' => 'https://api.fpjs.io',
        'headers' => ['Authorization' => "Bearer $apiKey"]
    ]);

    try {
        $response = $client->get("/events/$requestId");
        $event = json_decode($response->getBody(), true);
        $visitorId = $event['products']['identification']['data']['visitorId'];
        $botResult = $event['products']['botd']['data']['bot']['result'] ?? 'notDetected';

        if ($botResult !== 'notDetected') {
            http_response_code(403);
            echo json_encode(['error' => 'Bot detected. Account creation blocked.']);
            exit;
        }

        $stmt = $db->prepare("SELECT COUNT(*) FROM accounts WHERE visitorId = ?");
        $stmt->execute([$visitorId]);
        $count = $stmt->fetchColumn();

        if ($count > 0) {
            http_response_code(429);
            echo json_encode(['error' => 'Duplicate account from same device detected.']);
            exit;
        }

        $insert = $db->prepare("INSERT INTO accounts (username, password, visitorId) VALUES (?, ?, ?)");
        $insert->execute([$username, $password, $visitorId]);

        echo json_encode(['status' => 'Account created successfully']);
    } catch (Exception $e) {
        http_response_code(500);
        echo json_encode(['error' => 'Failed to verify requestId', 'details' => $e->getMessage()]);
    }
}
?>
```

## 🧪 6. Test the API

1. Start the PHP server:

```bash
php -S localhost:8000 index.php
```

2. Use your frontend to send a `POST` request to:

```
http://localhost:8000/create-account
```

With this JSON body:

```json
{
  "username": "alice",
  "password": "1234",
  "requestId": "YOUR_REQUEST_ID_FROM_FRONTEND"
}
```

3. Try submitting again with the same device and see the error.

## 📝 Notes

- This example uses **plain text passwords** — in production, always **hash passwords**.
- We use **SQLite** to simplify setup — for production, use a secure database like PostgreSQL or MySQL.
- CORS headers are added for easy local testing.
- This server assumes your frontend correctly fetches the `requestId` from the Fingerprint JS Pro client.

## 📚 Resources

- [Fingerprint PHP SDK Docs](https://dev.fingerprint.com)
- [API Reference – Events](https://dev.fingerprint.com/reference/server-api-get-event)
- [Fingerprint Playground](https://demo.fingerprint.com/playground)
