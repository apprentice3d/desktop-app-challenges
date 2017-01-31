## Forge API in desktop apps: challenges and solutions

***"Don’t Check Passwords into Source Control or Hard-Code Them in Your Application. Operations staff will remove your eyes with a spoon if they catch you doing this. Don’t give them the pleasure."***

This "nice" quote is from ["Continuous Delivery" book](http://martinfowler.com/books/continuousDelivery.html) and this continues with: 

***"Passwords should always be entered by the user performing the deployment. There are several acceptable ways to handle authentication for a multilayer system. You could use certificates, a directory service, or a single sign-on system"***

All this is nice and clear in case of a "WebApp", or I like better the software-as-a service name, where usually the client is a browser (or a thin desktop client) and you control the server that connects to Forge services. Something like the following schematic view:

![](img/webApp.png)

You are in control of the "WebApp" server, where you can use best practices to secure the Forge related secrets (at least Client ID and Client Secret, and better do that for tokens too), and if you look at our samples, we always recommend setting your secrets using environment variables, instead of hard coding them (maybe with some few exceptions).

------
![](img/01.png)

------

However, what about a desktop only application? What if I want to develop a desktop application that directly connects to Forge servers?
I just want to compile my app and distribute the binary to my customers, and I don't want to waste my time with maintaining a server.

### How the Forge related secrets could be secured in case of a desktop app, what are the best practices and what other challenges await us ahead?

This is exactly what we want to explore in our endeavor of developing a "pure" desktop app, schematically represented like this:

![](img/pureApp.png)

No other services, no proxy, just directly calling the Forge endpoints.


So ... we have at least a pair of secrets (Client ID and Client Secret) on our hand, how can we secure them in a desktop environment?

To set them into an environment variable, as we usually suggest for WebApps, is of course a catastrophic idea from security perspective, it is completely transparent and people with spoons will come for you sooner than you expected.


#### What about hard-coding them into source code (using a compiled language), compile it and distribute just the binary? 

At first sight it seems an useful approach, the very light version of [security by obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity).

Ok then, let us examine closer this approach through creating a very simple "Forge Hello Wolrd" program and what can be better that using C++ as an illustration language, right?

![](img/cpp_example.png)

A self-explanatory example, where we see our secrets, but we are not going to distribute our code to our customers, instead we compile it and distribute just the binary, so our secrets are safe, right?

Well ... not really ...
Take any Unix-like OS, and just run [strings](https://en.wikipedia.org/wiki/Strings_(Unix)) command on your binary and you will be surprised how many things comes up, along with our secrets:

![](img/strings.png)
 
An expected reaction to this is "C'mon, who keeps secrets in plain text, you should scramble them, encrypt them or even better put them into a database and fetch them when needed".

Yes, it is true, that will increase the difficulty of getting your secrets, but will not make it impossible, and it might not be so difficult after all.

To illustrate this, we can use any disassembler and have a look at our code:

![](img/disassamble.png)

Here we again see our beloved secrets and yes our secrets are still in plain text, just reinforcing the above idea in a more visually appealing context. 

Now let us ask ourselves, if I can see the variables, can I see the functions/procedures? 

And the answer is obviously "yes" and any self-respecting disassembler, will give you the ability not only see the callstack, but also the call paths:

![](img/funcs.png)

"Why should we care?" - you may ask. Well ... it doesn't matter how well you [obfuscated](https://en.wikipedia.org/wiki/Obfuscation_(software)) your code, the machine must understand it and functions calls must be made - meaning that you can trace those calls and identify the procedure responsible for assembling the request to Forge endpoints. Thus, sooner or later you must decrypt your secrets, or fetch them for a database, and assemble them into a request to Forge endpoints, which means that at a given moment your secrets will be exposed in the call stack, and all that somebody needs is just to "camp" in that very procedure and wait for your secrets to pop-up.

We should understand one thing, if your desktop app is on somebody's machine, he can do anything he likes with it: crack it, break it and even cheat it.

For instance, to get a [2-legged token](https://developer.autodesk.com/en/docs/oauth/v2/tutorials/get-2-legged-token/), you have to send a POST request (along with your secrets) to ***https://developer.api.autodesk.com/authentication/v1/authenticate***, but who said that it is not possible to fool the app and redirect to your own TLS-enabled server, where you can transparently see the given secrets?

Also, your application must rely on some libraries assuring a secure network connection, like [OpenSSL](https://www.openssl.org). If your application is dynamically linking to those library, then there is no problem for user to compile its own version of that library, with security reduced to zero, and give that library to your application. Then he can easily listen to all the traffic and extract from it the needed secrets.
 

In conclusion, as in quote from one of my favorite movies ["the only winning move is not to play"](http://www.imdb.com/title/tt0086567/?ref_=ttqt_qt_tt) and in this case, the only winning move is to neither hard-code, nor even pass through your app the secrets, as "Three may keep a Secret, if two of them are dead" (Benjamin Franklin).


#### Security through updates?
Meantime, somehow related to another angle on security, asks yourself this question: *"If I'll embed/hard-code them, how I am supposed to update them in case my secrets are compromised?"*. 

In this case, you will have to change and recompile your app and of course change them for each client individually *(not remotely update them - you said that you don't want to keep a server for that application)* - this is exactly the part when happiness of having thousands of clients using your application is rapidly changing to sorrow.


All that means that hard-coding secrets into your desktop app is a bad idea not only because of "guys with spoons", but also because of problems related to change of Forge secrets (due to compromised secret, migration to another Forge app etc.), and in the end this is also a non-trivial challenge.


#### Enough with security, is there anything else?

Unfortunately, there is. 

Sooner or later your app will need a 3-legged token, and here is another fun problem ...

You must get the end user's consent to access Forge resources. Since (fortunately or not) Forge OAuth **is not supporting** the ["password grant type"](https://support.zendesk.com/hc/en-us/articles/203663836-Using-OAuth-authentication-with-your-application) for exchanging the username and password for an access token directly, you will have to redirect the end user to Autodesk server, by ether opening the system browser or embedding the sign in page in a WebView ([strongly discouraged](https://developers.google.com/identity/protocols/OAuth2InstalledApp)) - it doesn't matter now. You must extract the authorization code from the ```redirect_uri```, and to do that you must listen for it, which means ... you should have a local server - so anyway you will need a server (yeah, but it is a local one). But wait, there is more: this server must open the port specified in **CallBack URL**

-------
![](img/05.png)

-------

and this means that in case of installation on client's machine, you must have the rights to open this port and **that port should be available**. I assume that you could do a survey among your clients and find out what ports they have available and tailor your CallBack URL to the commonly available port, but ... I hope you understand that I am just kidding.

Funny, yet quite challenging problem to solve

#### Is there more?

Yes, it is. Final within this article, yet I am afraid it is not final to be dealt with.

In many cases, as a defensive measure for different abuses, there are some limits/restrictions imposed on some calls. For example, in case of this request for getting the up-to-date list Forge-supported translations:

![](img/limits.png)

When the user of your desktop application is starting the app, it might be good idea to make sure that the he gets the latest available list of supported translations, so your app will call this endpoint. As a defensive measure, this api call mentions as a rate limit: "30 calls per minute", which by [pigeonhole principle](https://en.wikipedia.org/wiki/Pigeonhole_principle) (yes I am showing off with my Math knowledge), means that at least 31st call will be problematic, and here a nice challenge for you to deal with: "No more than 30 apps should start "world-wide" within a 1 minute time-frame". Please don't give me ["Impossible...but doable"](http://www.imdb.com/title/tt0137494/?ref_=ttqt_qt_tt) like replies - this is usually my answer.


### Conclusion

Thus, the identified challenges that we should deal with, when developing "pure" desktop applications using (in general) REST APIs, and Forge APIs (in particular), are: 

(1) **Forge secrets security**

(2) **Forge secrets update** (including access revocation) 

(3) **Resource (ports) availability**

(4) **Service restrictions/limits** 

and many others (like accountability).



In the end, I want to note that I am not saying that it is impossible to create Forge powered "pure" desktop applications, but merely that the effort of dealing with at least above-mentioned questions, is substantial compared to just having a server in the cloud.

Moreover, having a cloud server gives you more control and opportunities, and I am not referring here to scalability, availability and other *abilities, but more of things like having your server act as a proxy and serving requests through it, thus giving you more fine control over things like token monitoring and revocation, access monitoring and control, task prioritization etc.

