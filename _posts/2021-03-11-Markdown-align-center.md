---
title: How to align images in markdown
author: Edwin
date: 2021-03-11 15:11:41 +0800
categories: [Blogging, Tutorial]
tags: [Markdown]
---

Image alignment in markdown
Normal markdown image tags don’t allow for any alignment properties and thats a bummer when you are trying to make your README.md file pretty on github.

``` 
<!-- No alignment options -->

![GitHub Logo](/images/logo.png)

```

Luckily, we can use html image tags to make enhance our docs.

``` 
<!-- Alignment options!!!!! -->
<img align="left" width="100" height="100" src="http://www.fillmurray.com/100/100">
```

**Left alignment**

This is the code you need to align images to the left:

``` 
<img align="left" width="100" height="100" src="http://www.fillmurray.com/100/100">

```

**Right alignment**

This is the code you need to align images to the right:

``` 
<img align="right" width="100" height="100" src="http://www.fillmurray.com/100/100">
```

**Center alignment example**

Wrap images in a p or div to center.

``` 
<p align="center">
  <img width="460" height="300" src="http://www.fillmurray.com/460/300">
</p>

```

**Markdown Formatting on steroids**

If you like this, you might enjoy markdown-magic. I built it to automatically format markdown files and allow folks to sync docs/code/data from external sources.
