# Cookie Change Events #
Cookie Change Events is a minor extension to `document.cookie` in order to address one of the largest performance issues in using cookies in production today, without requiring developers to completely re-engineer their existing cookie frontend architecture.

## The Problem ##
Users frequently browse the same website in multiple tabs or windows. While user agents automatically sync any cookie changes between those tabs and windows, there is no built-in way for developers to be alerted of cookie changes in other tabs. This is essential in order to ensure that the state of an application (e.g. authentication status, logged-in status, etc.) is accurate within the application itself. To illustrate:

- Imagine Alice opens the social network `example.com` in window A, minimizes it, and then later logs out of window B.
- Bob comes to ues the same computer, opens another window, and signs into his `example.com`. He then closes the window without logging out.
- Alice then returns, and maximizes window A. Since it hasn't been interacted with, it still shows Alice's account and friends. However, when she posts a status, it then gets shared to Bob's wall. Window A inaccurately reflects the state of the cookies.

In order to get around this issue, most websites that require it poll `document.cookie` over and over again, doing untold number of string comparisons.

This is very silly.

We propose a minor IDL change to `document.cookie`, simply adding an `addEventListener` to it. This means it can be a near drop-in replacement for existing code.

So the polling technique:

```js
const current_cookie = document.cookie
const changed = () => {
  location.href="/home"
}

setTimeout(() => {
  if (current_cookie !== document.cookie) {
    changed()
  }
}, 500)
```

can be quickly rewritten as:

```js
const changed = () => {
  location.href="/home"
}

document.cookie.addEventListener('change', changed)
```

## Compat concerns ##
While the existing cookie interface has _substantial_ shortcomings, the fact that it ships everywhere already means that any replacement for it will take years for all browsers to ship, and all current browsers to be no longer considered supported by developers in order for it to be considered a palatable alternative. Rather than hope for developers to write the same logic twice over in largely different abstractions, we attempt to maintain full backwards compatibility. As a result, all existing frontend cookie libraries should continue to function without any changes.

Simply put, this means that

```js
() = {
  document.cookie="foo=bar"
  document.cookie="bar=baz"
  return document.cookie
}

```

will still return `foo=bar; bar=baz`. In addition, even `typeof document.cookie` will still be a `string` type. 

# Improvements # 

## CookieChangeEvent ##

After a developer has established a cookie change event listener, they have a new event type to deal with:

```js
document.cookie.addEventListener('change', (e) => {
  typeof e === 'CookieChangeEvent'
})
```

The `CookieChangeEvent` has several interesting properties: `cause` and `cookie`. 

[`cause`](https://patrickkettner.github.io/cookie-change-events/#enumdef-changecause) will alert the developer to the way a given cookie has been modified (e.g. it was created, expired, was explicitly deleted, etc.). It is worth noting that this information is not something that is easily detectable with today's `document.cookie`.


## Cookie object ##

In addition to `cause`, the event introduces `cookie` as an object representation of a cookie. Historically, cookies are magic strings that can contain a number of optional fields, e.g. `;expires=GMT-date-string`, `;max-age=time-in-seconds`, `;secure` to name a few.

This is very silly. 

The `cookie` field formats this into a standard JS object:

```json
  {
    "name": "foo",
    "expires": "Thu, 09 Nov 2030 12:00:00 GMT"
  }
```

(All fields covered are [available in the specification itself](https://patrickkettner.github.io/cookie-change-events/#cookie).)

Additionally, all cookies have a stringifier, so that a developer can quickly get a backwards compatible version of a cookie string:

```js
document.cookie.addEventListener('change', (e) => {
  const cookie = e.cookie
  cookie.name === 'foo' // true
  cookie.expires === 'Thu, 09 Nov 2030 12:00:00 GMT'

  cookie.toString() === 'foo=bar; expires="Thu, 09 Nov 2030 12:00:00 GMT"'
})
```

Finally, a `cookie` can be used as an alternative to the traditional string interface to set a cookie in a more modern way:

```js
let cookie = new Cookie({
  name: 'foo',
  value: 'bar', 
  expires: 'Thu, 09 Nov 2030 12:00:00 GMT'
})

document.cookie = cookie

console.log(document.cookie)
  // "foo=bar"
```

And of course, you can intermingle the old and new styles intermingling at will:

```js
let cookie = new Cookie({
  name: 'foo',
  value: 'bar', 
  expires: 'Thu, 09 Nov 2030 12:00:00 GMT'
})

document.cookie = cookie

console.log(document.cookie)
  // "foo=bar"

document.cookie = 'bar=baz'

console.log(document.cookie)
  // "foo=bar; bar=baz"

document.cookie = 'foo=baz'

console.log(document.cookie)
  // "foo=baz; bar=baz"
```

## Iterable support ##

A pleasant side effect of exposing individual cookies as objects within `document.cookie` is it becomes trivial to iterate over them. By doing so, `document.cookie` inherits all of the behaviors of [maplike structures](https://heycam.github.io/webidl/#idl-maplike) - e.g. `.set`, `.get`, and `.delete`. This allows code that is much less confusing that the existing purely string based interaction (keeping in mind that the old version still works).

```js

document.cookie.set('foo', 'bar')

document.cookie.set('bar', {
  value: 'baz',
  expires: 'Thu, 09 Nov 2030 12:00:00 GMT'
})

for (let cookie of document.cookie) {
  if (cookie.name === 'foo') {
    document.cookie.delete(cookie)
  }
}

const cookie = document.cookie.get('bar') 

console.log(cookie)
  /* 
    {
      name: 'bar',
      value: 'baz',
      expires: 'Thu, 09 Nov 2030 12:00:00 GMT'
    }
  */

console.log(cookie.toString())

// 'bar=baz; expires="Thu, 09 Nov 2030 12:00:00 GMT"'
```