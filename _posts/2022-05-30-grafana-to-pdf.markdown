---
layout: post
title:  "Export a Grafana dashboard to PDF with legends in Grafana v8+"
date:   2022-05-30 12:17:02 +0200
categories: grafana
---

# TL;DR

This short article describes how to automatically generate PDFs from a Grafana dashboard when:
- you are not using the paid Grafana Enterprise version
- you are using Grafana `v7.4.0` or more recent

# Context

If you use Grafana dashboards, you might want (or might have been asked) to generate PDF snapshots of those dashboards.
This is a feature supported by **Grafana Enterprise only** (starting from version `v6.7+`, see related [docs][grafana-pdfexport]).

Many, me included, use the Open Source version and have relied on an alternative relying on the "Print to PDF" option of web browsers, which can be automated using a headless browser using `NodeJS` and the `puppeteer` package. Full credit to GitHub user [svet-b][gist-user] and his detailed description found [here][gist-url].

However, since the update to `v7.4.0`, the legend is not properly shown when generating the PDF, as you can see from the screenshots below.

![The color lines of the legend are not properly rendered](/img/legend-nok.png)

I posted an [issue][issue] to Grafana's GitHub repository documenting the issue with the legend. It was not really investigated (neither by the Grafana team nor me).

A simple workaround is to simply take a (lossless) **screenshot** of the full page with a headless browser. The screenshot can then be converted to PDF via the command line.

# Installation

The screenshot is taken using `NodeJS` and the `puppeteer` package, and the PDF conversion is done via `img2pdf`. On a Debian-based system, the dependencies and programs can be installed via:

```
sudo apt install nodejs npm img2pdf gconf-service libasound2 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgcc1 libgconf-2-4 libgdk-pixbuf2.0-0 libglib2.0-0 libgtk-3-0 libnspr4 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 ca-certificates fonts-liberation libappindicator1 libnss3 lsb-release xdg-utils wget
```

Then, install the `puppeteer` package:

```
npm install puppeteer
```

# Running

The Node script generating the screenshot is the following:

{% highlight javascript %}
'use strict';

const puppeteer = require('puppeteer');

// URL to load should be passed as first parameter
const url = process.argv[2];
// Username and password (with colon separator) should be second parameter
const auth_string = process.argv[3];
// Output file name should be third parameter
const outfile = process.argv[4];

// TODO: Output an error message if number of arguments is not right or arguments are invalid

// Set the browser width in pixels. The paper size will be calculated on the basus of 96dpi,
// so 1200 corresponds to 12.5".
const width_px = 1200;
// Note that to get an actual paper size, e.g. Letter, you will want to *not* simply set the pixel
// size here, since that would lead to a "mobile-sized" screen (816px), and mess up the rendering.
// Instead, set e.g. double the size here (1632px), and call page.pdf() with format: 'Letter' and
// scale = 0.5.

// Generate authorization header for basic auth
const auth_header = 'Basic ' + new Buffer.from(auth_string).toString('base64');

(async() => {
  const browser = await puppeteer.launch({args: ['--no-sandbox', '--disable-setuid-sandbox']});
  const page = await browser.newPage();

  // Set basic auth headers
  await page.setExtraHTTPHeaders({'Authorization': auth_header});

  // Increase timeout from the default of 30 seconds to 120 seconds, to allow for slow-loading panels
  await page.setDefaultNavigationTimeout(7200);

  // Increasing the deviceScaleFactor gets a higher-resolution image. The width should be set to
  // the same value as in page.screenshot() below. The height is not important but should be long
  // enough to get the whole page
  await page.setViewport({
    width: width_px,
    height: 10000,
    deviceScaleFactor: 2,
    isMobile: false
  })

  // Wait until all network connections are closed (and none are opened withing 0.5s).
  // In some cases it may be appropriate to change this to {waitUntil: 'networkidle2'},
  // which stops when there are only 2 or fewer connections remaining.
  await page.goto(url, {waitUntil: 'networkidle0'});

  // Hide all panel description (top-left "i") pop-up handles and, all panel resize handles
  // Annoyingly, it seems you can't concatenate the two object collections into one
  await page.evaluate(() => {
    let infoCorners = document.getElementsByClassName('panel-info-corner');
    for (el of infoCorners) { el.hidden = true; };
    let resizeHandles = document.getElementsByClassName('react-resizable-handle');
    for (el of resizeHandles) { el.hidden = true; };
  });

  // Get the height of the main canvas, and add a margin
  var height_px = await page.evaluate(() => {
    return document.getElementsByClassName('react-grid-layout')[0].getBoundingClientRect().bottom;
  }) + 20;

  console.log(height_px);

  await page.screenshot({
    path: outfile,
    clip: {
      x: 0,
      y: 0,
      width: width_px,
      height: height_px
    }
  });

  await browser.close();
})();

{% endhighlight %}

The script, for example saved as `grafana_pdf.js`, is then run by the following command :

```
node grafana_pdf.js DASHBOARD_URL USERNAME:PASSWORD FILENAME
```

where:

- `DASHBOARD_URL` is the URL of the dashboard to generate
- `USERNAME:PASSWORD` are your credentials, for example the defaut `admin:admin`
- `FILENAME` is the expected output filename

The arguments can also be hardcoded in the code, or used as environmental variables depending on your preferences.

The PNG screenshot can then be converted to PDF via

```
img2pdf screenshot.png -o output.pdf
```

The series of command can easily be automated via a cron job to generate a report at a given interval.




[grafana-pdfexport]: https://grafana.com/docs/grafana/latest/enterprise/export-pdf/
[gist-user]: https://gist.github.com/svet-b
[gist-url]: https://gist.github.com/svet-b/1ad0656cd3ce0e1a633e16eb20f66425
[issue]: https://github.com/grafana/grafana/issues/36656