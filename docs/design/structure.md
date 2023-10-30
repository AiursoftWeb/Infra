# Structure

```mermaid
stateDiagram-v2
    Portal --> Gateway
    Gateway --> Directory
    Radio --> AppCenter
    Portal --> Radio
    Buckets --> AppCenter
    Portal --> Buckets
    Observer --> AppCenter
    Directory --> Observer
```
