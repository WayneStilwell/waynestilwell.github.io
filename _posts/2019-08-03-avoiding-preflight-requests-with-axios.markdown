---
layout: post
title: "Avoiding Preflight Requests with Axios"
date: 2019-08-03 22:45:20 -0600
description: How can we avoid the CORS preflight request in an axios.post()?
img: # Add image post (optional)
tags: [CORS,axios,Firebase]
---

## Background
Sometimes our AJAX requests can trigger a [CORS preflight](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Preflighted_requests). An HTTP request is sent first, with the OPTIONS method, to verify that the actual request we want to make is safe. 

Not all requests trigger a preflight check. These are known as ["simple requests"](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Simple_requests).

What happens when we can't make a "simple request", but want to avoid the preflight?

_If you're not familiar with cross-origin resource sharing, [this article from MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) is a great read._

## My Scenario
At work, I inherited a VueJS SPA hosted in Firebase. I was tasked with integrating our division's single sign-on into the app. With the serverless nature of Firebase, I was forced to use cloud functions to do most of the heavy lifting.

One part of this project involved creating new users in the SSO system. For this, I wanted a cloud function that would accept the new user's info (along with an access token). The function would post to the SSO API, parse the response from SSO, and return it to the client.

## Headaches due to CORS
VueJS, Firebase, "serverless", CORS... all of this was new to me. I followed examples online, and put a function together I thought could get the job done. It just wouldn't work though!

I added debug statements, checked the Firebase logs, and pin-pointed where the problem was. My cloud function was executing a POST to our division's SSO API. That POST required a custom header for authentication. This was triggering an OPTIONS preflight request.

Now, the actual POST would eventually run. But the cloud function was processing the preflight as if it was the actual request, and returning that response. No bueno!

## How I Made It Skip The Preflight
Making the content type *application/x-www-form-urlencoded* did the trick. Notice in the code sample below, I converted my request body into a query string instead of passing it as an object:

```js
// I'm not including other modules I imported
// I just wanted to highlight these two
const cors = require('cors')({
    origin: true
});
const qs = require('querystring')

exports.createUser = functions.https.onRequest((request, response) => {
    const postBody = JSON.parse(request.body)
    
    const endPoint = 'https://your.url.here'
    
    const headers = {
      headers: {
        'Authorization': 'Bearer ' + postBody.accessToken,
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    }
    
    const requestBody = {
      someField: postBody.someField,
      anotherField: postBody.anotherField
    }

    axios.post(endPoint, qs.stringify(requestBody), headers)
      .then((ssoResponse) => {
        return cors(request, response, () => {
          response.status(200).send(ssoResponse.data);
        }
      })
      .catch((error) => {
        return cors(request, response, () => {
          response.status(400).send(`Failed to call SSO API to create user: ${error}`);
        })
      })
})
```
