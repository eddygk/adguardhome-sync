# AdGuardHome Sync Progress Indicator Implementation Plan

## Overview
Add a lightweight, real-time progress indicator to the AdGuardHome Sync web interface to display:
- Time since last sync
- Time until next sync
- Current sync status
- Visual progress bar

## Design Principles
- **Lightweight**: No external UI libraries (no shadcn, Material-UI, etc.)
- **Terminal Aesthetic**: Match the existing dark terminal-style interface
- **Minimal Dependencies**: Use existing jQuery and Bootstrap already in the project
- **Real-time Updates**: Live countdown without page refresh
- **Non-intrusive**: Fits naturally into the current UI layout

## Architecture Changes

### Backend (Go) Modifications

#### 1. Worker State Tracking (`internal/sync/sync.go`)
```go
type worker struct {
    // Existing fields...
    lastSyncTime time.Time       // Track when last sync completed
    nextSyncTime time.Time       // Track when next sync is scheduled
    cronEntryID  cron.EntryID    // Store cron entry ID for schedule inspection
}
```

#### 2. Enhanced Status API (`internal/sync/http.go`)
**Endpoint: `GET /api/v1/sync-schedule`**
```go
type syncSchedule struct {
    LastSyncTime    time.Time `json:"lastSyncTime"`
    NextSyncTime    time.Time `json:"nextSyncTime"`
    SyncRunning     bool      `json:"syncRunning"`
    CronExpression  string    `json:"cronExpression,omitempty"`
    IntervalSeconds int       `json:"intervalSeconds"`
}
```

#### 3. Update Sync Methods
- **sync()**: Update `lastSyncTime` when sync completes
- **cron setup**: Store `cronEntryID` and calculate `nextSyncTime`
- **status()**: Include sync schedule information

### Frontend (HTML/JavaScript) Modifications

#### 1. HTML Structure (`internal/sync/static/index.html`)
Add progress indicator between control buttons and logs:

```html
<!-- After button-row div -->
<div class="row sync-progress-row" id="syncProgressRow" style="display: none;">
    <div class="col-12">
        <div class="sync-progress-card">
            <div class="progress-header">
                <div class="progress-info">
                    <span class="sync-status">
                        <span class="status-dot"></span>
                        <span id="syncStatusText">Idle</span>
                    </span>
                    <span class="sync-timing">
                        Last sync: <span id="lastSyncTime">--</span> ago |
                        Next sync: in <span id="nextSyncTime">--</span>
                    </span>
                </div>
            </div>
            <div class="progress-bar-container">
                <div class="progress-bar-fill" id="progressBarFill"></div>
            </div>
        </div>
    </div>
</div>
```

#### 2. CSS Styling
Terminal-style dark theme matching existing aesthetic:

```css
.sync-progress-card {
    background: rgba(0, 0, 0, 0.3);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 8px;
    padding: 12px;
    margin: 16px 0;
    font-family: monospace;
    font-size: 13px;
}

.progress-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
}

.status-dot {
    display: inline-block;
    width: 8px;
    height: 8px;
    border-radius: 50%;
    margin-right: 6px;
    background: #10b981; /* Green for idle */
}

.status-dot.syncing {
    background: #3b82f6; /* Blue for syncing */
    animation: pulse 1s infinite;
}

@keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.4; }
}

.sync-timing {
    color: #9ca3af;
    font-size: 12px;
}

.progress-bar-container {
    height: 4px;
    background: rgba(255, 255, 255, 0.1);
    border-radius: 2px;
    overflow: hidden;
}

.progress-bar-fill {
    height: 100%;
    background: linear-gradient(90deg, #10b981, #22c55e);
    transition: width 1s linear;
    width: 0%;
}
```

