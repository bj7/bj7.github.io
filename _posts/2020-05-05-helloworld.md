---
layout: post
title: "CSS Comments"
categories: [blog, css, gotchas]
tags: [doh, shouldveknown]
---
<!-- ### Why can't we use `//` as CSS comments -->
While at work, I was presented with a seemingly mundane bug. I had supplied another co-worker with some CSS overrides to apply to a stylesheet. He responded saying my first two overrides didn't work, but the last two operated correctly.
```css
// sidebar card-header darkening
.sidebar .card-header {
background-color:#d2d2d2
} 

// card-header font darken
.dash-sidebar .dash-sidenav>li>.sidebar.card>.card-header>.btn-link>a, .dash-sidebar .links-menu>li>.sidebar.card>.card-header>.btn-link>a {
color: rgb(74, 74, 74) !important // enables override of single-graph view styles
} 

// alternatively, for the single-graph view font
.graphView #sidebar .sidebar>.card-header>.btn-link>a, .dash-sidebar span, a {
color: rgb(74,74,74)
} 

.dash-sidebar .dash-sidenav>li>.sidebar.category-select>.panel-collapse>.panel-body, .dash-sidebar .links-menu>li>.sidebar.category-select>.panel-collapse>.panel-body {
background-color: #f1f1f1
} 
```
Most people familiar with CSS probably know that `/* */` is the correct comment form. I was aware that was a correct comment, but I did not know that I could not use `//` as a single line comment. Why is that not valid, and why did the last two work correctly?

Well, the most obvious reason is that it's not defined in the [specs](https://www.w3.org/TR/CSS2/syndata.html#comments), but what happens when a single line comment is used unknowingly? Well, the [current specs](https://drafts.csswg.org/css-syntax-3/#consume-comment) describe the token consumation looking for a solidus `/` followed immediately by an asterisk `*`. The comment then ends in reverse order `*/`. This particular token consumation returns nothing.

Now when we have two solidi `//`, the tokenizer doesn't seem to treat it as (anything)[https://drafts.csswg.org/css-syntax-3/#consume-token] other than a delim-token with a value of the previous input code point (a unicode code point). It appears this causes undefined behavior based on what was the preceding, valid code point.

I assume the first
```css
 // sidebar card-header darkening
 .sidebar .card-header {
 background-color:#d2d2d2
 }
 ```
 failed because it was immediately appended to the stylesheet with no blank lines, and the preceding code point was valid CSS, thus causing a parse error to follow. The second 
 ```css
 // card-header font darken
 .dash-sidebar .dash-sidenav>li>.sidebar.card>.card-header>.btn-link>a, .dash-sidebar .links-menu>li>.sidebar.card>.card-header>.btn-link>a {
 color: rgb(74, 74, 74) !important // enables override of single-graph view styles
 }
 ```
 failed due to the single line comment following the `important` keyword, and resulting in a parse error.
 
 How did the third one not fail though? My guess is because the preceding code point was a newline, so the CSS parser simply ignored that single line comment as a continuation of the newline code point. Obviously the fourth one worked because there were no comments. 