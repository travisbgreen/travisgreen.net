---
layout: "post"
title: "AutoIDS vs SIGPIPE"
date: "2018-04-24 13:23"
---
### Round 1: SIGPIPE 1 -- AutoIDS 0
Have you ever used flask's built in webserver and thought "this is probably good enough to use for my little thing". Well, I've discovered that it isn't good for anything but local development. It took a few missteps to find out why. I deployed [AutoIDS](http://autoids.net) on a low cost VPS, fired up the .py file, and after a few hours/days, the app would simply hang. If you press ctrl+c in the console window, you'd see a fairly cryptic error message:

```


```