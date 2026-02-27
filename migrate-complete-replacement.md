# AI Agent Instructions: Complete MaxSdk → MeticaSdk Migration

You are a Unity C# migration assistant. Your task is to migrate a Unity project from **AppLovin MaxSdk 8.6.0** to **MeticaSdk 2.2.2**. MeticaSdk wraps AppLovin MAX internally — the MAX Unity Plugin stays installed, but all direct `MaxSdk.*` code must be replaced with `MeticaSdk.*` equivalents.

Follow every step below in order. Do not skip steps. After each step, confirm what you changed.

---

## Critical Rules (Read Before Every Step)

1. **KEEP** the AppLovin MAX Unity Plugin installed — MeticaSdk depends on it internally
2. **NEVER** call `MaxSdk.*`, `MaxSdkCallbacks.*`, or `MaxSdkBase.*` directly — MeticaSdk handles MAX under the hood
3. **Reuse existing ad unit IDs** — they work unchanged with MeticaSdk
4. **Move MAX SDK Key** into `MeticaMediationInfo` constructor (no more `MaxSdk.SetSdkKey()`)
5. **Call privacy methods BEFORE initialization** — `SetHasUserConsent` and `SetDoNotSell` must precede `MeticaSdk.Initialize()`
6. **Casing change for MREC**: MaxSdk uses `MRec` (capital R), MeticaSdk uses `Mrec` (lowercase r)
7. **Rewarded method names drop "Ad"**: `LoadRewardedAd` → `LoadRewarded`, `IsRewardedAdReady` → `IsRewardedReady`, `ShowRewardedAd` → `ShowRewarded`
8. **Callback signatures are simplified**: MaxSdk passes `(string adUnitId, AdInfo adInfo)` as separate params → MeticaSdk passes a single `MeticaAd` record containing all fields

---

## Step 0: Verify MaxSdk Version (REQUIRED)

MeticaSdk 2.2.1 requires **AppLovin MAX Unity Plugin v8.2.0 or later**. Before making any changes, verify the installed version.

### How to check:
1. Search the project for the MaxSdk version property or changelog:
   ```
   Search for: MaxSdk.Version
   Search for files: CHANGELOG.md or package.json inside the MaxSdk or AppLovin plugin folder
   ```
2. Alternatively, check `MaxSdk.Version` at runtime: `Debug.Log("MAX SDK version: " + MaxSdk.Version);`

### If version is below 8.2.0:
**STOP.** Do not proceed with migration. The developer must update the AppLovin MAX Unity Plugin to v8.2.0+ first. Inform them:
```
// ERROR: AppLovin MAX Unity Plugin version [detected version] is below the minimum
// required v8.2.0. Please update the MAX plugin before migrating to MeticaSdk.
// Download from: https://github.com/AppLovin/AppLovin-MAX-Unity-Plugin/releases
```

### If version is 8.2.0 or later:
Proceed to Step 1.

---

## Step 1: Find All MaxSdk References

Search the entire project for files that reference MaxSdk. Use these patterns:

```
Regex patterns to search:
  MaxSdk\.
  MaxSdkCallbacks\.
  MaxSdkBase\.
  using\s+MaxSdk
  MaxSdk\.SetSdkKey
  MaxSdk\.InitializeSdk
```

List every file found. These are the files you will modify in subsequent steps.

---

## Step 2: Update Namespace Imports

In every file found in Step 1, update the `using` directives.

**Remove** any of these:
```csharp
using MaxSdk;
// and any references to MaxSdkBase, MaxSdkCallbacks namespaces
```

**Add** these at the top of each file:
```csharp
using Metica;
using Metica.Ads;
```

---

## Step 3: Replace SDK Initialization

Find the existing MaxSdk initialization code and replace it entirely.

### Search for (old patterns):
```
MaxSdk.SetSdkKey(
MaxSdk.InitializeSdk(
MaxSdkCallbacks.OnSdkInitializedEvent
MaxSdk.SetHasUserConsent(
MaxSdk.SetDoNotSell(
MaxSdk.SetVerboseLogging(
```

