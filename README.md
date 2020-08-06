# [Check if third-party cookies are enabled](https://stackoverflow.com/questions/3550790/check-if-third-party-cookies-are-enabled)

## Technical Background

The third party sets & reads cookies over HTTP (not in JavaScript).

So we need two requests to an external domain to test if third-party cookies are enabled:

+ One where the third party sets the cookie(s)
+ The second, with a differing response depending on whether the browser sent the cookie(s) back to the same third party in a second request.

We cannot use `XMLHTTPRequest` (Ajax) because of the DOM security model.

Obviously you can't load both scripts in parallel, or the second request may be made before the first request’s response makes it back, and the test cookie(s) will not have been set.

## Code Example

**Given:**

The `.html` file is on one domain, and

The `.js.php` files are on a second domain, we have:

### The HTML test page
Saved as `third-party-cookies.html`

```html
<!DOCTYPE html>
<html>
<head id="head">
  <meta charset=utf-8 />
  <title>Test if Third-Party Cookies are Enabled</title>
<style type="text/css">
body {
  color: black;
  background: white none;
}
.error {
  color: #c00;
}
.loading {
  color: #888;
}
.hidden {
  display: none;
}
</style>
<script type="text/javascript">
window._3rd_party_test_step1_loaded = function(){
  // At this point, a third-party domain has now attempted to set a cookie (if all went to plan!)
  var step2Url = 'http://third-party.example.com/step2.js.php',
    resultsEl = document.getElementById('3rd_party_cookie_test_results'),
    step2El = document.createElement('script');

  // Update loading / results message
  resultsEl.innerHTML = 'Stage one complete, loading stage 2&hellip;';
  // And load the second part of the test (reading the cookie)
  step2El.setAttribute('src', step2Url);
  resultsEl.appendChild(step2El);
}
window._3rd_party_test_step2_loaded = function(cookieSuccess){
  var resultsEl = document.getElementById('3rd_party_cookie_test_results'),
    errorEl = document.getElementById('3rd_party_cookie_test_error');
  // Show message
  resultsEl.innerHTML = (cookieSuccess ? 'Third party cookies are <b>functioning</b> in your browser.' : 'Third party cookies appear to be <b>disabled</b>.');

  // Done, so remove loading class
  resultsEl.className = resultsEl.className.replace(/\bloading\b/,' ');
  // And remove error message
  errorEl.className = 'hidden';
}
</script>
</head>
<body id="thebody">

  <h1>Test if Third-Party Cookies are Enabled</h1>

  <p id="3rd_party_cookie_test_results" class='loading'>Testing&hellip;</p>
  <p id="3rd_party_cookie_test_error" class="error hidden">(If this message persists, the test could not be completed; we could not reach the third-party to test, or another error occurred.)</p>

  <script type="text/javascript">
  window.setTimeout(function(){
    var errorEl = document.getElementById('3rd_party_cookie_test_error');
    if(errorEl.className.match(/\berror\b/)) {
      // Show error message
      errorEl.className = errorEl.className.replace(/\bhidden\b/,' ');
    } else {
    }
  }, 7*1000); // 7 sec timeout
  </script>
  <script type="text/javascript" src="http://third-party.example.com/step1.js.php"></script>
</body>
</html>
```

### The first third-party JavaScript file

Saved as `step1.js.php`

This is written in PHP so we can set cookies as the file loads. (It could, of course, be written in any language, or even done in server config files.)

```php
<?php
  header('Content-Type: application/javascript; charset=UTF-8');
  // Set test cookie
  setcookie('third_party_c_t', 'hey there!', time() + 3600*24*2);
?>
window._3rd_party_test_step1_loaded();
```

### The second third-party JavaScript file

Saved as `step2.js.php`

This is written in PHP so we can read cookies, server-side, before we respond. We also clear the cookie so the test can be repeated (if you want to mess around with browser settings and re-try).

```php
<?php
  header('Content-Type: application/javascript; charset=UTF-8');
  // Read test cookie, if there
  $cookie_received = (isset($_COOKIE['third_party_c_t']) && $_COOKIE['third_party_c_t'] == 'hey there!');
  // And clear it so the user can test it again 
  setcookie('third_party_c_t', '', time() - 3600*24);
?>
window._3rd_party_test_step2_loaded(<?php echo ($cookie_received ? 'true' : 'false'); ?>);
```

The last line uses the ternary operator to output a literal Javascript `true` or `false` depending on whether the test cookie was present.

[Test it here](https://alanhogan.github.io/web-experiments/3rd/third-party-cookies.html)



## Also

Here's a pure JS solution not requiring any server-side code, so it can work from a static CDN: https://github.com/mindmup/3rdpartycookiecheck - the first script sets the cookie in the code, then redirects to a second script that will post a message to the parent window.

You can try out a live version using https://jsfiddle.net/tugawg8y/

client-side `HTML`:

```html
third party cookies are <span id="result"/>
<iframe src="https://mindmup.github.io/3rdpartycookiecheck/start.html"
    style="display:none" />
```

client-side `JS`

```javascript
var receiveMessage = function (evt) {
   if (evt.data === 'MM:3PCunsupported') {
     document.getElementById('result').innerHTML = 'not supported';
   } else if (evt.data === 'MM:3PCsupported') {
     document.getElementById('result').innerHTML = 'supported';
   }
 };
 window.addEventListener("message", receiveMessage, false);
```

Of course, this requires that the client runs JavaScript which is a downside compared to the server-based solutions; on the other hand, it's simpler, and you were asking about a JS solution.