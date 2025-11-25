# iOS Universal Links Complete Debugging Guide

If you're struggling with iOS Universal Links not working in your app, this comprehensive guide covers all possible issues and debugging techniques, including advanced sysdiagnose methods.

## Common Universal Link Issues

### 1. Link Not Opening Your App

When tapping a Universal Link doesn't open your application, follow this validation process:

**Validation Step:**
1. Copy your Universal Link
2. Paste it into the Notes app on your device/simulator
3. Long press the link
4. Observe the options presented

#### Scenario A: App Displayed in Notes Menu ✅
If your app appears in the long-press menu, your configuration is correct. The issue is likely device cache-related.

**Fix Options:**
1. **Clear Safari Website Data:**
   - Open Settings app
   - Scroll to Safari
   - Tap "Clear History and Website Data"
   - Confirm by tapping "Clear History and Data"
   - ⚠️ Note: This clears all browsing history, cookies, and cache

2. **Reinstall Your App:**
   - Uninstall the app completely
   - Reinstall from App Store/TestFlight

3. **Device Restart:**
   - Restart your iOS device

4. **Manual App Selection:**
   - Select your app directly from the Notes long-press menu

#### Scenario B: App NOT Displayed in Notes Menu ❌
If your app doesn't appear, there's a configuration issue. Follow this debugging workflow:

**Step 1: Validate Mobile Setup**
- Open Settings app
- Scroll to Developer (enable Developer mode if needed)
- Tap "Universal Links"
- Select "Diagnostics"
- Paste your Universal Link and tap "GO"
- Check if it shows "Opens Installed Application" with your bundle ID

**If successful → Issue is server-side (proceed to Step 3)**
**If failed → Issue is app-side (proceed to Step 2)**

**Step 2: Debug App Configuration**
- Open your Xcode project
- Select your app target
- Navigate to "Signing & Capabilities" tab
- Verify:
  - Associated Domains capability is added
  - Domain URL format: `applinks:yourdomain.com`
  - Domain matches your Universal Link exactly (e.g., if link is `https://example.com/path`, use `applinks:example.com`)
- Uninstall and reinstall app after fixes
- Retest with Notes app validation

**Step 3: Debug AASA (Apple App Site Association) File**
1. **Access AASA File:**
   - Open in device browser: `https://yourdomain.com/.well-known/apple-app-site-association`
   - File should download automatically

2. **Verify No Redirects:**
   ```bash
   curl -v https://yourdomain.com/.well-known/apple-app-site-association
   ```
   - ❌ 301/302 status codes indicate redirects (not supported)
   - ✅ 200 status code indicates proper hosting

3. **Fix Hosting Issues:**
   - Ensure AASA file is hosted at exact path: `/.well-known/apple-app-site-association`
   - No redirects allowed
   - File must be served with `application/json` content type
   - After fixes, uninstall/reinstall app and retest

**Step 4: Debug Server Restrictions**
If previous steps fail, check for server restrictions:

1. **Enable Developer Mode:**
   - In Xcode Associated Domains, add: `applinks:yourdomain.com?mode=developer`
   - On device: Settings → Developer → Enable "Associated Domains Development"
   - Now uninstall the app and reinstall it and Retest with Notes app validation
   - If it is working normally that confirm that your AASA file is not reachable outside of your network.

2. **Test Network Restrictions:**
   - Geographic restrictions
   - Corporate network limitations
   - Firewall rules
   - VPN requirements

3. **Verify Server Accessibility:**
   - Test from different networks
   - Check if server is publicly accessible
   - Verify SSL certificate validity

### 2. App Launches But Doesn't Receive Universal Link

When your app opens but doesn't receive the Universal Link data, follow these debugging steps:

#### Step 1: Verify App Delegate Methods
Ensure you're implementing the correct methods in your `AppDelegate` or `SceneDelegate`:

**For AppDelegate (iOS 12 and earlier):**
```swift
func application(_ application: UIApplication, 
                 continue userActivity: NSUserActivity,
                 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
    
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let incomingURL = userActivity.webpageURL else {
        return false
    }
    
    // Handle the Universal Link
    print("Received Universal Link: \(incomingURL)")
    return true
}
```

**For SceneDelegate (iOS 13 and later):**
```swift
func scene(_ scene: UIScene, 
           willConnectTo session: UISceneSession, 
           options connectionOptions: UIScene.ConnectionOptions) {
    
    if let userActivity = connectionOptions.userActivities.first {
        handleUserActivity(userActivity)
    }
}

func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
    handleUserActivity(userActivity)
}

private func handleUserActivity(_ userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let incomingURL = userActivity.webpageURL else {
        return
    }
    
    // Handle the Universal Link
    print("Received Universal Link: \(incomingURL)")
}
```

#### Step 2: Check AASA File Configuration
Verify your AASA file includes correct paths and your app's bundle ID:

```json
{
    "applinks": {
        "apps": [],
        "details": [
            {
                "appID": "TEAMID.com.yourapp.bundle",
                "paths": ["*", "/"]
            }
        ]
    }
}
```

**Common AASA Issues:**
- Incorrect team ID or bundle identifier
- Missing or restrictive paths array
- Invalid JSON format
- Wrong MIME type (should be `application/json`)

#### Step 3: Test URL Handling
Add debugging to verify the URL is being received:

```swift
private func handleUserActivity(_ userActivity: NSUserActivity) {
    print("User Activity Type: \(userActivity.activityType)")
    
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let incomingURL = userActivity.webpageURL else {
        print("Not a web browsing activity or missing URL")
        return
    }
    
    print("Universal Link URL: \(incomingURL)")
    print("URL Components: \(incomingURL.path)")
    print("Query Parameters: \(incomingURL.query ?? "None")")
    
    // Add your URL routing logic here
}
```

