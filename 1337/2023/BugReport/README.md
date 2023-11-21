# Bug Report Repo

> I started my own bug bounty platform! The UI is in the early stages but we've already got plenty of submissions. I wonder why I keep getting emails about a "critical" vulnerability report though, I don't see it anywhere on the system

Sources:
* https://pastebin.com/EuNqbHc7 (downloaded client js)


Opening the website, we find a bug tracker website. In this site, we can type in a number into the input and check the report status by bug id (see below).

![normal operation](https://i.imgur.com/Drol0Gj.png)

## Stage 1: SQLi

### Research

The flavortext hints at a critical vulnerability report that we do not see. I tried changing the id outside of the range `1..10` and found that there is a report with `id=11` from `ethical_hacker`.

![ethical_hacker bug](https://i.imgur.com/RHxbHxI.png)

I then tried putting in various SQL Injection payloads like `' OR 1=1 --` only to get a response indicating that there is an error. I also tried putting in normal letters like `a` and `b` and got the same error response.

![error message](https://i.imgur.com/vnlIxw1.png)

Eventually, I put in `'a'` and `'b'` and got the message `bug not found`.

![bug not found](https://i.imgur.com/J6NXvDH.png)

Since quoting `a` resulted in the query succeeding but not quoting `a` resulted in an error, I assumed that the id field is not quoted at all in the query. In SQL, columns can be compared against integers without quotes, but strings do need quotes. This is why we needed to quote `'a'`.

Now that we know that we have unrestricted injection, we can leak the description field of bug 11 using the `LIKE` SQL operator. `LIKE` allows us to compare a column to a wildcard string value. We can find out if the description starts with abc by seeing if `description LIKE 'abc%'` (`%` means wildcard). Using this, we can test out many different prefixes and build up the description value 1 character at a time.

Let us test out this mechanism on bug id 1 with below injection:
```sql
'1' AND description LIKE 'I%'
```
![bug id 1 test](https://i.imgur.com/yshHdVL.png)

Wow, it works.

### Implementation

Looking at the [page source](https://pastebin.com/EuNqbHc7), it seems like the site uses a websocket for server communications. It sends a request every time we type into the input bar and updates the `status` div when the server responds.

I have uploaded [my very heavily commented solve script](https://gist.github.com/r2dev2/347a2692f094921e4c3c83df5b0ad759). I ran the script in the console in the same window as the site. As I started this challenge around 50 minutes before the CTF ended (was doing SquareCTF which was during same time as this CTF), I had to take a few shortcuts.

Instead of properly using callbacks and promises, I just made a `wait_until` async function that polls for a condition and resolves when it succeeds. This helped me quickly implement Promise-based websocket requests at the cost of efficiency.

I also made a websocket connection to test each possible alphabet character in parallel. My original serial solution was taking too long to run so I had to parallelize it and it was very easy to parallelize across every possible character.

The mix of `var` and `let/const` may be jarring for some readers but it was done so that I could re-run parts of my payload in the same console.

### Results

The description field was leaked fairly quickly.

![leaked field](https://i.imgur.com/xSjNmmw.png)

This leads us to the next stage of the challenge: `/4dm1n_z0n3` endpoint.

## Stage 2: Cookie Cracking

Going to `/4dm1n_z0n3`, we are met with a login form

![login form](https://i.imgur.com/UPuzyry.png)

I logged in with the credentials from the leaked description and was met with a restriction that I need admin permission to view the secret key.

![need admin](https://i.imgur.com/E4vExQW.png)

In order to log in as admin, we would have to either guess their password, modify our cookie, or sqli the login page. Looking at the cookie, it seems to be a jwt. Let us first explore maliciously modifying the jwt as it is much easier than the other possible attacks.

![jwt viewer](https://i.imgur.com/ntVluUw.png)

Opening the jwt in [https://jwt.io](https://jwt.io), we can dissect this jwt. It appears to be a HS256-signed jwt and has the username of the logged in user. What this means is that the cookie says that `crypt0` is logged in and to prevent tampering, more or less `crypt0 + <secret>` is hashed as a signature. The server can then check that `hash(<cookie> + <secret>) == <signature>` to verify that the cookie is not tampered with.

There are many possible jwt attacks and the two that came to my mind are:

1. edit the authorization type field to have `"alg": "none"` so that the signature will not be checked (although most sane jwt implementations will guard against this by default)
2. try cracking the signing secret from a wordlist, [the rockyou leak wordlist](https://en.wikipedia.org/wiki/RockYou#Data_breach) is a very solid list to check against

I decided to first start off a hashcat to try to crack the jwt and then concurrently figure out how to tamper with auth field for attack 1.

I saved the jwt to `jwt.txt` and ran below command

```
$: hashcat -m 16500 jwt.txt ~/Downloads/rockyou.txt
<omitted>
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZGVudGl0eSI6ImNyeXB0MCJ9.zbwLInZCdG8Le5iH1fb5GHB5OM4bYOm8d5gZ2AbEu_I:catsarethebest
<omitted>
```

It turns out that the jwt was signed with the secret `catsarethebest`. Now that we know how the jwt is signed, we can use the online editor to tamper with the jwt.

![tampering](https://i.imgur.com/rNJrFWY.png)

Setting our session to the tampered cookie, we are able to log into the admin

![admin log in](https://i.imgur.com/2Py0LAu.png)