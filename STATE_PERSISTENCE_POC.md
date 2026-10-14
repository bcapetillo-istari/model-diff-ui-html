# State Persistence in In-Istari HTML "Apps"

## Problem

All application state is tracked in DOM and/or memory. A page refresh erases all state, making this very volatile.

## Solution

Use a light-weight JSON file uploaded/persisted to Istari that the app will read from and write to. This should represent last application state, NOT maintain an entire history of all prior states. If the file or JSON is empty, that means no state exists. HTML Apps are responsible for managing their state files.

Provisioning/linking these state files is expected to be a manual process, and should be per-app, per-user. Could potentially move to a master state file per-app tracking all users' states (each user's state keyed by user_id). Linking to the application's state file will have to be hardcoded in the HTML, as that is the only way to persist the relationship to this file. 

## Benefit

Maintaining HTML-app state expands on the capabilities of this new application type, furthering the FDEs' abilities to write last-mile UI capabilities (e.g. dual-file input for jobs, which is currently only possible via SDK. Smart Diff UI allows dual-file inputs in a UI-forward way). This ability to bridge UI gaps in order to reduce time-to-value for customers is especially important for UI-users in heavily restricted environments (e.g. Model One)

# Blocker (10/14/2026)

## Problem

Resource read/writes are done through direct S3 connection via a presigned URL-- file bytes do NOT pass through the Registry service. This blocks the ability to write/read state from Istari, because the browser blocks in-browser S3 reads/writes due to S3's CORS policies not allowing the iframe's opaque origin. This is a problem solely in browser-executed code due to CORS. Resource read/writes execute flawlessly outside the browser.

## Proposed Solution

Expose a minimal message-passing API on the host page that responds to postMessage requests from the sandboxed File Preview iframe. Three operations are all that are required to allow HTML apps to read/write data and perform authorized API actions via the TS SDK, greatly improving current capability:

1. *Read a resource revision*: The iframe sends a resource ID. The host, running in its real CORS-allowlisted origin, fetches that resource's current content (from the object store) and posts the content back to the iframe. Can be performed via API or SDK.

2. *Write a resource revision*: The iframe sends a resource ID + content. The host, from that same allowlisted origin, makes the create-revision call to store the content as a new revision of that resource (content + properties to the object store, then registers the revision) and posts back the new revision ID to the iframe. Can be performed via API or SDK.

3. *Provide a scoped, short-lived access token*: The host mints or relays a scoped, time-limited bearer token to the iframe so the app can make its own API/SDK calls directly. Should be refreshable via the same API. Could potentially relay the user's existing session token to allow/restrict the app to only the authorizations/accesses of the logged-in user. 
