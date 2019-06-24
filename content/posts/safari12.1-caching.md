---
title: "Safari 12.1 and caching changes"
date: 2019-06-23T07:32:07-07:00
tags: ["safari", "caching", "beacon", "javascript"]
---

In April this year, our webpage started observing a sudden movement in the convertion rates for one of the page load web beacons (tracking pixel) on mobile. On digging deeper it became apparent that the movement coincided with the release of [safari version 12.1](https://webkit.org/blog/8718/new-webkit-features-in-safari-12-1/). As this new version of safari was gradually rolled out across mobile devices in the first half of April, the number of page load event beacons saw an increase which was unexpected. Troubleshooting this revealed a change in the safari behavior. To understand this more, breaking down the beaconing basics
<!--more-->

## Beacon
Beacon requests are typically constructed in Javascript to log tiny amount of data back to the server. This can be done using multiple Javascript techniques such as constructing an image object and setting the source, sending via XHR or using the native [navigator.sendbeacon](https://developer.mozilla.org/en-US/docs/Web/API/Beacon_API) interface if available.

The server responds to these requests with a small content (like a 1x1 pixel image) or "no content" HTTP [status code 204](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/204). These requests are cacheable and the appropriate cache-control headers need to be provided so that browser will always send the request to the server and not serve from cache which would result in data loss to the server. If its a 3rd party server, the response headers however are not controllable.

To workaround the caching issue for the beacon request, the javascript beacon utility added a [cache busting](https://www.maxcdn.com/one/tutorial/cache-busting/) parameter to the URL like a timestamp so that the URL will be unique and unavailable in the cache and forces the browser to make the request

{{< highlight javascript >}}
function log(beaconUrl) {
    // Timestamp for cache busting
    beaconUrl += '&ts=' + (new Date()).getTime();
    fireBeacon(beaconUrl);
}

function fireBeacon(url) {
    if (window.navigator && (typeof window.navigator.sendBeacon === 'function')) {
        navigator.sendBeacon(url);
    } else {
        // Fallback methods
    }
}
{{< /highlight >}}

## Safari changes
It looked like everything was in place to successfully fire the beacon request for a page load event. Analyzing the data showed that the increase in the beacons seemed to be from a change to the back button behavior from safari version 12.1.

So a user would visit the page, click on a link and navigate away and then press back to return to the original page. The back to the original page would serve all cache-able content from the browser cache and then rerun all the javascript on page and trigger the onload. The beacon utility would then append the timestamp and fire a new request.

### Before Safari 12.1
For safari versions 12.0 and earlier, it was found that the back button would incorrectly return the old timestamp value from the original page load for a (`(new Date()).getTime();`) call and generate a stale beacon URL value, ending up with the beacon served from cache. It appears to be some sort of aggresive Javascript optimization by the browser to cache the function return value on the back button. So the beacons were not accurately counting the page loads in the previous version. This was observed across desktop, tablet and mobile safari and for other methods for generating unique values such as:

{{< highlight javascript >}}
(new Date()).getTime();

window.performance.now();

Math.random();
{{< /highlight >}}

### Safari 12.1 and onwards
This behavior seems to have now been rectified with version 12.1 and higher and the observed uplift in the beacon count was correct and attributed to the back button page views. Safari is now more inline with other modern browsers generating unique values per call regardless of the page navigation.
