# Model Diff UI - HTML Edition

## Overview

This project is an HTML instantiation of Gerhard's [model_diff_ui](https://github.com/Istari-digital/model_diff_u) PySide6 desktop app. It is designed to bridge the platform gap of 1-file-input-per-job, and to run inside the Istari platform's File Preview pane to allow users seamless access.

## Current State

(10/14/2026) All model_diff_ui functionality has been replicated and tested, with the exception of:
1. No user prompt capability
    - *Reason*: The user prompt is passed as a param to the function, which means it must be uploaded to S3, then referenced by pointer when passed to the function. The browser restricts the File Preview iframe's opaque origin (which is null) from performing S3 uploads due to S3's CORS policies.

2. End-to-end execution not tested yet
    - *Reason*: The app has been tested from boot -> function execution. The function execution is currently failing in demo due to import failures, which is the same behavior I am experiencing in the PySide6 app. Will need to update the smart_diff script to be built with the required deps to complete testing, although remaining end-to-end behavior is out of this app's hands and is largely scoped to the module.

## Enhancements

1. The File Registry/API URLs can be inferred from inside the iframe, eliminating the need for users to fill in an additional input field. Currently auto-detecting/-filling this in for users.

## Next Steps

1. Refactor/clean up Javascript.
2. Consider migrating to TS client.
3. State persistence across parent page renders.