### BEFORE (MaxSdk):
```csharp
void Start()
{
    MaxSdk.SetSdkKey("YOUR_MAX_SDK_KEY");
    MaxSdk.SetHasUserConsent(true);
    MaxSdk.SetDoNotSell(false);
    MaxSdk.SetVerboseLogging(true);

    MaxSdkCallbacks.OnSdkInitializedEvent += OnSdkInitialized;
    MaxSdk.InitializeSdk();
}

private void OnSdkInitialized(MaxSdkBase.SdkConfiguration sdkConfiguration)
{
    // SDK ready — load ads here
}
```

### AFTER (MeticaSdk):
```csharp
async void Start()
{
    // Privacy settings MUST come before Initialize
    MeticaSdk.Ads.SetHasUserConsent(true);
    MeticaSdk.Ads.SetDoNotSell(false);
    MeticaSdk.SetLogEnabled(true); // replaces SetVerboseLogging

    var config = new MeticaInitConfig("YOUR_API_KEY", "YOUR_APP_ID", "USER_ID");
    var mediationInfo = new MeticaMediationInfo(
        MeticaMediationInfo.MeticaMediationType.MAX,
        "YOUR_MAX_SDK_KEY"  // same key, now passed here
    );

    MeticaInitResponse initResponse = await MeticaSdk.InitializeAsync(config, mediationInfo);

    if (initResponse.SmartFloors.IsSuccess)
    {
        // SDK ready — load ads here
    }
}
```

### Alternative: Callback-based initialization (if async/await is not suitable):
```csharp
void Start()
{
    MeticaSdk.Ads.SetHasUserConsent(true);
    MeticaSdk.Ads.SetDoNotSell(false);
    MeticaSdk.SetLogEnabled(true);

    var config = new MeticaInitConfig("YOUR_API_KEY", "YOUR_APP_ID", "USER_ID");
    var mediationInfo = new MeticaMediationInfo(
        MeticaMediationInfo.MeticaMediationType.MAX,
        "YOUR_MAX_SDK_KEY"
    );

    MeticaSdk.Initialize(config, mediationInfo, OnInitialized);
}

private void OnInitialized(MeticaInitResponse initResponse)
{
    // SDK ready — load ads here
}
```

### Key differences:
- `MeticaInitConfig` requires: API Key, App ID, and optionally User ID (auto-generated if omitted)
- `MeticaMediationInfo` takes the MAX SDK Key (replaces `MaxSdk.SetSdkKey()`)
- `MeticaInitResponse` contains `SmartFloors` (user group assignment: TRIAL/HOLDOUT) — new MeticaSdk feature
- `SetVerboseLogging(bool)` → `SetLogEnabled(bool)`
- `MaxSdk.IsInitialized()` → `MeticaSdk.Ads.Max.IsSuccessfullyInitialized()` (available after init completes)

---

## Step 4: Replace Ad API Calls

### 4a. Banner Ads

| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdk.CreateBanner(id, new AdViewConfiguration(AdViewPosition.XXX))` | `MeticaSdk.Ads.CreateBanner(id, new MeticaAdViewConfiguration(MeticaAdViewPosition.XXX))` |
| `MaxSdk.LoadBanner(id)` | `MeticaSdk.Ads.LoadBanner(id)` |
| `MaxSdk.ShowBanner(id)` | `MeticaSdk.Ads.ShowBanner(id)` |
| `MaxSdk.HideBanner(id)` | `MeticaSdk.Ads.HideBanner(id)` |
| `MaxSdk.DestroyBanner(id)` | `MeticaSdk.Ads.DestroyBanner(id)` |
| `MaxSdk.SetBannerPlacement(id, placement)` | `MeticaSdk.Ads.SetBannerPlacement(id, placement)` |
| `MaxSdk.StartBannerAutoRefresh(id)` | `MeticaSdk.Ads.StartBannerAutoRefresh(id)` |
| `MaxSdk.StopBannerAutoRefresh(id)` | `MeticaSdk.Ads.StopBannerAutoRefresh(id)` |
| `MaxSdk.SetBannerBackgroundColor(id, color)` | `MeticaSdk.Ads.SetBannerBackgroundColor(id, hexColorString)` |
| `MaxSdk.SetBannerExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetBannerExtraParameter(id, key, value)` |
| `MaxSdk.SetBannerLocalExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetBannerLocalExtraParameter(id, key, value)` |
| `MaxSdk.SetBannerCustomData(id, data)` | `MeticaSdk.Ads.SetBannerCustomData(id, data)` |
| `MaxSdk.SetBannerWidth(id, width)` | `MeticaSdk.Ads.SetBannerWidth(id, width)` |

**Banner position mapping:**
- `MaxSdkBase.BannerPosition.TopCenter` → `MeticaAdViewPosition.TopCenter`
- `MaxSdkBase.BannerPosition.BottomCenter` → `MeticaAdViewPosition.BottomCenter`
- `MaxSdkBase.BannerPosition.Centered` → `MeticaAdViewPosition.Centered`

**Note on `SetBannerBackgroundColor`:** MaxSdk takes `UnityEngine.Color`, MeticaSdk takes a hex color string (e.g., `"#FFFFFF"`). Convert accordingly.

### 4b. MREC Ads (Note: `MRec` → `Mrec` casing change)

| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdk.CreateMRec(id, config)` | `MeticaSdk.Ads.CreateMrec(id, config)` |
| `MaxSdk.LoadMRec(id)` | `MeticaSdk.Ads.LoadMrec(id)` |
| `MaxSdk.ShowMRec(id)` | `MeticaSdk.Ads.ShowMrec(id)` |
| `MaxSdk.HideMRec(id)` | `MeticaSdk.Ads.HideMrec(id)` |
| `MaxSdk.DestroyMRec(id)` | `MeticaSdk.Ads.DestroyMrec(id)` |
| `MaxSdk.SetMRecPlacement(id, placement)` | `MeticaSdk.Ads.SetMrecPlacement(id, placement)` |
| `MaxSdk.StartMRecAutoRefresh(id)` | `MeticaSdk.Ads.StartMrecAutoRefresh(id)` |
| `MaxSdk.StopMRecAutoRefresh(id)` | `MeticaSdk.Ads.StopMrecAutoRefresh(id)` |
| `MaxSdk.SetMRecExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetMrecExtraParameter(id, key, value)` |
| `MaxSdk.SetMRecLocalExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetMrecLocalExtraParameter(id, key, value)` |
| `MaxSdk.SetMRecCustomData(id, data)` | `MeticaSdk.Ads.SetMrecCustomData(id, data)` |

### 4c. Interstitial Ads

| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdk.LoadInterstitial(id)` | `MeticaSdk.Ads.LoadInterstitial(id)` |
| `MaxSdk.IsInterstitialReady(id)` | `MeticaSdk.Ads.IsInterstitialReady(id)` |
| `MaxSdk.ShowInterstitial(id)` | `MeticaSdk.Ads.ShowInterstitial(id)` |
| `MaxSdk.ShowInterstitial(id, placement)` | `MeticaSdk.Ads.ShowInterstitial(id, placement)` |
| `MaxSdk.ShowInterstitial(id, placement, customData)` | `MeticaSdk.Ads.ShowInterstitial(id, placement, customData)` |
| `MaxSdk.SetInterstitialExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetInterstitialExtraParameter(id, key, value)` |
| `MaxSdk.SetInterstitialLocalExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetInterstitialLocalExtraParameter(id, key, value)` |

### 4d. Rewarded Ads (Note: "Ad" dropped from method names)

| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdk.LoadRewardedAd(id)` | `MeticaSdk.Ads.LoadRewarded(id)` |
| `MaxSdk.IsRewardedAdReady(id)` | `MeticaSdk.Ads.IsRewardedReady(id)` |
| `MaxSdk.ShowRewardedAd(id)` | `MeticaSdk.Ads.ShowRewarded(id)` |
| `MaxSdk.ShowRewardedAd(id, placement)` | `MeticaSdk.Ads.ShowRewarded(id, placement)` |
| `MaxSdk.ShowRewardedAd(id, placement, customData)` | `MeticaSdk.Ads.ShowRewarded(id, placement, customData)` |
| `MaxSdk.SetRewardedAdExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetRewardedAdExtraParameter(id, key, value)` |
| `MaxSdk.SetRewardedAdLocalExtraParameter(id, key, value)` | `MeticaSdk.Ads.SetRewardedAdLocalExtraParameter(id, key, value)` |

