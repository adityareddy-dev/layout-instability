# Computing Cumulative Layout Shift from layout-shift entries

[Cumulative Layout Shift (CLS)](https://web.dev/articles/cls) measures the largest
burst of unexpected layout shifts during a page visit. Libraries compute this
metric from the entries exposed by the
[Layout Instability API](https://wicg.github.io/layout-instability/).
This document explains the calculation. It does not add requirements to the API.

## Eligible shifts

Only entries whose
[`hadRecentInput`](https://wicg.github.io/layout-instability/#dom-layoutshift-hadrecentinput)
is false contribute to the score. The API sets this flag according to its
[excluding-input rules](https://wicg.github.io/layout-instability/#excluding-input).
Use that flag rather than deciding from the observer callback time whether an
input was recent.

Process eligible entries in order of their `startTime`. Keep the window state
across observer callbacks, since callback batches do not define the windows.

## Session windows

A session window starts with the first eligible shift. Each later eligible shift
joins that window if it occurs less than 1,000 milliseconds after the previous
eligible shift and less than 5,000 milliseconds after the first shift in the
window. Otherwise, it starts a new window.

A gap of exactly 1,000 milliseconds, or a shift exactly 5,000 milliseconds after
the first shift, starts a new window. Excluded entries do not extend a window.
The score of a window is the sum of its contributing layout shift values.
Keep the largest window score, or zero if there are no eligible shifts.

For example, shifts with values 0.1 and 0.2 at 100 and 600 milliseconds form a
window with score 0.3. A shift with value 0.25 at 1,600 milliseconds starts a new
window, so the score remains 0.3 rather than increasing to 0.55.

## Documents and frames

The [document cumulative layout shift (DCLS) score](https://wicg.github.io/layout-instability/#document-cumulative-layout-shift-dcls-score)
uses shifts within one document. A `PerformanceObserver` in that document does
not receive entries from its child frames.

For the [CLS score](https://wicg.github.io/layout-instability/#cumulative-layout-shift-cls-score),
combine eligible shifts from the top-level browsing context and its descendant
contexts into one timeline before forming windows. Express all times relative
to a common time origin. Each top-level shift contributes its layout shift value.
Each descendant shift contributes its value multiplied by the
[subframe weighting factor](https://wicg.github.io/layout-instability/#subframe-weighting-factor),
the fraction of the top-level viewport occupied by that descendant's viewport.

Adding the largest scores from separate documents would combine windows that
might have occurred at different times. Page JavaScript also cannot read
cross-origin child frames' entries without their cooperation, so its observed
score can differ from a browser's full-page measurement.

## Example for one document

This example computes DCLS for the observed document. It keeps the largest
session-window score and processes pending entries before reporting each time
the document becomes hidden. `sendToAnalytics` is defined by the application.

```js
let documentCumulativeLayoutShiftScore = 0;
let sessionScore = 0;
let sessionStartTime;
let previousShiftTime;

function updateDCLS(entries) {
  for (const entry of entries) {
    if (entry.hadRecentInput)
      continue;

    if (previousShiftTime !== undefined &&
        entry.startTime - previousShiftTime < 1000 &&
        entry.startTime - sessionStartTime < 5000) {
      sessionScore += entry.value;
    } else {
      sessionScore = entry.value;
      sessionStartTime = entry.startTime;
    }
    previousShiftTime = entry.startTime;
    documentCumulativeLayoutShiftScore = Math.max(
      documentCumulativeLayoutShiftScore, sessionScore);
  }
}

const observer = new PerformanceObserver((list) => {
  updateDCLS(list.getEntries());
});
observer.observe({type: 'layout-shift', buffered: true});

document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    updateDCLS(observer.takeRecords());
    sendToAnalytics({documentCumulativeLayoutShiftScore});
  }
});
```

## Page lifecycle

Becoming hidden is a chance to report the score, since an unload event might not
arrive. If the user returns to the page, continue observing and report again
when it becomes hidden next.

The example illustrates windowing for one document. Production measurement also
needs page lifecycle handling. A back/forward cache restoration starts a new
page visit, so reset the score and window state. A page that starts in the
background, or becomes hidden before its first content is painted, should not
report a CLS value. See the
[metric and API differences](https://web.dev/articles/cls#differences-between-the-metric-and-the-api)
for these cases.

## Further reading

- [CLS definition and measurement guidance](https://web.dev/articles/cls)
- [web-vitals CLS implementation](https://github.com/GoogleChrome/web-vitals/blob/main/src/onCLS.ts)
