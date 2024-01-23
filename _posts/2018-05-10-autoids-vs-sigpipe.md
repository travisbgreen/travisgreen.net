---
layout: post
title: AutoIDS vs SIGPIPE
date: '2018-05-10 09:57'
---

### Round 1: The problem
Have you ever used flask's built in webserver and thought "this is probably good enough to use for my little thing"? I've discovered that it isn't good for production, and it took a few missteps to find out why. 

I deployed [AutoIDS](http://autoids.net) on a low cost VPS, fired up the .py file, and after a few hours/days, the app would simply hang. If you press ctrl+c in the console window, you'd see a fairly cryptic error message:

```
error: [Errno 32] Broken pipe
```

What gives? Well, it seems that this happens when a [client closes a socket](https://stackoverflow.com/questions/11866792/how-to-prevent-errno-32-broken-pipe?noredirect=1&lq=1), and when the process attempts to continue writing to the closed socket it recieves `SIGPIPE` [signal](https://www.quora.com/What-are-SIGPIPEs), which flask does not accomodate. Why does the client socket close? I suspect it is due to high speed scanners initiating a session but not completing it for the sake of performance. I've seen speculation that browsers check the first few bytes of an image and close the socket if the first `n` bytes match what is in cache (although I can't find the original reference to this, so grain of salt).

### Round 2: The solution
The thing to do is to follow the [guidance](http://flask.pocoo.org/docs/0.12/deploying/) of the flask project and deploy your app using WSGI. I found Apache's mod_wsgi to be a good fit, so I followed the [instructions](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/) and after some trial and error structring and importing the app within mod_wsgi, everything works well. In the future, I'll be sure to structure my app to conform to mod_wsgi's deployment scheme from the start rather than refactor later to accomodate. 

