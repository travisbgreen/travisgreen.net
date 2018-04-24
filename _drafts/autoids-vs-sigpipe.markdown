---
layout: "post"
title: "AutoIDS vs SIGPIPE"
date: "2018-04-24 13:23"
---
### Round 1: SIGPIPE 1 -- AutoIDS 0
A while back I ran into an issue using flask's built in webserver to serve AutoIDS where the app would simply hang. As part of troubleshooting, I would run it in a console window and when it would hang, I could still Ctrl+c, upon which 