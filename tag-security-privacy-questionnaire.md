# TAG Security & Privacy Review Document
Answering the questions from [the questionnaire doc here](https://www.w3.org/TR/security-privacy-questionnaire/).

## 3. Questions to Consider

### 3.1. Does this specification deal with personally-identifiable information?
No

### 3.2. Does this specification deal with high-value data?
No

### 3.3. Does this specification introduce new state for an origin that persists across browsing sessions?
No

### 3.4. Does this specification expose persistent, cross-origin state to the web?
No

### 3.5. Does this specification expose any other data to an origin that it doesn’t currently have access to?
No.  Right now, an origin can determine usage broken down by storage system by keeping track of the changes in total usage with each storage system operation.

Angle 1 (exposing new data): Per-system quota usage (IndexedDB vs Cache Storage vs AppCache etc.) is a function of all calls made by an origin to the respective storage APIs. The numbers summarize information that the origin already has. An origin can monitor the change in total quota with every storage API call to keep a running total.

Angle 2 (user identification/tracking): An origin that has data stored on the client (non-zero quota usage) can store a unique identifier for the user. Instead of using this new API, the origin can simply read a user ID from IndexedDB, or from Cache Storage etc. In other words, the new API does not make it any easier to identify or track users.

### 3.6. Does this specification enable new script execution/loading mechanisms?
No

### 3.7. Does this specification allow an origin access to a user’s location?
No

### 3.8. Does this specification allow an origin access to sensors on a user’s device?
No

### 3.9. Does this specification allow an origin access to aspects of a user’s local computing environment?
No

### 3.10. Does this specification allow an origin access to other devices?
No

### 3.11. Does this specification allow an origin some measure of control over a user agent’s native UI?
No

### 3.12. Does this specification expose temporary identifiers to the web?
No

### 3.13. Does this specification distinguish between behavior in first-party and third-party contexts?
No

### 3.14. How should this specification work in the context of a user agent’s "incognito" mode?
No difference, user agents may implement storage systems differently but quota reporting is not impacted.

### 3.15. Does this specification persist data to a user’s local device?
No

### 3.16. Does this specification have a "Security Considerations" and "Privacy Considerations" section?
No

### 3.17. Does this specification allow downgrading default security characteristics?
No
