# SignalR Real-Time Job Notification System

## 📋 Overview

This refactored system implements a **server-driven, real-time notification architecture** using **SignalR** for ASP.NET Web API job notifications from Mostaql.com.

### Key Features:
✅ **No Polling** - The browser extension only registers to the SignalR hub  
✅ **Server-Driven** - The server detects new jobs and pushes notifications  
✅ **Real-Time** - Instant notifications via WebSockets  
✅ **Preserved Logic** - Existing scraping and notification logic remains unchanged  
✅ **Backward Compatible** - Other job categories (AI, All) still work with the alarm system  

---

## 🏗️ Architecture

### Before (Polling-Based)
```
Browser Extension → Polls every 1 minute → Mostaql.com → Parse HTML → Detect New Jobs → Show Notification
```

### After (SignalR-Based)
```
ASP.NET Server → Scrapes Mostaql.com every 1 minute → Detects New ASP.NET Jobs
       ↓
   SignalR Hub
       ↓
Browser Extension (Connected) → Receives Job IDs → Fetches Full Details → Show Notification
```

---

## 📁 Project Structure

```
Frelancia/
├── SignalRServer/                    # ASP.NET Core Web API Server
│   ├── Program.cs                    # Server entry point
│   ├── Hubs/
│   │   └── JobNotificationHub.cs     # SignalR hub for real-time messaging
│   ├── Services/
│   │   ├── IJobScraperService.cs
│   │   └── JobScraperService.cs      # Background service that scrapes Mostaql
│   ├── Models/
│   │   └── JobListing.cs             # Job data model
│   ├── MostaqlJobNotifier.csproj     # Project file
│   └── appsettings.json
│
├── background.js                      # Extension service worker (modified)
├── signalr-client.js                 # SignalR client implementation (NEW)
├── signalr.min.js                    # SignalR library (to be downloaded)
├── manifest.json                      # Extension manifest (updated)
├── content.js                         # Content script (unchanged)
├── popup.js                           # Popup UI (unchanged)
└── offscreen.js                       # Offscreen document (unchanged)
```

---

## 🚀 Setup Instructions

### Step 1: Download SignalR Client Library

You need to download the Microsoft SignalR JavaScript client library:

**Option A: Direct Download**
1. Visit: https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.0/signalr.min.js
2. Save the file as `signalr.min.js` in the extension root directory

**Option B: Using npm**
```bash
npm install @microsoft/signalr
# Then copy node_modules/@microsoft/signalr/dist/browser/signalr.min.js to the extension root
```

### Step 2: Setup the SignalR Server

1. **Navigate to the server directory:**
   ```bash
   cd SignalRServer
   ```

2. **Restore dependencies:**
   ```bash
   dotnet restore
   ```

3. **Run the server:**
   ```bash
   dotnet run
   ```

   The server will start at:
   - HTTP: `http://localhost:5000`
   - HTTPS: `https://localhost:5001`

   You should see:
   ```
   info: MostaqlJobNotifier.Services.JobScraperService[0]
         Job Scraper Service started
   ```

### Step 3: Install/Reload the Extension

1. Open Chrome and navigate to: `chrome://extensions/`
2. Enable "Developer mode" (top right)
3. Click "Load unpacked"
4. Select the extension directory (Frelancia folder)
5. The extension will load and automatically connect to SignalR

### Step 4: Verify the Connection

1. Open the extension popup
2. Check the browser console (F12) → Background page
3. You should see:
   ```
   SignalR: Initializing connection to http://localhost:5000/jobNotificationHub
   SignalR: Connected successfully
   SignalR: Connection confirmed
   ```

---

## 🔧 How It Works

### Server-Side Flow (JobScraperService.cs)

1. **Background Service Starts** - Runs every 1 minute
2. **Fetches Mostaql.com** - Gets the latest development jobs
3. **Filters ASP.NET Jobs** - Looks for keywords: "asp.net", "web api", ".net core", "c#", etc.
4. **Detects New Jobs** - Compares against previously seen job IDs
5. **Sends SignalR Notification** - Broadcasts to all connected clients:
   ```javascript
   {
     timestamp: "2026-02-17T10:30:00Z",
     count: 2,
     jobs: [
       { id: "12345", title: "تطوير Web API بـ ASP.NET", budget: "$500", url: "..." },
       { id: "12346", title: "برمجة نظام بـ C#", budget: "$800", url: "..." }
     ]
   }
   ```

