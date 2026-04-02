# WebSec – ChaChaCha writeup

This challenge looks like a “find a SHA-1 collision” problem, but the real bug is a bad interaction between `sha1()` and `password_hash()`.

Relevant code:

```php
$h2 = password_hash(sha1($_POST['c'], fa1se), PASSWORD_BCRYPT);

if (password_verify(sha1($flag, fa1se), $h2) === true) {
    echo $flag;
} else {
    echo sha1($flag, false);
}
```

## Step 1: Understand the typo

The second parameter of `sha1()` is supposed to be a boolean:

- `false` → return 40-character hex string
- `true` → return raw 20-byte binary digest

The code uses `fa1se`, not `false`.

In the challenge environment, that ends up being truthy, so the calls behave like:

```php
sha1($_POST['c'], true)
sha1($flag, true)
```

So the input to bcrypt is not hex SHA-1 text. It is **raw binary SHA-1 output**.

## Step 2: Use the leaked SHA-1

When verification fails, the page leaks:

```php
sha1($flag, false)
```

which is:

```text
7c00249d409a91ab84e3f421c193520d9fb3674b
```

Split into bytes:

```text
7c 00 24 9d 40 9a 91 ab 84 e3 f4 21 c1 93 52 0d 9f b3 67 4b
```

The key observation is that the **second byte is `00`**.

## Step 3: Why this matters

bcrypt does not safely handle arbitrary raw binary input like this. An early NUL byte (`\x00`) causes the comparison to effectively stop early.

So instead of needing the real flag, we only need an input whose raw SHA-1 begins with:

```text
7c 00
```

That is much easier than finding a full SHA-1 collision.

## Step 4: Brute-force a matching prefix

A simple brute-force script finds a short string whose SHA-1 starts with `7c00`:

```python
import hashlib
import itertools
import string

alphabet = string.ascii_letters + string.digits

for n in range(1, 8):
    for tup in itertools.product(alphabet, repeat=n):
        s = "".join(tup)
        if hashlib.sha1(s.encode()).hexdigest().startswith("7c00"):
            print(s)
            raise SystemExit
```

This gives:

```text
CWM
```

## Step 5: Submit the value

Submitting `CWM` as `c` makes:

- `sha1("CWM", true)` start with `7c 00`
- `sha1($flag, true)` also start with `7c 00`

Because of the early NUL byte, `password_verify()` accepts it and the page prints the flag.

## Flag

```text
WEBSEC{Please_Do_not_combine_rAw_hash_functions_mi}
```

## Takeaway

The challenge is a nice example of why you should not:

- pre-hash input with raw binary output
- then feed that into bcrypt
- especially when PHP type/typo behavior changes how the hash function is called

This is not a real SHA-1 collision challenge. It is a **raw-hash + bcrypt truncation bug**.
