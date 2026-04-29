# Receipt Parser — Frontend Offline Sync & State Recovery
This state machine illustrates the PWA's resilience. It handles network drops via IndexedDB and prevents data loss during accidental page refreshes via LocalStorage state snapshots.

```mermaid
flowchart TD
    Start((Start)) --> CaptureImage[Capture receipt image]
    CaptureImage --> CompressCanvas[Compress in browser canvas]
    CompressCanvas --> CheckNetwork{Network available?}

    CheckNetwork -->|No| OfflineQueue[Queue receipt locally]
    OfflineQueue --> IndexedDB[Persist base64 payload in IndexedDB]
    IndexedDB --> SyncLoop[Listen for browser online event]
    SyncLoop --> UploadBackend[Drain queue to backend]

    CheckNetwork -->|Yes| UploadBackend
    UploadBackend -->|HTTP 206: math validation failed| RenderForm

    subgraph HITL["Human correction and state recovery"]
        RenderForm[Render editable receipt form]
        RenderForm --> LocalStorage[Save parse memory snapshot]
        LocalStorage -->|Page refresh| RenderForm
        RenderForm --> ClientMathCalc[Recalculate taxes and totals]
        ClientMathCalc --> RenderForm
    end

    RenderForm -->|User clicks Save| SubmitCorrection[Submit correction]
    SubmitCorrection --> Done((HTTP 200 OK))
```
