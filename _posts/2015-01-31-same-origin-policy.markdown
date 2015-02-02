---
layout: post
title:  "Same-Origin Policy"
date:   2015-01-31 18:07:02
categories: jekyll update
---
This post is exploring Same-Origin Policy (SOP) in modern web browsers by making assumptions based on the documentation (https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy), performing experiments to test those assumptions and adjusting understanding of SOP.

SOP was introduced in Netscape Navigator some 20 years ago in order to protect sensitive webcontent from malicious scripts. It uses the term Origin that is a combination of scheme (http, https, file etc.), domain and port. And the idea is that script can access web content's DOM only from the same origin and can manipulate it. It's applied to accessibility of iframes, child windows and out-of-browser communications as well like XHR requests, the last one is tricky and I'd like to explore it in a separate post.

There are several chapters in this post:

* I. Access DOM from a foreign code included using `<script>` element

* II. Access DOM of a child frame (iframe and new window)

* III. `postMessaging` in web browsers (iframe and new window)

* IV. Exploring limits of `postMessaging` in web browsers (iframe only)

* V. Conclusions

I. Access DOM from a foreign script
-----------------------------------
The experiment was to try to access DOM from the script that's hosted from different origin. We have two files here: main page and script that is included in the main page.

main page (that is hosted on `http://host1.dev:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Hello, world!</h1>
		<script src="http://host2.dev:8081/service.js"></script>
	</body>
</html>
{% endhighlight %}
included script (that is hosted on `http://host2.dev:8081` which is a different origin):
{% highlight javascript %}
console.log('service.js started');
var element = document.querySelector('h1');
console.log(element);
{% endhighlight %}

And the output to browser's console is:
{% highlight html %}
service.js started
<h1>Hello, world!</h1>
{% endhighlight %}

The output implies that the script has access to the DOM. I was confused by this because the SOP specification says that script from different origin can not access the DOM, but it's definitely opposite in our example.

As it turned out Same-Origin Policy is not applied to `<script>` as well as `<img>` and others because that's responsibility of the website owner to be careful about what content is included into the webpage or at least that's W3C's philosophy. So let's explore in what cases SOP is applied.

II. Access DOM of a child frame
-------------------------------
In this experiment we'll create an iframe and new window with a content from the same origin and from the different origin and we'll see how browser restricts access in this case. We have two files that contain main page content and iframe's content.

## II.a. iframe
The iframe's content needs to be loaded and it takes some, that's why it should be some delay before scanning iframe's content.

# same origin
main page (that is hosted on `http://localhost:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<iframe src="iframe.html"></iframe>
		<script>
			setTimeout(function () {
				var iframe = document.querySelector('iframe');
				console.log('[main] ', iframe);
				console.log('[main] ', iframe.contentDocument.querySelector('h2'));
			}, 1000);
		</script>
	</body>
</html>
{% endhighlight %}
embedded iframe (that is hosted on `http://localhost:8080` as well):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is an embedded iframe</h2>
		<script>
			console.log('[iframe] ', parent.document.querySelector('h1'));
		</script>
	</body>
</html>
{% endhighlight %}
output to console:
{% highlight html %}
[iframe] <h1>Main page</h1>
[main] <iframe src="iframe.html"></iframe>
[main] <h2>This is an embedded iframe</h2>
{% endhighlight %}
The output shows that main page and embedded iframe can access each other's DOM (document object) since they both are from the same origin `http://localhost:8080`

# different origins
main page (that is hosted on `http://host1.dev:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<iframe src="http://host2.dev:8081/iframe.html"></iframe>
		<script>
			setTimeout(function () {
				var iframe = document.querySelector('iframe');
				console.log('[main] ', iframe);
				console.log('[main] ', iframe.contentDocument.querySelector('h2'));
			}, 1000);
		</script>
	</body>
</html>
{% endhighlight %}
embedded iframe (that is hosted on `http://host2.dev:8081` which is a different origin):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is an embedded iframe</h2>
		<script>
			console.log('[iframe] ', parent.document.querySelector('h1'));
		</script>
	</body>
</html>
{% endhighlight %}
output to console:
{% highlight html %}
Uncaught SecurityError: Blocked a frame with origin "http://host2.dev:8081" from accessing a frame with origin "http://host1.dev:8080". Protocols, domains, and ports must match.
[main] <iframe src="http://host2.dev:8081/iframe.html"></iframe>
Uncaught SecurityError: Failed to read the 'contentDocument' property from 'HTMLIFrameElement': Blocked a frame with origin "http://host1.dev:8080" from accessing a frame with origin "http://host2.dev:8081". Protocols, domains, and ports must match.
{% endhighlight %}
The output shows that main page and embedded iframe can NOT access each other's DOM since they are from different origins `http://host1.dev:8080` and `http://host2.dev:8081`

## II.b. new window
Web browsers don't allow webpages to open new windows without a direct user interaction(http://stackoverflow.com/questions/11821009/javascript-window-open-not-working). Thus we have to create a button and open new window in response to user interaction and don't forget about a delay that allows content to be fully loaded.

# same origin
main page (that is hosted on `http://localhost:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<button onclick="start()">Start</button>
		<script>
			function start() {
				var childWin = window.open('newwindow.html');

				setTimeout(function () {
					console.log('[main] ', childWin);
					console.log('[main] ', childWin.document.querySelector('h2'));
				}, 1000);
			}
		</script>
	</body>
</html>
{% endhighlight %}
new window (that is hosted on `http://localhost:8080` as well):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is a new window</h2>
		<script>
			console.log('[newwin] ', window.opener.document.querySelector('h1'));
		</script>
	</body>
</html>
{% endhighlight %}
output to main page's console:
{% highlight html %}
[main] Window {top: Window, window: Window, location: Location, external: Object, chrome: Object…}
[main] <h2>This is a new window</h2>
{% endhighlight %}
output to new window's console:
{% highlight html %}
[newwin] <h1>Main page</h1>
{% endhighlight %}
The output shows that main page and new window can access each other's DOM since they both are from the same origin `http://localhost:8080` despite of separate tabs in a web browser. This is a discovery for me, I viewed separate tabs as independent before.

# different origins
main page (that is hosted on `http://host1.dev:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<button onclick="start()">Start</button>
		<script>
			function start() {
				var childWin = window.open('http://host2.dev:8081/newwindow.html');

				setTimeout(function () {
					console.log('[main] ', childWin);
					console.log('[main] ', childWin.document.querySelector('h2'));
				}, 1000);
			}
		</script>
	</body>
</html>
{% endhighlight %}
new window (that is hosted on `http://host2.dev:8081` which is a different origin):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is a new window</h2>
		<script>
			console.log('[newwin] ', window.opener.document.querySelector('h1'));
		</script>
	</body>
</html>
{% endhighlight %}
output to main page's console:
{% highlight html %}
[main] Window {}
Uncaught SecurityError: Blocked a frame with origin "http://host1.dev:8080" from accessing a frame with origin "http://host2.dev:8081". Protocols, domains, and ports must match.
{% endhighlight %}
output to new window's console:
{% highlight html %}
Uncaught SecurityError: Blocked a frame with origin "http://host2.dev:8081" from accessing a frame with origin "http://host1.dev:8080". Protocols, domains, and ports must match.
{% endhighlight %}
The output shows that main page and new window can NOT access each other's DOM since they are from different origins `http://host1.dev:8080` and `http://host2.dev:8081`.

III. postMessaging in web browsers
----------------------------------
Here we'll explore how to overcome limitations set by SOP, that's important in order to embed a piece of functionality and communicate with it from a provider like Comments widget from Disqus or web widgets (Register, Login, Comments and others) that are provided by Vixlet (http://developers.vixlet.com/widgets). Client website's origin and web widget provider's origin are different but they have to exchange messages with each other like SDK calls and callbacks.
Web browsers have developed approach to this problem: `postMessage`. DOM is still unavailable for a frame with a different origin but frames can communicate any data using `window.postMessage` and we'll explore this method in the third experiment testing only webpages from different origins.

## III.a. iframe
The iframe's content needs to be loaded and it takes some, that's why it should be some delay before communication with the iframe.

main page (that is hosted on `http://host1.dev:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<iframe src="http://host2.dev:8081/iframe.html"></iframe>
		<script>
			var startTime, endTime;

			window.addEventListener('message', function (event) {
				console.log('[main] message received: ', event);
				endTime = new Date().getTime();
				console.log('[main] delta is ', endTime - startTime, ' milliseconds');
			});

			setTimeout(function () {
				var iframe = document.querySelector('iframe');
				startTime = new Date().getTime();
				iframe.contentWindow.postMessage('test message from parent', '*');
				console.log('[main] message has been sent');
			}, 1000);
		</script>
	</body>
</html>
{% endhighlight %}
iframe embedded page (that is hosted on `http://host2.dev:8081` which is a different origin):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is embedded iframe</h2>
		<script>
			window.addEventListener('message', function (event) {
				console.log('[iframe] message received: ', event);
				parent.window.postMessage('test response from iframe', '*');
				console.log('[iframe] response has been sent back');
			});
		</script>
	</body>
</html>
{% endhighlight %}
output to console:
{% highlight html %}
[main] message has been sent
[iframe] message received:  MessageEvent {ports: Array[0], data: "test message from parent", source: Window, lastEventId: "", origin: "http://host1.dev:8080"…}
[iframe] response has been sent back
[main] message received:  MessageEvent {ports: Array[0], data: "test response from iframe", source: Window, lastEventId: "", origin: "http://host2.dev:8081"…}
[main] delta is  17  milliseconds
{% endhighlight %}
The output shows that main page and embedded iframe can communicate with each other despite of different origins but they can NOT access each other's DOM as the previous experiment showed. And the communication speed is pretty fast: delta between sending a `postMessage` to iframe and receiving response is 17ms for Google Chrome 40 and 6ms for MSIE 10 WTF! The performance will be explored in the next chapter.

`postMessaging` with iframes is supported by all the major browsers (http://caniuse.com/#search=postmessage). Our next goal is to test it with a new window.

## III.a. new window
Web browsers don't allow to open new window without a preliminary user interaction, so we need a button that user has to click in order to trigger the test. Moreover new window's content needs to be loaded and it takes some, that's why it should be some delay before communication with new window.

main page (that is hosted on `http://host1.dev:8080`):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<button onclick="start()">Start</button>
		<script>
			function start() {
				var startTime, endTime;
				var childWin = window.open('http://host2.dev:8081/newwindow.html');

				window.addEventListener('message', function (event) {
					console.log('[main] message received: ', event);
					endTime = new Date().getTime();
					console.log('[main] delta is ', endTime - startTime, ' milliseconds');
				});

				setTimeout(function () {
					startTime = new Date().getTime();
					childWin.postMessage('test message from parent', '*');
					console.log('[main] message has been sent');
				}, 1000);
			}
		</script>
	</body>
</html>
{% endhighlight %}
new window (that is hosted on `http://host2.dev:8081` which is a different origin):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is a new window</h2>
		<script>
			window.addEventListener('message', function (event) {
				console.log('[newwin] message received: ', event);
				window.opener.postMessage('test response from a new window', '*');
				console.log('[newwin] response has been sent back');
			});
		</script>
	</body>
</html>
{% endhighlight %}
output to main page's console:
{% highlight html %}
[main] message has been sent
[main] message received:  MessageEvent {ports: Array[0], data: "test response from a new window", source: Window, lastEventId: "", origin: "http://host2.dev:8081"…}
[main] delta is  12  milliseconds
{% endhighlight %}
output to new window's console:
{% highlight html %}
[newwin] message received:  MessageEvent {ports: Array[0], data: "test message from parent", source: Window, lastEventId: "", origin: "http://host1.dev:8080"…}
[newwin] response has been sent back
{% endhighlight %}
The output shows that main page and new window can communicate with each other despite of separated browser tabs and different origins but they can NOT access each other's DOM as the previous experiment showed. And the communication speed is still pretty fast: delta between sending a `postMessage` to iframe and receiving response is 12ms for Google Chrome 40 and 5ms for MSIE 10. `postMessaging` works fine even between separated windows and different origins. The performance will be explored in the next chapter.

IV. Exploring limits of postMessaging in web browsers
----------------------------------------------------
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h1>Main page</h1>
		<iframe src="http://host2.dev:8081/iframe.html"></iframe>
		<script>
			var startTime, endTime;
			var dataSize = 100; // data has dataSize*dataSize integer elements
			var data = [];

			for (var i = 0; i < dataSize; i++) {
				data[i] = [];
				for (var j = 1; j < dataSize; j++)
					data[i][j] = (i + j) * (i + j - 11);
			}

			window.addEventListener('message', function (event) {
				console.log('[main] message received: ', event);
				endTime = new Date().getTime();
				console.log('[main] delta is ', endTime - startTime, ' milliseconds');
			});

			setTimeout(function () {
				var iframe = document.querySelector('iframe');
				startTime = new Date().getTime();
				iframe.contentWindow.postMessage(data, '*');
				console.log('[main] message has been sent');
			}, 1000);
		</script>
	</body>
</html>
{% endhighlight %}
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is embedded iframe</h2>
		<script>
			window.addEventListener('message', function (event) {
				console.log('[iframe] message received: ', event);
				parent.window.postMessage(event.data, '*');
				console.log('[iframe] echo has been sent back');
			});
		</script>
	</body>
</html>
{% endhighlight %}

V. Conclusions
---------------
SOP protects content of a wepsite from different frames by restricting access only to scripts from the same origin as the website and simultaneously it prevents foreign scripts to participate in user interaction, it makes things safer but more compicated to implement. Browsers came up with an idea to keep security on a satisfactory level but allow frames to communicate with each other: `postMessaging`. This mechanism is well-implemented, easy to use and is supported by all the major browsers including IE10, ready for production.