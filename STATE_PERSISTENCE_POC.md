# State Persistence in In-Istari HTML "Apps"

## Problem

All application state is tracked in DOM and/or memory. A page refresh erases all state, making this very volatile.

## Solution

Use a light-weight JSON file uploaded/persisted to Istari that the app will read from and write to. This should represent last application state, NOT maintain an entire history of all prior states. If the file or JSON is empty, that means no state exists. HTML Apps are responsible for managing their state files.

Provisioning/linking these state files is expected to be a manual process, and should be per-app, per-user. Could potentially move to a master state file per-app tracking all users' states (each user's state keyed by user_id). Linking to the application's state file will have to be hardcoded in the HTML, as that is the only way to persist the relationship to this file. 

## Benefit

Maintaining HTML-app state expands on the capabilities of this new application type, furthering the FDEs' abilities to write last-mile UI capabilities (e.g. dual-file input for jobs, which is currently only possible via SDK. Smart Diff UI allows dual-file inputs in a UI-forward way). This ability to bridge UI gaps in order to reduce time-to-value for customers is especially important for UI-users in heavily restricted environments (e.g. Model One)