### 4e. Privacy & Settings

| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdk.SetHasUserConsent(bool)` | `MeticaSdk.Ads.SetHasUserConsent(bool)` |
| `MaxSdk.HasUserConsent()` | `MeticaSdk.Ads.Max.HasUserConsent()` |
| `MaxSdk.IsUserConsentSet()` | `MeticaSdk.Ads.Max.IsUserConsentSet()` |
| `MaxSdk.SetDoNotSell(bool)` | `MeticaSdk.Ads.SetDoNotSell(bool)` |
| `MaxSdk.SetVerboseLogging(bool)` | `MeticaSdk.SetLogEnabled(bool)` |
| `MaxSdk.IsVerboseLoggingEnabled()` | _(no getter — `SetLogEnabled` only)_ |
| `MaxSdk.SetMuted(bool)` | `MeticaSdk.Ads.Max.SetMuted(bool)` |
| `MaxSdk.IsMuted()` | `MeticaSdk.Ads.Max.IsMuted()` |
| `MaxSdk.SetExtraParameter(key, value)` | `MeticaSdk.Ads.Max.SetExtraParameter(key, value)` |
| `MaxSdk.ShowMediationDebugger()` | `MeticaSdk.Ads.Max.ShowMediationDebugger()` |
| `MaxSdk.IsInitialized()` | `MeticaSdk.Ads.Max.IsSuccessfullyInitialized()` |
| `MaxSdk.Version` | `MeticaSdk.Version` |

---

## Step 5: Replace Callback Subscriptions

### 5a. Callback Name Mapping

Replace all event subscriptions using this mapping:

**Banner:**
| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdkCallbacks.Banner.OnAdLoadedEvent` | `MeticaAdsCallbacks.Banner.OnAdLoadSuccess` |
| `MaxSdkCallbacks.Banner.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Banner.OnAdLoadFailed` |
| `MaxSdkCallbacks.Banner.OnAdClickedEvent` | `MeticaAdsCallbacks.Banner.OnAdClicked` |
| `MaxSdkCallbacks.Banner.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Banner.OnAdRevenuePaid` |

**MREC (MRec → Mrec):**
| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdkCallbacks.MRec.OnAdLoadedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadSuccess` |
| `MaxSdkCallbacks.MRec.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadFailed` |
| `MaxSdkCallbacks.MRec.OnAdClickedEvent` | `MeticaAdsCallbacks.Mrec.OnAdClicked` |
| `MaxSdkCallbacks.MRec.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Mrec.OnAdRevenuePaid` |

**Interstitial:**
| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdkCallbacks.Interstitial.OnAdLoadedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdLoadSuccess` |
| `MaxSdkCallbacks.Interstitial.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdLoadFailed` |
| `MaxSdkCallbacks.Interstitial.OnAdDisplayedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdShowSuccess` |
| `MaxSdkCallbacks.Interstitial.OnAdDisplayFailedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdShowFailed` |
| `MaxSdkCallbacks.Interstitial.OnAdClickedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdClicked` |
| `MaxSdkCallbacks.Interstitial.OnAdHiddenEvent` | `MeticaAdsCallbacks.Interstitial.OnAdHidden` |
| `MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Interstitial.OnAdRevenuePaid` |

**Rewarded:**
| MaxSdk (find this) | MeticaSdk (replace with) |
|---------------------|--------------------------|
| `MaxSdkCallbacks.Rewarded.OnAdLoadedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdLoadSuccess` |
| `MaxSdkCallbacks.Rewarded.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdLoadFailed` |
| `MaxSdkCallbacks.Rewarded.OnAdDisplayedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdShowSuccess` |
| `MaxSdkCallbacks.Rewarded.OnAdDisplayFailedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdShowFailed` |
| `MaxSdkCallbacks.Rewarded.OnAdClickedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdClicked` |
| `MaxSdkCallbacks.Rewarded.OnAdHiddenEvent` | `MeticaAdsCallbacks.Rewarded.OnAdHidden` |
| `MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent` | `MeticaAdsCallbacks.Rewarded.OnAdRewarded` |
| `MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Rewarded.OnAdRevenuePaid` |

