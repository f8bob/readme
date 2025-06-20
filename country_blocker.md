# Country-Based Access Blocking

This README describes the end-to-end implementation of a country-based user blocking mechanism in PHP. It includes all the steps, file structure, code examples, and instructions for a new developer.

IMPORTANT: That's pseudo-code of main logic but it doesn't mean that this code will work
---

## Table of Contents

1. [Task Description]
2. [Project Structure]
3. [Setup and Preparation]
4. [Implementation]
   1. [Saving the Blocked Countries List (AJAX → JSON)]
   2. [Loading the List and Initialization in `ipCheck.php` 
   3. [Exceptions (GET Parameter, User-Agent, Constant)]
   4. [Blocking and IP Logging]
   5. [PQA Tests for Block Verification]
5. [Development Recommendations]

---

## Task Description

Implement a mechanism to block users by country:

- **Save**/load a list of blocked countries/continents from the UI (JSON file).  
- **Exclude** from blocking:  
  - the GET parameter `skip_access_blocker=1`  
  - well-known User-Agents (Googlebot, SemrushBot, etc.)  
  - the `ROLEX_TYPE` constant  
- **Block** (HTTP 403) when the user’s country matches the blacklist.  
- **Log** the IP addresses of blocked users.  
- **Cover** the logic with PQA tests.

---

## Project Structure
```code
build-core/
├── assets/
│   └── functions/
│       └── VisitorInfo/
│           └── get_VisitorInfo.php        # getLocation()
├── controller/
│   └── Security/
│       └── ipCheck.php                   # handleRequest(), blockByCountry()
├── tests/
│   └── test_blocked_ips.php              # PQA tests
├── view/
│   └── dashboard/
│       └── pages/
│           └── website_access_blocker.php # UI for country selection
└── ws/
    └── ajax_blocked_countries.php        # AJAX endpoint to save list

public_html/
└── private/
    ├── 1.0/
    │   └── settings/
    │       └── blacklist/
    │           └── blocked_countries.json # stores JSON array of ISO codes
    └── blacklist/
        └── blocked_ips.log               # log of blocked IPs


```

## Setup and Preparation

1. **Create** the file `public_html/private/1.0/settings/blacklist/blocked_countries.json` with the following content:

```json
[]
```

2. **Ensure** the web server (PHP) has write permissions for:

* `blocked_countries.json`
* `blocked_ips.log`

---

## Implementation

### Saving the Blocked Countries List (AJAX → JSON)

**File:** `build-core/ws/ajax_blocked_countries.php`

```php
<?php
// ajax_blocked_countries.php
header('Content-Type: application/json');

$countries = $_POST['countries'] ?? [];
if (!is_array($countries)) {
    http_response_code(400);
    echo json_encode(['status' => 'error', 'message' => 'Invalid format']);
    exit;
}

// Validate ISO codes
$filtered = array_filter($countries, function($c) {
    return is_string($c) && preg_match('/^[A-Z]{2}$/', $c);
});

$path = $_SERVER['DOCUMENT_ROOT'] . '/private/1.0/settings/blacklist/blocked_countries.json';
if (file_put_contents($path, json_encode(array_values($filtered), JSON_PRETTY_PRINT)) === false) {
    http_response_code(500);
    echo json_encode(['status' => 'error', 'message' => 'Cannot write file']);
    exit;
}

echo json_encode(['status' => 'ok']);
```

---

### Loading the List and Initialization in `ipCheck.php`

**File:** `build-core/controller/Security/ipCheck.php`

```php
public function handleRequest() {
    // 1. Exceptions (see below)
    // …

    // 2. Load blocked countries list
    $jsonPath = $_SERVER['DOCUMENT_ROOT'] . '/private/1.0/settings/blacklist/blocked_countries.json';
    $blockedCountries = [];
    if (file_exists($jsonPath)) {
        $data = file_get_contents($jsonPath);
        $blockedCountries = json_decode($data, true) ?: [];
    }

    // 3. Determine user country
    if (!empty($_SESSION['geo-information'])) {
        $visitorLocation = $_SESSION['geo-information'];
    } else {
        require_once __DIR__ . '/../../assets/functions/VisitorInfo/get_VisitorInfo.php';
        $visitorLocation = getLocation();
        $_SESSION['geo-information'] = $visitorLocation;
    }
    $userCountry = $visitorLocation['country'] ?? null;

    // 4. Blocking and IP logging
    if (in_array($userCountry, $blockedCountries, true)) {
        // Log IP
        $logPath = $_SERVER['DOCUMENT_ROOT'] . '/private/blacklist/blocked_ips.log';
        file_put_contents($logPath, $this->ip . " ({$userCountry})\n", FILE_APPEND);
        // Block
        $this->blockByCountry($userCountry);
    }

    // Continue with the rest of the application logic
}
```

---

### Exceptions (GET Parameter, User-Agent, Constant)

Add at the beginning of `handleRequest()`:

```php
// Skip if GET parameter is present
if (!empty($_GET['skip_access_blocker'])) {
    return;
}

// Skip for known bots/services
$ua = $_SERVER['HTTP_USER_AGENT'] ?? '';
$allowedAgents = ['Googlebot', 'SemrushBot', 'AhrefsBot', 'PageSpeed', 'AdsBot', 'bingbot'];
foreach ($allowedAgents as $agent) {
    if (stripos($ua, $agent) !== false) {
        return;
    }
}

// Skip if ROLEX_TYPE constant is defined
if (defined('ROLEX_TYPE')) {
    return;
}
```

---

### Blocking and IP Logging

**Method:** `blockByCountry()` in `ipCheck.php`:

```php
public function blockByCountry(string $countryCode) {
    header('HTTP/1.1 403 Forbidden');
    exit;
}
```

IP logging is implemented in the loading section using `file_put_contents()`.

---

### PQA Tests for Block Verification

**File:** `build-core/tests/test_blocked_ips.php`

```php
<?php
// test_blocked_ips.php
require_once __DIR__ . '/../assets/functions/VisitorInfo/get_VisitorInfo.php';

$blocked = json_decode(
    file_get_contents(__DIR__ . '/../settings/blacklist/blocked_countries.json'),
    true
) ?: [];

$testIps = [
    '1.2.3.4', // example IP from a blocked country
    '8.8.8.8', // example IP from an allowed country
];

foreach ($testIps as $ip) {
    $country = getCountryByIp($ip);
    $result = in_array($country, $blocked, true) ? 'blocked' : 'allowed';
    echo "$ip ($country) — $result\n";
}
```

---

## Development Recommendations

* Cache the country list in memory with periodic updates (cron or on UI changes).
