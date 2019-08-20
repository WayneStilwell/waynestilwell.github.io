---
layout: post
title: "Installing Babel CLI on macOS Mojave"
date: 2019-08-05 13:45:20 -0600
description: How to install the Babel CLI on macOS.
img: # Add image post (optional)
tags: [babel,macOS]
---

I'm taking a modern JavaScript course on [Udemy](https://www.udemy.com/course/modern-javascript). I highly recommend it if you've never used ES6 before.

The instructor introduces Babel in one of the sections, and has us install the Babel command line tool.

Using npm, you should be able to install it like so:

```bash
npm install -g babel-cli@6.26.0
```

When I tried this, I got an "EACCES: permission denied" error.

You can try following the instructions [here](https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally) to fix it for you. A lot of people in the course had success with this.

I don't know what I did to my MacBook, but the suggestions above STILL did not fix it for me.

After scouring Stack Overflow and Google, the only way I could get the Babel CLI to work was to recursively update the permissions of the .npm folder in my home directory:

```bash
sudo chown -R $(whoami) ~/.npm
```

I hope this helps somebody!