### 5b. Callback Signature Changes

All callback handler methods must have their signatures updated. MaxSdk passes separate parameters; MeticaSdk passes a single record.

**Standard ad callback (loaded, clicked, displayed, hidden, revenue):**
```csharp
// BEFORE (MaxSdk)
void OnAdLoaded(string adUnitId, MaxSdkBase.AdInfo adInfo) { ... }

// AFTER (MeticaSdk)
void OnAdLoaded(MeticaAd ad) { ... }
```

**Error callback (load failed):**
```csharp
// BEFORE (MaxSdk)
void OnAdLoadFailed(string adUnitId, MaxSdkBase.ErrorInfo errorInfo) { ... }

// AFTER (MeticaSdk)
void OnAdLoadFailed(MeticaAdError error) { ... }
```

**Show-failed callback:**
```csharp
// BEFORE (MaxSdk)
void OnAdDisplayFailed(string adUnitId, MaxSdkBase.ErrorInfo errorInfo, MaxSdkBase.AdInfo adInfo) { ... }

// AFTER (MeticaSdk)
void OnAdShowFailed(MeticaAd ad, MeticaAdError error) { ... }
```

**Rewarded callback:**
```csharp
// BEFORE (MaxSdk)
void OnAdReceivedReward(string adUnitId, MaxSdk.Reward reward, MaxSdkBase.AdInfo adInfo)
{
    var rewardLabel = reward.Label;
    var rewardAmount = reward.Amount;
}

// AFTER (MeticaSdk) — no separate Reward object
void OnAdRewarded(MeticaAd ad)
{
    // TODO: [MIGRATION] MeticaSdk OnAdRewarded does not provide Reward.Label or Reward.Amount.
    // If your game uses reward details, you must track them separately (e.g., hardcode per ad unit
    // or fetch from your own config).
}
```

---

## Step 6: Replace Data Model Property Access

Inside every callback handler, update property access from MaxSdk models to MeticaSdk models.

### AdInfo → MeticaAd

| MaxSdk (`MaxSdkBase.AdInfo`) | MeticaSdk (`MeticaAd`) | Notes |
|------------------------------|------------------------|-------|
| `adInfo.AdUnitIdentifier` | `ad.adUnitId` | |
| `adInfo.Revenue` | `ad.revenue` | Nullable in MeticaSdk (`double?`) |
| `adInfo.NetworkName` | `ad.networkName` | Nullable in MeticaSdk |
| `adInfo.Placement` | `ad.placementTag` | Different name |
| `adInfo.AdFormat` | `ad.adFormat` | Nullable in MeticaSdk |
| `adInfo.CreativeIdentifier` | `ad.creativeId` | Different name |
| `adInfo.LatencyMillis` | `ad.latency` | Different name, nullable |
| `adInfo.NetworkPlacement` | — | Not available in MeticaSdk |
| `adInfo.RevenuePrecision` | — | Not available in MeticaSdk |
| `adInfo.WaterfallInfo` | — | Not available in MeticaSdk |
| `adInfo.DspName` | — | Not available in MeticaSdk |

### ErrorInfo → MeticaAdError

| MaxSdk (`MaxSdkBase.ErrorInfo`) | MeticaSdk (`MeticaAdError`) | Notes |
|---------------------------------|-----------------------------|-------|
| `errorInfo.Message` | `error.message` | Direct match |
| `errorInfo.Code` | — | No error code enum in MeticaSdk |
| `errorInfo.MediatedNetworkErrorCode` | — | Not available |
| `errorInfo.MediatedNetworkErrorMessage` | — | Not available |
| `errorInfo.AdLoadFailureInfo` | — | Not available (use `error.message`) |
| `errorInfo.WaterfallInfo` | — | Not available |
| _(adUnitId from callback param)_ | `error.adUnitId` | MeticaSdk includes adUnitId in the error object |

For any property that is not available in MeticaSdk, remove the reference or add a `// TODO: [MIGRATION]` comment if the value was used for analytics or logging.

---

## Step 7: Handle Unsupported Features

Search for uses of the following MaxSdk features and handle them as described:

