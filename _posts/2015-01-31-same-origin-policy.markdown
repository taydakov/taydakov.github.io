---
layout: post
title:  "Same-Origin Policy"
date:   2015-01-31 18:07:02
categories: jekyll update
---
The point of this post is to explore Same-Origin Policy (SOP) in modern web browsers by making assumptions based on the documentation (https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) and performing experiments. The feature was introduced in Netscape Navigator in order to protect sensitive webcontent from malicious scripts. And the idea is that script can access web content's DOM only from the same origin (origin is scheme + domain + port) and manipulate it. It's applied to iframe, child windows and out-of-browser communications like XHR requests (the last is tricky).

I. First Experiment
----------------
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

The output implies that the script has access to the DOM. I was confused by this because the SOP specification says that script from different origin can not access the DOM, but it definitely can in our example.

Actually Same-Origin Policy is not applied to `<script>` tag as well as `<img>` and others because that's responsibility of the website owner to be careful about what content is included into the webpage or at least that's W3C's philosophy. So let's explore in what cases SOP is applied

II. Second Experiment
-----------------
In this experiment we'll create an iframe and new window with a content from the same origin and from the different origin and we'll see how browser restricts access in this case. We have two files that contain main page content and iframe's content.

## II.a iframe
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
iframe embedded page (that is hosted on `http://localhost:8080` as well):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is embeddable iframe</h2>
		<script>
			console.log('[embed] ', parent.document.querySelector('h1'));
		</script>
	</body>
</html>
{% endhighlight %}
output to console:
{% highlight html %}
[embed] <h1>Main page</h1>
[main] <iframe src="iframe.html"></iframe>
[main] <h2>This is embeddable iframe</h2>
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
iframe embedded page (that is hosted on `http://host2.dev:8081` which is a different origin):
{% highlight html %}
<html>
	<head>
	</head>
	<body>
		<h2>This is embeddable iframe</h2>
		<script>
			console.log('[embed] ', parent.document.querySelector('h1'));
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

## II.b new window
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

III. Third experiment
Here we'll explore how to overcome limitations set by SOP, that's important in order to embed a piece of functionality and communicate with it from a provider like Comments widget from Disqus or web widgets (Register, Login, Comments and others) that are provided by Vixlet (http://developers.vixlet.com/widgets). Client website's origin and web widget provider's origin are different but they want to exchange messages with each other like SDK calls and callbacks.
Web browsers have developed approach to this problem: `postMessage`. DOM is still unavailable for a frame with a different origin but frames can communicate any data using `window.postMessage`.