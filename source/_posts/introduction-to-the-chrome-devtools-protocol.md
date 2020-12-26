---
title: Introduction to the Chrome DevTools Protocol 
description: Automate your code inspections, debugging, and testing!
date: 2018-04-07 22:19:17
tags:
    - Programming
    - Last Sprint Today
    - JavaScript
---

At the end of every two-week [MindTouch Engineering](https://mindtouch.com) sprint, [Patty Ramert](https://www.linkedin.com/in/pramert) hosts _Last Sprint Today_: a chance for us to share with other MindTouchers what we've learned, anything we are working on, or any technical topic of interest. This week, I presented an introduction to the [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol).

{% vimeo 484328827 %}

Here is the simple [Puppeteer](https://developers.google.com/web/tools/puppeteer) code that I used for the demo (checking CSS rule coverage on a webpage):

```js
const puppeteer = require('puppeteer');

// simple puppeteer CSS coverage tracking
(async () => {
  const browser = await puppeteer.launch({ devtools: true });
  const page = await browser.newPage();  
  await page.coverage.startCSSCoverage()
  await page.goto('https://mindtouch.com');
  const coverage = await page.coverage.stopCSSCoverage();
  await browser.close();

  // output coverage report
  let totalBytes = 0;
  let usedBytes = 0;
  const deadRules = [];
  for(const entry of coverage) {
    totalBytes += entry.text.length;
    if(!entry.ranges.length) {
      deadRules.push(entry.text);  
    }
    for(const range of entry.ranges) {
      usedBytes += range.end - range.start - 1;
    }
  }
  console.log(`Total bytes: ${totalBytes}`);
  console.log(`Used bytes: ${usedBytes}`);
  console.log(`Dead rules: ${JSON.stringify(deadRules)}`);
})();

// manual implementation of CSS coverage tracking
(async () => {
  const browser = await puppeteer.launch({ devtools: true });
  const page = await browser.newPage();
  const client = await page.target().createCDPSession();
  await client.send('CSS.startRuleUsageTracking');
  await page.goto('https://mindtouch.com');
  await client.on('SomeDomain.someEvent', (data) => doSomething(data));
  const coverage = await client.send('CSS.stopRuleUsageTracking');

  // output coverage report
  // (coverage data is likely in a different format than page.coverage)
})();
```