### App Open Ads — ENTIRE FORMAT NOT SUPPORTED
```
Search for: MaxSdk.LoadAppOpenAd, MaxSdk.IsAppOpenAdReady, MaxSdk.ShowAppOpenAd,
            MaxSdkCallbacks.AppOpen, SetAppOpenAdExtraParameter, SetAppOpenAdLocalExtraParameter
```
**Action:** Comment out the entire App Open Ad implementation and add:
```csharp
// TODO: [MIGRATION] App Open Ads are not supported in MeticaSdk 2.2.1.
// Options: Remove this ad format, or keep MaxSdk App Open alongside MeticaSdk for other formats.
```

### Banner/MREC Position Updates — NOT SUPPORTED
```
Search for: MaxSdk.UpdateBannerPosition, MaxSdk.UpdateMRecPosition
```
**Action:** Add TODO comment:
```csharp
// TODO: [MIGRATION] MeticaSdk does not support updating banner/MREC position after creation.
// Workaround: Destroy and recreate the ad view at the new position.
```

### Banner/MREC Layout Retrieval — NOT SUPPORTED
```
Search for: MaxSdk.GetBannerLayout, MaxSdk.GetMRecLayout
```
**Action:** Add TODO comment:
```csharp
// TODO: [MIGRATION] MeticaSdk does not support GetBannerLayout/GetMRecLayout.
```

### Banner Coordinate Positioning / Adaptive Banner — NOT SUPPORTED
```
Search for: AdViewConfiguration with x/y coordinates, IsAdaptive
```
**Action:** MeticaSdk `MeticaAdViewConfiguration` only supports predefined positions (TopCenter, BottomCenter, Centered), not x/y coordinates or adaptive banners. Add TODO if used.

### CMP Service — PARTIALLY SUPPORTED
```
Search for: MaxSdk.CmpService, ShowCmpForExistingUser, HasSupportedCmp
```
**Action:** `ShowCmpForExistingUser` is available via `MeticaSdk.Ads.Max.ShowCmpForExistingUser()`, but the MeticaSdk version has no completion/error callback. `HasSupportedCmp` has no equivalent. Replace what you can, then add a TODO for the missing callback:
```csharp
// BEFORE
MaxSdk.CmpService.ShowCmpForExistingUser(error => { /* handle error */ });

// AFTER
MeticaSdk.Ads.Max.ShowCmpForExistingUser();
// TODO: [MIGRATION] MeticaSdk.Ads.Max.ShowCmpForExistingUser() has no completion or
// error callback. Any post-CMP logic that was in the MaxCmpError handler must be
// moved to a separate trigger (e.g., user action, timer, or next-session logic).

// BEFORE
bool hasCmp = MaxSdk.CmpService.HasSupportedCmp;
// TODO: [MIGRATION] HasSupportedCmp has no MeticaSdk equivalent. Remove this check
// or handle CMP presence detection via a third-party CMP SDK.
```

### Privacy Getters — PARTIALLY SUPPORTED
```
Search for: MaxSdk.IsDoNotSell(), MaxSdk.IsDoNotSellSet()
```
**Action:** These getters have no MeticaSdk equivalent. Add TODO:
```csharp
// TODO: [MIGRATION] IsDoNotSell() and IsDoNotSellSet() have no MeticaSdk equivalent.
// Track do-not-sell state in your own code if needed.
```

### SDK Configuration — NOT SUPPORTED
```
Search for: MaxSdk.GetSdkConfiguration()
```
**Action:** `SdkConfiguration` properties (CountryCode, IsTestModeEnabled, AppTrackingStatus, IsSuccessfullyInitialized) are not available. Only `ConsentFlowUserGeography` is accessible via `MeticaSdk.Ads.Max.GetConsentFlowUserGeography()`.

### Debugging/Testing Features — NOT SUPPORTED
```
Search for: MaxSdk.ShowCreativeDebugger, MaxSdk.SetCreativeDebuggerEnabled,
            MaxSdk.DisableStubAds, MaxSdk.SetTestDeviceAdvertisingIdentifiers
```
**Action:** Remove these calls. They have no MeticaSdk equivalent.