### Client-Side Flow (signalr-client.js)

1. **Extension Starts** - Automatically connects to SignalR hub
2. **Listens for Events** - Waits for `NewJobsDetected` event
3. **Receives Notification** - Gets job data from server
4. **Fetches Full Details** - Scrapes each job page for complete information (description, hiring rate, etc.)
   - **This is where the scraping happens AFTER the request!** ✅
5. **Applies Filters** - Uses existing filter logic (budget, keywords, etc.)
6. **Shows Notification** - Uses the existing `showNotification()` function (unchanged) ✅

---

## 🔍 Key Requirements Met

### ✅ Requirement 1: Move Page Loading Logic
**"Relocate the existing page loading and scraping logic so that it executes after the request is completed."**

**Implementation:**
- The server sends only basic job info (ID, title, budget, URL)
- The extension receives the notification
- **Then** the extension calls `fetchProjectDetails(job.url)` for each job
- This scrapes the full page content AFTER receiving the SignalR notification

See [signalr-client.js](signalr-client.js#L157-L175):
```javascript
// Fetch project details (this is the "scraping after request" requirement)
const projectDetails = await fetchProjectDetails(job.url);
```

### ✅ Requirement 2: Detect New ASP.NET Web API Jobs
**"The system should continuously check for newly added jobs related to ASP.NET Web API."**

**Implementation:**
- The server's `IsAspNetWebApiJob()` method filters jobs
- Keywords: `asp.net`, `web api`, `.net core`, `c#`, `aspnet`, etc.
- Only new jobs (not in `_seenJobIds`) are sent

See [JobScraperService.cs](SignalRServer/Services/JobScraperService.cs#L88-L104)

### ✅ Requirement 3: Real-Time Notification Using SignalR
**"The browser extension must only register (connect) to the SignalR hub. Not send any polling or repeated HTTP requests."**

**Implementation:**
- Extension connects once on startup: `await signalRClient.connect()`
- No polling loops for ASP.NET jobs
- Server pushes notifications via `NewJobsDetected` event
- Automatic reconnection if connection drops

See [signalr-client.js](signalr-client.js#L21-L55)

### ✅ Requirement 4: Existing Notification Logic Unchanged
**"The notification system should continue working without requiring any changes to the existing notification logic."**

**Implementation:**
- The same `showNotification()` function is called
- The same `playSound()` function is used
- The same `applyFilters()` function works
- The same storage mechanism (`seenJobs`, `recentJobs`, `stats`) is preserved

See [signalr-client.js](signalr-client.js#L195-L202)

---

## 🎛️ Configuration

### Server Configuration

Edit `SignalRServer/appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MostaqlJobNotifier.Services.JobScraperService": "Debug"
    }
  },
  "Urls": "http://localhost:5000;https://localhost:5001"
}
```

To change the server port, modify the `Urls` setting.

### Extension Configuration

Edit `signalr-client.js`:

```javascript
this.serverUrl = 'http://localhost:5000/jobNotificationHub';
```

Change this URL if you deploy the server elsewhere.

You can also disable SignalR in the extension settings:
```javascript
// In popup or settings
chrome.storage.local.set({ settings: { signalREnabled: false } });
```

---

## 🧪 Testing

### Test SignalR Connection

1. **Start the server:**
   ```bash
   cd SignalRServer
   dotnet run
   ```

2. **Watch server logs:**
   You should see:
   ```
   info: MostaqlJobNotifier.Hubs.JobNotificationHub[0]
         Client connected: <connection-id>
   ```

3. **Check extension console:**
   ```
   SignalR: Connected successfully
   ```

### Simulate a New Job Notification

You can manually test the notification by temporarily modifying `JobScraperService.cs` to always send a test job:

```csharp
// In CheckForNewJobsAsync(), after fetching jobs
var testJob = new JobListing
{
    Id = "test-" + DateTime.Now.Ticks,
    Title = "اختبار - تطوير Web API بـ ASP.NET Core",
    Budget = "$500 - $800",
    Url = "https://mostaql.com/projects"
};

await NotifyClientsAsync(new List<JobListing> { testJob });
```

---

## 📊 Performance & Scaling

### Current Configuration
- **Server Poll Interval**: 1 minute
- **Extension**: No polling (event-driven)
- **Max Stored Job IDs**: 500 (both server and client)
- **Reconnect Strategy**: Exponential backoff (2s, 10s, 30s, 60s max)

### Scaling Considerations

If you need to support multiple users:

1. **Deploy Server to Cloud**: Azure App Service, AWS EC2, or Heroku
2. **Use Redis Backplane**: For SignalR scalability across multiple servers
3. **Add Authentication**: Secure the SignalR hub with JWT tokens
4. **Database Integration**: Store job history in SQL Server or MongoDB

---

## 🛠️ Troubleshooting

### "SignalR client not available"
- Ensure `signalr.min.js` is downloaded and placed in the extension root
- Check browser console for script loading errors
- Verify `importScripts` in `background.js` succeeded

### "Connection failed" or "NET::ERR_CONNECTION_REFUSED"
- Make sure the server is running: `dotnet run`
- Check the server URL matches in `signalr-client.js`
- Verify `manifest.json` has localhost permissions

### "No new jobs detected"
- The server only detects ASP.NET-related jobs
- Jobs must be new (not previously seen)
- Check server logs for scraping errors:
  ```bash
  dotnet run --verbosity detailed
  ```

### Extension doesn't connect on startup
- Service workers in Chrome can be suspended
- SignalR will auto-reconnect when the service worker wakes up
- You can manually trigger a connection check from the popup

---

## 🔄 Migration from Old System

The refactored system is **backward compatible**:

- ✅ Other job categories (AI, All) still use the alarm-based polling
- ✅ Tracked projects still work with periodic checks
- ✅ All existing settings, filters, and UI remain unchanged
- ✅ You can disable SignalR and fall back to the old system

To disable SignalR:
```javascript
// In background.js
const SIGNALR_ENABLED = false;
```

---

## 📝 Code Changes Summary

### Files Modified:
1. **manifest.json** - Added localhost permissions, SignalR library references
2. **background.js** - Added SignalR initialization, imports SignalR client

### Files Created:
1. **signalr-client.js** - SignalR client logic
2. **SignalRServer/** - Complete ASP.NET Core server project
3. **SIGNALR_SETUP.js** - Setup instructions for SignalR library

### Files Unchanged:
- ✅ **content.js** - No changes
- ✅ **popup.js** - No changes (can add SignalR status UI later)
- ✅ **offscreen.js** - No changes
- ✅ **chatgpt.js** - No changes

---

## 🎯 Next Steps & Future Enhancements

### Immediate Next Steps:
1. Download `signalr.min.js` library
2. Start the SignalR server
3. Test the extension with a new ASP.NET job post

### Future Enhancements:
- [ ] Add SignalR connection status indicator in popup UI
- [ ] Support custom server URLs in settings
- [ ] Add job filtering on the server side to reduce client processing
- [ ] Implement user authentication for secure connections
- [ ] Add job categories as server-side parameters
- [ ] Deploy server to Azure for production use
- [ ] Add database persistence for job history
- [ ] Implement push notifications for mobile devices

---

## 📞 Support & Documentation

### Key Files to Reference:
- **Architecture**: This file (SIGNALR_GUIDE.md)
- **Server Setup**: SignalRServer/Program.cs
- **Client Logic**: signalr-client.js
- **API Reference**: https://docs.microsoft.com/en-us/aspnet/core/signalr/

### Debugging Tips:
1. **Server Logs**: Watch the console output from `dotnet run`
2. **Extension Logs**: Check the service worker console in `chrome://extensions/`
3. **Network Tab**: Monitor WebSocket connections in DevTools
4. **SignalR Tracing**: Enable detailed logging in `appsettings.json`

---

## 📄 License

This project maintains the same license as the original extension.

---

**Last Updated**: February 17, 2026  
**Version**: 2.0.0 (SignalR Integration)
