# cookie change events
[Draft specification](https://patrickkettner.github.io/cookie-change-events/)

Web applications have no way of knowing when a cookie has been modified if it is
opened in multiple tabs or windows. As a result web apps have to poll document.cookie
in order to determine if another window has modified the cookie.

## Use-cases
  - Get a notification to allow a developer to Automatically log out of a web app
  in one window when another window on the same domain logs out.

## Example

```javascript
document.cookie.addEventListener('change', event => {
  if (event.removed) {
    console.log(`
      The cookie ${JSON.stringify(event.cookie)} has been removed.
    `)

    switch (event.cause) {
      case 'evicted':
        // the cookie has been removed because of garbage collection
        break
      case 'expired':
        // the cookie has automatically been removed because of it's expiration info
        break
      case 'expired_overwrite':
        // the cookie has been removed because of it was overwritten with a date in the past
        break
      case 'explicit':
        // the cookie has manually change via javascript
        break
    }
  }
  else if (event.cause === 'overwrite') {
    console.log(`
      The cookie ${JSON.stringify(event.cookie)} has been overwritten with a new value
    `)
  }

})
```