### Mute/Audio — SUPPORTED (moved to `Ads.Max`)
```
Search for: MaxSdk.SetMuted, MaxSdk.IsMuted
```
**Action:** Replace with the equivalent calls under `MeticaSdk.Ads.Max`:
```csharp
MaxSdk.SetMuted(true)       →  MeticaSdk.Ads.Max.SetMuted(true)
MaxSdk.IsMuted()            →  MeticaSdk.Ads.Max.IsMuted()
```

### Safe Area / Misc — NOT SUPPORTED
```
Search for: MaxSdk.GetSafeAreaInsets, MaxSdk.SetExceptionHandlerEnabled,
            MaxSdkBase.InvokeEventsOnUnityMainThread
```
**Action:** Remove or add TODO comments as appropriate.

### User ID / Segmentation
```
Search for: MaxSdk.SetUserId, MaxSdk.SetSegmentCollection, MaxSegmentCollection, MaxSegment
```
**Action:** `SetUserId` is replaced by passing `userId` in `MeticaInitConfig`. `SetSegmentCollection` has no MeticaSdk equivalent — remove and add TODO.

### Event Tracking
```
Search for: MaxSdk.TrackEvent
```
**Action:** MaxSdk's `TrackEvent(string, IDictionary)` has no direct equivalent. MeticaSdk has a richer event system via `MeticaSdk.Events.*` — consider migrating to typed events:
```csharp
// TODO: [MIGRATION] Replace MaxSdk.TrackEvent with MeticaSdk.Events methods.
// Available: LogLoginEvent, LogInstallEvent, LogAdRevenueEvent, LogCustomEvent, etc.
```

### Callbacks Without MeticaSdk Equivalent
```
Search for: OnApplicationStateChangedEvent, OnExpiredAdReloadedEvent,
            OnAdReviewCreativeIdGeneratedEvent, OnAdExpandedEvent, OnAdCollapsedEvent
```
**Action:** These events do not exist in MeticaSdk. Remove the subscriptions and handler methods. Add TODO if the behavior was important.

---

## Step 8: Verification

After completing all changes, perform these checks:

### 8a. Search for remaining MaxSdk references
```
Search entire project for: MaxSdk\. , MaxSdkCallbacks\. , MaxSdkBase\.
```
There should be **zero matches** in your application code (matches in the AppLovin plugin folder itself are expected and should be left alone).

### 8b. Compile check
Build the project in Unity. Fix any compilation errors. Common issues:
- Missing `using Metica;` or `using Metica.Ads;`
- Wrong casing: `MRec` instead of `Mrec`
- Old callback signatures with `(string adUnitId, ...)` instead of `(MeticaAd ad)`
- `MaxSdkBase.AdInfo` or `MaxSdkBase.ErrorInfo` type references not replaced
- `Color` passed to `SetBannerBackgroundColor` instead of hex string

### 8c. Runtime test checklist
- [ ] SDK initializes without errors (check console for Metica init logs)
- [ ] `initResponse.SmartFloors.IsSuccess` returns true
- [ ] Banner ads load and display
- [ ] MREC ads load and display
- [ ] Interstitial ads load, show, and dismiss correctly
- [ ] Rewarded ads load, show, grant reward, and dismiss correctly
- [ ] Revenue callbacks fire for all ad formats
- [ ] Privacy consent is applied correctly
- [ ] Test on both iOS and Android

### 8d. Cleanup
- Remove any unused MaxSdk-related helper methods or variables that are no longer needed
- Remove unused `MaxSdk.Reward` variable references
- Remove any `#if` blocks that were gating MaxSdk-specific behavior

---

## Complete Before/After Example: Interstitial Ad Manager