#### 3. JavaScript Logic
```javascript
// Global variables for sync tracking
let syncScheduleData = null;
let progressUpdateInterval = null;

// Fetch sync schedule from API
function fetchSyncSchedule() {
    $.get("api/v1/sync-schedule", function(data) {
        syncScheduleData = data;
        if (data.intervalSeconds > 0) {
            $("#syncProgressRow").show();
            updateProgressDisplay();
        }
    });
}

// Update progress display every second
function updateProgressDisplay() {
    if (!syncScheduleData) return;

    const now = Date.now();
    const lastSync = new Date(syncScheduleData.lastSyncTime).getTime();
    const nextSync = new Date(syncScheduleData.nextSyncTime).getTime();

    // Calculate times
    const timeSinceSync = Math.floor((now - lastSync) / 1000);
    const timeUntilSync = Math.max(0, Math.floor((nextSync - now) / 1000));
    const totalInterval = syncScheduleData.intervalSeconds;

    // Update text
    $("#lastSyncTime").text(formatDuration(timeSinceSync));
    $("#nextSyncTime").text(formatDuration(timeUntilSync));

    // Update progress bar
    const progress = Math.min(100, (timeSinceSync / totalInterval) * 100);
    $("#progressBarFill").css("width", progress + "%");

    // Update status
    if (syncScheduleData.syncRunning) {
        $("#syncStatusText").text("Syncing...");
        $(".status-dot").addClass("syncing");
    } else {
        $("#syncStatusText").text("Idle");
        $(".status-dot").removeClass("syncing");
    }
}

// Format duration helper
function formatDuration(seconds) {
    if (seconds < 60) return seconds + "s";
    if (seconds < 3600) return Math.floor(seconds / 60) + "m " + (seconds % 60) + "s";
    return Math.floor(seconds / 3600) + "h " + Math.floor((seconds % 3600) / 60) + "m";
}

// Initialize on document ready
$(document).ready(function() {
    // Existing code...

    // Fetch initial sync schedule
    fetchSyncSchedule();

    // Update progress every second
    progressUpdateInterval = setInterval(function() {
        updateProgressDisplay();
    }, 1000);

    // Refresh schedule data every 30 seconds
    setInterval(fetchSyncSchedule, 30000);

    // Update schedule after manual sync
    $("#sync").click(function() {
        // Existing sync code...
        setTimeout(fetchSyncSchedule, 2000); // Refresh after sync starts
    });
});
```

## Implementation Steps

### Phase 1: Backend Foundation
1. Add sync time tracking fields to worker struct
2. Implement `/api/v1/sync-schedule` endpoint
3. Update sync() to record completion time
4. Calculate next sync time from cron schedule
5. Add unit tests for new functionality

### Phase 2: Frontend UI
1. Add HTML structure for progress indicator
2. Implement CSS styling matching dark theme
3. Add JavaScript for fetching schedule data
4. Implement real-time countdown logic
5. Test visual appearance in both light/dark modes

### Phase 3: Integration & Polish
1. Wire up progress updates with existing sync button
2. Handle edge cases (no cron, one-time sync)
3. Add smooth transitions and animations
4. Test with various cron expressions
5. Optimize API polling frequency

## Technical Considerations

### Cron Schedule Parsing
- Use existing `robfig/cron/v3` package capabilities
- Access cron.Entry.Schedule.Next() for accurate next run time
- Handle cases where cron is not configured

### Time Synchronization
- Use server time for all calculations
- Send timestamps in ISO 8601 format
- Handle timezone differences gracefully

### Performance
- Minimize API calls (poll every 30s, update UI every 1s)
- Use CSS transitions for smooth progress bar
- Debounce rapid sync button clicks

### Error Handling
- Gracefully handle missing cron configuration
- Show appropriate message for one-time sync mode
- Handle network errors in API calls

## Testing Strategy

### Unit Tests
- Test sync schedule calculation logic
- Test duration formatting functions
- Test progress percentage calculations

### Integration Tests
- Test API endpoint responses
- Test cron schedule updates
- Test sync completion tracking

### UI Tests
- Visual regression testing for light/dark themes
- Test progress bar animation smoothness
- Test countdown accuracy
- Test responsive layout on different screen sizes

## Alternative Approaches Considered

### Option 1: Server-Sent Events (SSE)
- **Pros**: Real-time updates, no polling
- **Cons**: More complex, requires WebSocket-like connection
- **Decision**: Not needed for 1-second precision

### Option 2: WebSocket Connection
- **Pros**: Bidirectional real-time communication
- **Cons**: Overkill for this use case, adds complexity
- **Decision**: Polling is sufficient

### Option 3: Badge-Style Indicator
- **Pros**: More compact, less visual space
- **Cons**: Less informative, harder to see progress
- **Decision**: Progress bar provides better UX

## Success Criteria
1. ✅ Users can see time since last sync
2. ✅ Users can see countdown to next sync
3. ✅ Visual progress bar shows sync cycle progress
4. ✅ No external UI libraries added
5. ✅ Matches existing terminal aesthetic
6. ✅ Works with both cron and manual sync modes
7. ✅ Performs well (minimal CPU/memory impact)
8. ✅ Responsive on mobile devices

## Rollback Plan
If issues arise:
1. Feature can be disabled via config flag
2. UI gracefully degrades if API unavailable
3. Existing functionality remains unchanged
4. Easy removal via feature toggle

## Future Enhancements
- Sync history graph (last 24h)
- Estimated sync duration based on history
- Alert when sync fails multiple times
- Sync statistics (success rate, average duration)
- Click progress bar to show detailed sync log