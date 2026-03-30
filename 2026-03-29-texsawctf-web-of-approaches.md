# Web of Approaches Writeup

Challenge:
- `Web of Approaches`
- Author: `Bluedetroyer`
- Flag format: `texsaw{flag}`

Recovered flag:

```text
texsaw{tH3rE_4r3_M4nY_W4Ys_t0_s0lV3_4_cH4l1nG3}
```

## Overview

This challenge hid flag fragments across three web-layer paths:

1. `structure` via a hidden HTML span
2. `style` via the background PNG
3. `script` via a semantic backend response

The recurring trick was:

- render gibberish through a custom font
- interpret the rendered text as Base64-like chunks
- read the visible clue text
- extract the real flag data from the low nibble of the last meaningful Base64 character before `==`

The custom font made raw source look unreadable while the browser showed normal text.

## Step 1: Render The Page Correctly

Raw HTML looked like gibberish, but once rendered with the site font the page became an ordinary HTML quiz.

<img width="1440" height="1800" alt="page" src="https://github.com/user-attachments/assets/f8d3cb4c-d538-4566-b4ce-bf0b88967e95" />

Useful font-remapped identifiers:

- `istsgt` -> `detect`
- `gbsgTh9Xms3X` -> `checkAnswers`
- `D4kG-XsG7s9t` -> `flag-segment`
- `E2sXtKV9` -> `question`
- `X2o7Kt` -> `submit`
- `3sXRV9Xs` -> `response`

## Step 2: Extract The HTML Fragment

The page contained a hidden span:

```html
<span class="bKiis9" id="D4kG-XsG7s9t">...</span>
```

`script.js` shifted each character by:

```js
((i * i + 3) % 5)
```

After applying that shift and then rendering through the custom font, the span became Base64-like chunks such as:

```text
T25lIH==
b2Ygc0==
ZXZlck==
...
```

The visible message decodes approximately to:

```text
One of several parts of the flag is contained somewhere inside of this string.
```

The text is slightly corrupted because the challenge intentionally abuses non-canonical Base64 padding.

The actual fragment is stored in the low nibble of the sixth Base64 character in each chunk. Packing those nibbles into bytes yields:

```text
tH3rE_4r3_M4n
```

## Step 3: Extract The CSS Fragment

The stylesheet uses a tiny repeating background image:

```css
body {
    background-image: url('./D4kG_XsG7s9t.png');
}
```

That PNG stores ASCII in the alpha channel. When those rows are rendered through the custom font, they become another Base64-like stream.

<img width="1600" height="2600" alt="bg_rows_large" src="https://github.com/user-attachments/assets/176efa2e-b52d-4eee-a02c-6583a947a05c" />


The visible message decodes approximately to:

```text
There are three parts of a web page, the structure, the style, and the script.
```

Applying the same hidden-nibble extraction gives:

```text
V3_4_cH4l1nG3
```

## Step 4: Understand The Trap

The quiz frontend and the devtools-detection script are mostly a diversion.

`istsgt.js` really does detect an inspected/debugged session, but its callback payload only reports detector metadata such as:

- `LogPerformance`
- `ObjectID`
- `defineGetter`
- timing values

That path did not contain the third flag fragment.

The visible POST response for ordinary quiz submissions also looked like a dead end:

<img width="1400" height="500" alt="resp" src="https://github.com/user-attachments/assets/4fa8ef77-b1e4-4e5b-9425-5c16aa45150b" />


Decoded, it says roughly:

```text
Incorrect answers. Please ensure all existing questions have a correct answer.
```

Brute-forcing all `3^5 = 243` visible answer combinations still returned the same response.

## Step 5: Send Semantic Answers Instead Of Obfuscated Values

The key pivot was noticing that the quiz options decode cleanly into real HTML answers:

- `XRk9` -> `span`
- `V4` -> `ol`
- `9VXg3KRt` -> `noscript`
- `s7` -> `em`
- `o3` -> `br`

But posting those raw obfuscated values keeps the backend in the default response path.

Posting semantic field names and semantic values changes the behavior. For example:

```json
{
  "question-1": "span",
  "question-2": "ol",
  "question-3": "noscript",
  "question-4": "em",
  "question-5": "br"
}
```

That returns a different rendered response:

```text
correct answers! Part of the flag is: ...
```

The returned chunk stream decodes approximately to:

```text
To find all the parts you will need to search where you normally might overlook!
```

Again, the visible message is slightly corrupted because the same padding-based covert channel is being used.

Extracting the hidden nibbles from that response yields the third fragment:

```text
Y_W4Ys_t0_s0l
```

## Step 6: Assemble The Flag

Recovered fragments:

```text
html: tH3rE_4r3_M4n
js  : Y_W4Ys_t0_s0l
css : V3_4_cH4l1nG3
```

Combined:

```text
texsaw{tH3rE_4r3_M4nY_W4Ys_t0_s0lV3_4_cH4l1nG3}
```

## Notes

- The visible clue text in all three paths is not the authoritative data source.
- The flag fragments are hidden in malformed Base64 padding, not in the visible decoded English.
- The devtools detector is real, but it is not the missing third carrier.