### BEFORE (MaxSdk):
```csharp
using UnityEngine;

public class InterstitialAdManager : MonoBehaviour
{
    private const string AdUnitId = "YOUR_INTERSTITIAL_ID";
    private int retryAttempt;

    void Start()
    {
        MaxSdkCallbacks.Interstitial.OnAdLoadedEvent += OnAdLoaded;
        MaxSdkCallbacks.Interstitial.OnAdLoadFailedEvent += OnAdLoadFailed;
        MaxSdkCallbacks.Interstitial.OnAdDisplayedEvent += OnAdDisplayed;
        MaxSdkCallbacks.Interstitial.OnAdDisplayFailedEvent += OnAdDisplayFailed;
        MaxSdkCallbacks.Interstitial.OnAdHiddenEvent += OnAdHidden;
        MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent += OnAdRevenuePaid;

        MaxSdk.LoadInterstitial(AdUnitId);
    }

    void OnAdLoaded(string adUnitId, MaxSdkBase.AdInfo adInfo)
    {
        retryAttempt = 0;
        Debug.Log("Loaded from " + adInfo.NetworkName);
    }

    void OnAdLoadFailed(string adUnitId, MaxSdkBase.ErrorInfo errorInfo)
    {
        retryAttempt++;
        float delay = Mathf.Pow(2, Mathf.Min(6, retryAttempt));
        Debug.LogWarning("Load failed: " + errorInfo.Message);
        Invoke(nameof(LoadAd), delay);
    }

    void OnAdDisplayed(string adUnitId, MaxSdkBase.AdInfo adInfo)
    {
        Debug.Log("Displayed");
    }

    void OnAdDisplayFailed(string adUnitId, MaxSdkBase.ErrorInfo errorInfo, MaxSdkBase.AdInfo adInfo)
    {
        Debug.LogError("Display failed: " + errorInfo.Message);
        LoadAd();
    }

    void OnAdHidden(string adUnitId, MaxSdkBase.AdInfo adInfo)
    {
        LoadAd();
    }

    void OnAdRevenuePaid(string adUnitId, MaxSdkBase.AdInfo adInfo)
    {
        Debug.Log($"Revenue: {adInfo.Revenue} from {adInfo.NetworkName}");
    }

    void LoadAd() => MaxSdk.LoadInterstitial(AdUnitId);

    public void ShowAd()
    {
        if (MaxSdk.IsInterstitialReady(AdUnitId))
            MaxSdk.ShowInterstitial(AdUnitId);
    }
}
```

### AFTER (MeticaSdk):
```csharp
using Metica;
using Metica.Ads;
using UnityEngine;

public class InterstitialAdManager : MonoBehaviour
{
    private const string AdUnitId = "YOUR_INTERSTITIAL_ID";
    private int retryAttempt;

    void Start()
    {
        MeticaAdsCallbacks.Interstitial.OnAdLoadSuccess += OnAdLoaded;
        MeticaAdsCallbacks.Interstitial.OnAdLoadFailed += OnAdLoadFailed;
        MeticaAdsCallbacks.Interstitial.OnAdShowSuccess += OnAdDisplayed;
        MeticaAdsCallbacks.Interstitial.OnAdShowFailed += OnAdDisplayFailed;
        MeticaAdsCallbacks.Interstitial.OnAdHidden += OnAdHidden;
        MeticaAdsCallbacks.Interstitial.OnAdRevenuePaid += OnAdRevenuePaid;

        MeticaSdk.Ads.LoadInterstitial(AdUnitId);
    }

    void OnAdLoaded(MeticaAd ad)
    {
        retryAttempt = 0;
        Debug.Log("Loaded from " + ad.networkName);
    }

    void OnAdLoadFailed(MeticaAdError error)
    {
        retryAttempt++;
        float delay = Mathf.Pow(2, Mathf.Min(6, retryAttempt));
        Debug.LogWarning("Load failed: " + error.message);
        Invoke(nameof(LoadAd), delay);
    }

    void OnAdDisplayed(MeticaAd ad)
    {
        Debug.Log("Displayed");
    }

    void OnAdDisplayFailed(MeticaAd ad, MeticaAdError error)
    {
        Debug.LogError("Display failed: " + error.message);
        LoadAd();
    }

    void OnAdHidden(MeticaAd ad)
    {
        LoadAd();
    }

    void OnAdRevenuePaid(MeticaAd ad)
    {
        Debug.Log($"Revenue: {ad.revenue} from {ad.networkName}");
    }

    void LoadAd() => MeticaSdk.Ads.LoadInterstitial(AdUnitId);

    public void ShowAd()
    {
        if (MeticaSdk.Ads.IsInterstitialReady(AdUnitId))
            MeticaSdk.Ads.ShowInterstitial(AdUnitId);
    }
}
```
