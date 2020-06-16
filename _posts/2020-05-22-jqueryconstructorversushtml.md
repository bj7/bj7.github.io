---
layout: post
title: "Quirks in the jQuery Constructor"
categories: [blog, javascript, gotchas, jQuery]
tags: [doh, bugs, js, jquery]
---
Another war story from work.

This was a pretty time consuming bug to hunt down, but
once discovered, it seems readily searchable, and easily fixable. 

I got an email this morning saying our user-sharing system was broken. Every time a user would go to share something, a JS error would popup saying the expected `shareUserItem` function call for the `onclick` event was not defined. Pretty odd since none of that code had been touched in months, and it was working before.

So I do some digging. The share icon iniates a bootstrap modal that contains sharing options. The html for the modal is retrieved via an ajax call to our server where PHP generates the desired html and returns it to be rendered via jQuery. After verifying that the PHP was operating correctly, and the modal did show, I started to try and determine why the actual share command failed. 

The `onclick` event that triggered the `shareUserItem` function was present, as was the proper script tags containing the function definition. Now, the functions were appended to the `window` (not the best idea, but legacy systesm and all...)
```
window.shareUserItem = function() {.....}
```
I figured maybe the `onclick` was improperly calling the share function, so I tested in the console if the window had the function. No, it didn't. So somehow the function definitions were not being placed on the window object. 

Why would the window not have the function? Something was wrong with the processing of the script tags in this dynamic modal content, but what? 

A quick solution would have been to run `eval` on all the scripts, like
```
eval($('#myscript').text());
```
This would work, but `eval` is best used sparingly if at all, and I didn't want just another hack. So I continued to dig. My first thought was that perhaps it was some issue wit bootstrap 4 or our version of jQuery, since those had been more recently updated. I couldn't find any evidence of that. 

So after a lot of googling, browsing StackOverflow, and general bewilderment at why it worked before, I came across a StackOverflow issue that was similar to what I was experiencing. 

Somone was having an issue getting included scripts to parse when using jQuery's `append`. The modal I was working with didn't use `append`, but was created like
```
var m = $(data).modal(
    {
        show: true
    }
);
```
 So that led me to start really staring at the jQuery docs for creating html. Well, it turns out that in the `.html()` documentation, there's a little note about code execution:
>By design, any jQuery constructor or method that accepts an HTML string — jQuery(), .append(), .after(), etc. — **can potentially** execute code. This can occur by injection of script tags or use of HTML attributes that execute code (for example, <img onload="">). Do not use these methods to insert strings obtained from untrusted sources such as URL query parameters, cookies, or form inputs. Doing so can introduce cross-site-scripting (XSS) vulnerabilities. Remove or escape any user input before adding content to the document.

(emphasis mine)

I figured that the code **should be** executing after using the constructor. I went to verify if the code would execute by switching to 
```
var m = $("#shareModal").html(content).modal({
    show: true
})
```
Boom! The scripts were being processed! 

Why would `.html()` make the difference though? After another google search about jQuery constructors and script tags, I came across a StackOverflow answer explaining my exact situation. [The constructor can mess with script tags](https://stackoverflow.com/a/2699905). Another [answer](https://stackoverflow.com/a/19034133) right below explained about using `$.parseHTML` and making sure the `keepScripts` flag was true and then injecting using jquery. So that was the solution.
```
var content = $($.parseHTML(data, null, true))
var m = $("#shareModal").html(content).modal({
    show: true
})
```

This code worked in the past because jQuery is loose with whether or not scripts are executed, but after the recent jQuery version upate for our library, something changed that stopped the implicit execution.