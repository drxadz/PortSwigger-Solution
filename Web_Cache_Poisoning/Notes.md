

## What is web cache poisoning?

- Attacker exploits the behavior of a web server and cache so that a harmful HTTP response is served to other users;
- Attack made by two phases
	- First, the attacker must work out how to elicit a response from the back-end server that inadvertently contains some kind of dangerous payload;
	- Second, once successful the first step, they need to make sure that their response is cached and subsequently served to the intended victims;
-  The impact cold be, such as [XSS](https://portswigger.net/web-security/cross-site-scripting), JavaScript injection, open redirection, and so on.
- 2018 research paper, "Practical Web Cache Poisoning" - https://portswigger.net/research/practical-web-cache-poisoning
- 2020 with a second research paper, "Web Cache Entanglement: Novel Pathways to Poisoning". - https://portswigger.net/research/web-cache-entanglement

- If a server had to send a new response to every single HTTP request separately, this would likely overload the server, resulting in latency issues and a poor user experience, especially during busy periods. Caching is primarily a means of reducing such issues. The cache sits between the server and the user, where it saves (caches) the responses to particular requests, usually for a fixed amount of time. If another user then sends an equivalent request, the cache simply serves a copy of the cached response directly to the user, without any interaction from the back-end. This greatly eases the load on the server by reducing the number of duplicate requests it has to handle.

- Cache Keys
	- When the cache receives an HTTP request, it first has to determine whether there is a cached response;
	- Caches identify equivalent requests by comparing a predefined subset of the request's components, known collectively as the "cache key";
	- Typically, this would contain the request line and `Host` header;
	- 🚨 If the cache key of an incoming request matches the key of a previous request, then the cache considers them to be equivalent;


### Constructing a web cache poisoning attack ( 3 steps )


> You can use the following extension from burp ( Param Miner )
> You can automate the process of identifying unkeyed inputs by adding the [Param Miner](https://portswigger.net/bappstore/17d2949a985c4b7ca092728dba871943) extension to Burp from the BApp store. To use Param Miner, you simply right-click on a request that you want to investigate and click "Guess headers". Param Miner then runs in the background, sending requests containing different inputs from its extensive, built-in list of headers. If a request containing one of its injected inputs has an effect on the response

1. Identify and evaluate unkeyed inputs
	1. You can identify **unkeyed** inputs manually by adding **random inputs** to requests and observing whether or not they have an **effect on the response**;

**Caution:** When testing for unkeyed inputs on a live website, there is a risk of inadvertently causing the cache to serve your generated responses to real users. **Therefore, it is important to make sure that your requests all have a unique cache key** so that they will only be served to you. To do this, y**ou can manually add a cache buster** (such as a unique parameter) to the request line each time you make a request. Alternatively, if you are using Param Miner, there are options for automatically adding a cache buster to every request.

#### Elicit a harmful response from the back-end server

> Once you have identified an unkeyed input, the next step is to evaluate exactly how the website processes it. **Understanding this is essential to successfully eliciting a harmful response.** If an input is reflected in the response from the server without being properly sanitized, or is used to dynamically generate other data, then this is a potential entry point for web cache poisoning.


### Get the response cached

- Manipulating inputs to elicit a harmful response is half the battle, but it doesn't achieve much unless you can cause the response to be cached, which can sometimes be tricky. Whether or not a response gets cached can depend on all kinds of factors, such as the file extension, content type, route, status code, and response headers.


---

### Exploiting cache design flaws

- Websites are vulnerable to web cache poisoning if they handle unkeyed input in an unsafe way and allow the subsequent HTTP responses to be cached.

##### Using web cache poisoning to deliver an XSS attack

1. When unkeyed input is reflected in a cacheable response without proper sanitization.

For example, consider the following request and response:

```http
GET /en?region=uk HTTP/1.1 
Host: innocent-website.com 
X-Forwarded-Host: innocent-website.co.uk 

HTTP/1.1 200 OK Cache-Control: public 
<meta property="og:image" content="https://innocent-website.co.uk/cms/social.png" />`
```

Here, the value of the `X-Forwarded-Host` header is being used to dynamically generate an Open Graph image URL, which is then reflected in the response. Crucially for web cache poisoning, the `X-Forwarded-Host` header is often unkeyed. In this example, the cache can potentially be poisoned with a response containing a simple [XSS](https://portswigger.net/web-security/cross-site-scripting) payload:

```http
GET /en?region=uk HTTP/1.1 
Host: innocent-website.com 
X-Forwarded-Host: a."><script>alert(1)</script>" 
HTTP/1.1 200 OK Cache-Control: public 
<meta property="og:image" content="https://a."><script>alert(1)</script>"/cms/social.png" />
```

If this response was cached, all users who accessed `/en?region=uk` would be served this XSS payload. This example simply causes an alert to appear in the victim's browser, but a real attack could potentially steal passwords and hijack user accounts.

#### Using web cache poisoning to exploit unsafe handling of resource imports

1. Some websites use **unkeyed** headers to dynamically generate URLs for importing resources, such as externally hosted JavaScript files. In this case, if an attacker changes the value of the appropriate header to a domain that they control, they could potentially manipulate the URL to point to their own malicious JavaScript file instead.
2. If the response containing this malicious URL is cached, the attacker's JavaScript file would be imported and executed in the browser session of any user whose request has a matching cache key.

```http
GET / HTTP/1.1 Host: innocent-website.com 
X-Forwarded-Host: evil-user.net 
User-Agent: Mozilla/5.0 Firefox/57.0 HTTP/1.1 200 OK 
<script src="https://evil-user.net/static/analytics.js"></script>
```

#### Using web cache poisoning to exploit cookie-handling vulnerabilities

A common example might be a cookie that indicates the user's preferred language, which is then used to load the corresponding version of the page:

```http
GET /blog/post.php?mobile=1 HTTP/1.1 Host: innocent-website.com 
User-Agent: Mozilla/5.0 Firefox/57.0 
Cookie: language=pl; Connection: close
```

In this example, the Polish version of a blog post is being requested. Notice that the information about which language version to serve is only contained in the `Cookie` header. Let's suppose that the cache key contains the request line and the `Host` header, but not the `Cookie` header. In this case, if the response to this request is cached, then all subsequent users who tried to access this blog post would receive the Polish version as well, regardless of which language they actually selected.

This flawed handling of cookies by the cache can also be exploited using web cache poisoning techniques. In practice, however, this vector is relatively rare in comparison to header-based cache poisoning. When cookie-based cache poisoning vulnerabilities exist, they tend to be identified and resolved quickly because legitimate users have accidentally poisoned the cache.

#### Using multiple headers to exploit web cache poisoning vulnerabilities

> Some websites are vulnerable to simple web cache poisoning exploits, as demonstrated above. However, others require more sophisticated attacks and only become vulnerable when an **attacker is able to craft a request that manipulates multiple unkeyed inputs**.


For example, let's say a website requires secure communication using HTTPS. To enforce this, if a request that uses another protocol is received, the website dynamically generates a redirect to itself that does use HTTPS:

```http
GET /random HTTP/1.1 
Host: innocent-site.com X-Forwarded-Proto: http 
HTTP/1.1 301 moved permanently 
Location: https://innocent-site.com/random
```

By itself, this behavior isn't necessarily vulnerable. However, by combining this with what we learned earlier about vulnerabilities in dynamically generated URLs, an attacker could potentially exploit this behavior to generate a cacheable response that redirects users to a malicious URL.

##### Exploiting responses that expose too much information

- Sometimes websites make themselves more vulnerable to web cache poisoning by giving away too much information about themselves and their behavior.

> One of the challenges when constructing a web cache poisoning attack is ensuring that the harmful response gets cached;

- Example:
	- When responses contain information about how often the cache is purged or how old the currently cached response is:
	```
	HTTP/1.1 200 OK 
	Via: 1.1 varnish-v4 Age: 174 
	Cache-Control: public, max-age=1800
	```

- The rudimentary way that the `Vary` header is often used can also provide attackers with a helping hand:
	- The `Vary` header specifies a list of additional headers that should be treated as part of the cache key even if they are normally **unkeyed**;
	- If the attacker knows that the `User-Agent` header is part of the cache key, by first identifying the user agent of the intended victims, they could tailor the attack so that only users with that user agent are affected.

#### Using web cache poisoning to exploit DOM-based vulnerabilities

- If the website unsafely uses **unkeyed** headers to import files, this can potentially be exploited by an attacker to import a malicious file instead. However, this applies to more than just JavaScript files.
- If you use web cache poisoning to make a website load malicious JSON data from your server, you may need to grant the website access to the JSON using [CORS](https://portswigger.net/web-security/cors):

```http 
HTTP/1.1 200 OK 
Content-Type: application/json 
Access-Control-Allow-Origin: *

{ "malicious json" : "malicious json" }
```

#### Exploiting cache implementation flaws

###### Cache key flaws

1. Any payload injected via keyed inputs would act as a cache buster, meaning your poisoned cache entry would almost certainly never be served to any other users.
- Many websites and CDNs perform various transformations on keyed components when they are saved in the cache key. This can include:

	- Excluding the query string
	- Filtering out specific query parameters
	- Normalizing input in keyed components

2. Cache probing methodology.
	1. The methodology of probing for cache implementation flaws differs slightly from the classic web cache poisoning methodology. These newer techniques rely on flaws in the specific implementation and configuration of the cache, which may vary dramatically from site to site. This means that you need a deeper understanding of the target cache and its behavior;
	2. The methodology involves the following steps:
		1. [Identify a suitable cache oracle](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws#identify-a-suitable-cache-oracle)
		2. [Probe key handling](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws#probe-key-handling)
		3. [Identify an exploitable gadget](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws#identify-an-exploitable-gadget)

##### Identify a suitable cache oracle

1.  A cache oracle is simply a page or endpoint that provides feedback about the cache's behavior;
2. This needs to be cacheable and must indicate in some way whether you received a cached response or a response directly from the server, this can be achieve:
	1.  An HTTP header that explicitly tells you whether you got a cache hit;
	2. Observable changes to dynamic content;
	3. Distinct response times;
> If you can identify that a specific third-party cache is being used, you can also consult the corresponding documentation.

- For example, Akamai-based websites may support the header `Pragma: akamai-x-get-cache-key`, which you can use to display the cache key in the response headers:

```http
GET /?param=1 HTTP/1.1 
Host: innocent-website.com 
Pragma: akamai-x-get-cache-key 
HTTP/1.1 200 OK 
X-Cache-Key: innocent-website.com/?param=1
```

###### Probe key handling

- The next step is to investigate whether the cache performs any additional processing of your input when generating the cache key;
- You should specifically look at any **transformation** that is taking place. Is anything being excluded from a keyed component when it is added to the cache key? Common examples are excluding specific query parameters, or even the entire query string, and removing the port from the `Host` header.

Let's say that our hypothetical cache oracle is the target website's home page. This automatically redirects users to a region-specific page. It uses the `Host` header to dynamically generate the `Location` header in the response:

`GET / HTTP/1.1 Host: vulnerable-website.com HTTP/1.1 302 Moved Permanently Location: https://vulnerable-website.com/en Cache-Status: miss`

To test whether the port is excluded from the cache key, we first need to request an arbitrary port and make sure that we receive a fresh response from the server that reflects this input:

`GET / HTTP/1.1 Host: vulnerable-website.com:1337 HTTP/1.1 302 Moved Permanently Location: https://vulnerable-website.com:1337/en Cache-Status: miss`

Next, we'll send another request, but this time we won't specify a port:

`GET / HTTP/1.1 Host: vulnerable-website.com HTTP/1.1 302 Moved Permanently Location: https://vulnerable-website.com:1337/en Cache-Status: hit`

##### Identify an exploitable gadget

- These gadgets will often be classic client-side vulnerabilities, such as [reflected XSS](https://portswigger.net/web-security/cross-site-scripting/reflected) and open redirects;
- Perhaps even more interestingly, these techniques enable you to exploit a number of unclassified vulnerabilities that are often dismissed as "unexploitable" and left unpatched. This includes the use of dynamic content in resource files, and exploits requiring malformed requests that a browser would never send.


#### Exploiting cache key flaws

##### Unkeyed port

- Example, consider the case we saw earlier where a redirect URL was dynamically generated based on the `Host` header. This might enable you to construct a denial-of-service attack by simply adding an arbitrary port to the request. All users who browsed to the home page would be redirected to a dud port, effectively taking down the home page until the cache expired. This kind of attack can be escalated further if the website allows you to specify a non-numeric port. You could use this to inject an XSS payload, for example.

##### Unkeyed query string

1. Like the `Host` header, the request line is typically keyed. However, one of the most common cache-key transformations is to exclude the entire query string.
2. 


### Unkeyed query parameters

So far we've seen that on some websites, the entire query string is excluded from the cache key. But some websites only exclude specific query parameters that are not relevant to the back-end application, **such as parameters for analytics or serving targeted advertisements. UTM parameters like `utm_content` are good candidates to check during testing**.

Parameters that have been excluded from the cache key are unlikely to have a significant impact on the response. The chances are there won't be any useful gadgets that accept input from these parameters. That said, some pages handle the entire URL in a vulnerable manner, making it possible to exploit arbitrary parameters.


###### Cache parameter cloaking

> If the cache excludes a harmless parameter from the cache key, and you can't find any exploitable gadgets based on the full URL, you'd be forgiven for thinking that you've reached a dead end. However, this is actually where things can get interesting.

> If you can work out how the cache parses the URL to identify and remove the unwanted parameters, you might find some interesting quirks. Of particular interest are any parsing discrepancies between the cache and the application. This can potentially allow you to sneak arbitrary parameters into the application logic by "cloaking" them in an excluded parameter.


Let's assume that the algorithm for excluding parameters from the cache key behaves in this way, but the server's algorithm only accepts the first `?` as a delimiter. Consider the following request:

`GET /?example=123?excluded_param=bad-stuff-here`

In this case, the cache would identify two parameters and exclude the second one from the cache key. However, the server doesn't accept the second `?` as a delimiter and instead only sees one parameter, `example`, whose value is the entire rest of the query string, including our payload. If the value of `example` is passed into a useful gadget, we have successfully injected our payload without affecting the cache key.


#### Exploiting parameter parsing quirks

Similar parameter cloaking issues can arise in the opposite scenario, where the back-end identifies distinct parameters that the cache does not. The Ruby on Rails framework, for example, interprets both ampersands (&) and semicolons (;) as delimiters. When used in conjunction with a cache that does not allow this, you can potentially exploit another quirk to override the value of a keyed parameter in the application logic.

Consider the following request:

`GET /?keyed_param=abc&excluded_param=123;keyed_param=bad-stuff-here`

As the names suggest, `keyed_param` is included in the cache key, but `excluded_param` is not. Many caches will only interpret this as two parameters, delimited by the ampersand:

1. `keyed_param=abc`
2. `excluded_param=123;keyed_param=bad-stuff-here`

Once the parsing algorithm removes the `excluded_param`, the cache key will only contain `keyed_param=abc`. On the back-end, however, Ruby on Rails sees the semicolon and splits the query string into three separate parameters:

1. `keyed_param=abc`
2. `excluded_param=123`
3. `keyed_param=bad-stuff-here`

But now there is a duplicate **keyed_param**. This is where the second quirk comes into play. If there are duplicate parameters, each with different values, Ruby on Rails gives precedence to the final occurrence. The end result is that the cache key contains an innocent, expected parameter value, allowing the cached response to be served as normal to other users. On the back-end, however, the same parameter has a completely different value, which is our injected payload. It is this second value that will be passed into the gadget and reflected in the poisoned response.

This exploit can be especially powerful if it gives you control over a function that will be executed. For example, **if a website is using JSONP to make a cross-domain request, this will often contain a callback parameter to execute a given function on the returned data:**

GET /jsonp?callback=innocentFunction
In this case, you could use these techniques to override the expected callback function and execute arbitrary JavaScript instead.


