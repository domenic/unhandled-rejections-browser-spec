# Tests for Promise Rejection Events

These tests are written in [testharness.js](http://testthewebforward.org/docs/testharness-library.html) format, used by the [web-platform-tests](https://github.com/w3c/web-platform-tests) project.

They were ported from the original work by [@petkaantonov](https://github.com/petkaantonov) in [nodejs/io.js#758](https://github.com/nodejs/io.js/pull/758), as well as additions since then by others, up through [953b3e](https://github.com/nodejs/io.js/blob/953b3e75e899d99c43654280b2c2777f1364f0b0/test/parallel/test-promises-unhandled-rejections.js), and by [@jeisinger](https://codereview.chromium.org/1179113007).

promise-rejection-events-console.html contains a blink-ism to access messages logged to the console, but should also pass if the respective method is not present.