### 3. Advanced Debugging with Sysdiagnose

When all other debugging methods fail, sysdiagnose provides detailed system-level logs for comprehensive troubleshooting.

#### Generating Sysdiagnose Logs

**Method 1: Physical Device Trigger**
1. **Enable Developer Options:**
   - Go to Settings → Privacy & Security → Analytics & Improvements
   - Ensure "Share With App Developers" is enabled

2. **Trigger Sysdiagnose:**
   - **iPhone 14 Pro or later:** Press Volume Up + Volume Down + Lock buttons simultaneously
   - **Older iPhones with Home button:** Press Home + Lock + Volume Up buttons
   - **All devices:** Hold buttons for ~1 second until haptic feedback
   - The screen will flash briefly indicating capture started

3. **Access Logs:**
   - Wait 5-10 minutes for log processing
   - Go to Settings → Privacy & Security → Analytics & Improvements → Analytics Data
   - Look for files starting with "sysdiagnose_" with timestamp
   - Share this file to your MAC and un zip it.
   - Search for "swcutil_show.txt" file and open it in text editor.
   - In the text editor search for your app bundle ID. You will find it as bellow.
     <img width="801" height="501" alt="Screenshot 2025-11-24 at 12 10 05 PM" src="https://github.com/user-attachments/assets/f8e1caf1-d324-4c8e-b7a4-3abc67e783b7" />


**Method 2: macOS Console App (Real-time)**
1. Connect iOS device to Mac
2. Open Console app on macOS
3. Select your connected device
4. Filter logs with:
   - `swcd` (Sharing Web Credentials Daemon - handles Universal Links)
   - `apsd` (Apple Push Services Daemon)
   - `nsurld` (NSURLSession Daemon)
   - Your app's bundle identifier

#### Analyzing Sysdiagnose Logs

**Key Log Areas to Examine:**

1. **swcd Daemon Logs:**
   ```
   filter:process == "swcd"
   search: "universal links", "applinks", "AASA", "associated domains"
   ```

2. **AASA Validation Logs:**
   Look for entries like:
   ```
   swcd: [AASA] Fetching AASA for domain: yourdomain.com
   swcd: [AASA] Validation result for yourdomain.com
   swcd: [AASA] Registered apps: [{"appID":"TEAMID.com.yourapp.bundle","paths":["*"]}]
   ```

3. **Universal Link Routing:**
   ```
   swcd: [UniversalLinks] Determining application for URL: https://yourdomain.com/path
   swcd: [UniversalLinks] Routing decision: opening app with bundleID com.yourapp.bundle
   ```

**Common Sysdiagnose Findings:**

1. **AASA Fetch Failures:**
   ```
   swcd: [AASA] Failed to fetch AASA for domain: yourdomain.com, error: NSURLErrorDomain -1003
   ```
   *Indicates DNS or network connectivity issues*

2. **Validation Errors:**
   ```
   swcd: [AASA] Invalid JSON format for domain: yourdomain.com
   swcd: [AASA] AppID mismatch: expected TEAMID.com.yourapp.bundle, found different
   ```
   *Indicates AASA file format issues*

3. **Entitlement Mismatches:**
   ```
   swcd: [AssociatedDomains] Entitlement mismatch for bundleID com.yourapp.bundle
   ```
   *Indicates app configuration issues*

#### Command Line Analysis (Advanced)

Extract and analyze sysdiagnose archives:

```bash
# Extract sysdiagnose archive
tar -xzf sysdiagnose_YYYY.MM.DD_HH-MM-SS-XXX.tar.gz

# Search for Universal Link related logs
grep -r "universal links" ./sysdiagnose/
grep -r "swcd" ./sysdiagnose/
grep -r "applinks" ./sysdiagnose/

# Check AASA validation logs
find . -name "*.log" -exec grep -l "AASA" {} \;
```

### 4. Comprehensive Troubleshooting Checklist

- [ ] **SSL Certificate:** Valid and trusted certificate
- [ ] **AASA File:** Accessible without redirects at correct path
- [ ] **Content-Type:** `application/json` for AASA file
- [ ] **App Entitlements:** Associated Domains properly configured
- [ ] **Bundle Identifier:** Matches exactly in AASA file and Xcode
- [ ] **Team ID:** Correct in AASA file (`TEAMID.bundle.id`)
- [ ] **Paths Array:** Includes desired URL patterns (`["*"]` for all paths)
- [ ] **Device Testing:** Tested on physical device, not just simulator
- [ ] **Network Conditions:** Works on different networks (cellular/WiFi)
- [ ] **App Installation:** Fresh install after configuration changes

### 5. Quick Debugging Commands

**Server-side AASA Verification:**
```bash
# Check AASA accessibility
curl -I https://yourdomain.com/.well-known/apple-app-site-association

# Download and validate AASA content
curl -s https://yourdomain.com/.well-known/apple-app-site-association | python -m json.tool

# Check for redirects
curl -L -w "%{url_effective}" https://yourdomain.com/.well-known/apple-app-site-association -o /dev/null
```

**Client-side Debugging:**
- Use Safari Developer Tools on macOS to inspect network requests
- Enable `Associated Domains Development` mode for real-time debugging
- Monitor `swcd` logs in Console app during Universal Link testing

By systematically following this guide—from basic configuration checks to advanced sysdiagnose analysis—you can identify and resolve even the most complex Universal Link issues in your iOS application.
