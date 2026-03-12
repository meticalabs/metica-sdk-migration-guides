# Migration Guide: AppLovin MAX SDK 8.6.0 → MeticaSdk 2.2.5

> **Scope**: Unity C# game code only. This guide covers public API call-site changes.
> Native plugin configuration (Podfile, Gradle, AppLovin dashboard) is out of scope.

---

## Before You Start

### Prerequisites checklist

- [ ] MeticaSdk 2.2.5 unitypackage imported into your project
- [ ] AppLovin MAX 8.6.0 unitypackage **kept** in the project (MeticaSdk wraps it; it is still required)
- [ ] Review the [known gaps](#known-gaps--no-direct-replacement) — if your game uses **App Open Ads**, **reward amount/label logic**, or **MAX segmentation**, plan workarounds before migrating

### What changes and what stays the same

| | MaxSdk | MeticaSdk |
|---|---|---|
| Entry class | `MaxSdk` (static) | `MeticaSdk.Ads` (instance via `MeticaSdk`) |
| AppLovin-specific helpers | `MaxSdk.*` | `MeticaSdk.Ads.Max.*` |
| Events | `MaxSdkCallbacks.*` | `MeticaAdsCallbacks.*` |
| Ad info type | `MaxSdkBase.AdInfo` | `MeticaAd` |
| Error type | `MaxSdkBase.ErrorInfo` | `MeticaAdError` |
| Ad position enum | `MaxSdkBase.AdViewPosition` | `Metica.Ads.MeticaAdViewPosition` |
| Ad view config | `AdViewConfiguration` | `MeticaAdViewConfiguration` |

---

## Step 1 — Initialization

### Before (MaxSdk)

```csharp
MaxSdkCallbacks.OnSdkInitializedEvent += OnSdkInitialized;
MaxSdk.SetHasUserConsent(true);
MaxSdk.SetDoNotSell(false);
MaxSdk.InitializeSdk();

private void OnSdkInitialized(MaxSdkBase.SdkConfiguration config)
{
    Debug.Log("MAX initialized. Country: " + config.CountryCode);
    // start loading ads...
}
```

### After (MeticaSdk)

```csharp
// Set consent BEFORE initialize, same as before
MeticaSdk.Ads.SetHasUserConsent(true);
MeticaSdk.Ads.SetDoNotSell(false);
MeticaSdk.SetLogEnabled(true); // replaces MaxSdk.SetVerboseLogging(true)

// Callback-based
var config = new MeticaInitConfig("YOUR_API_KEY", "YOUR_APP_ID", "USER_ID");
var mediationInfo = new MeticaMediationInfo(
    MeticaMediationInfo.MeticaMediationType.MAX,
    "YOUR_MAX_SDK_KEY"  // same key previously passed to MaxSdk.SetSdkKey()
);
MeticaSdk.Initialize(config, mediationInfo, response =>
{
    Debug.Log("Metica initialized. Country: " + MeticaSdk.Ads.Max.CountryCode());
    // response.SmartFloors?.UserGroup — TRIAL or HOLDOUT
    // start loading ads...
});

// OR async
var response = await MeticaSdk.InitializeAsync(config, mediationInfo);
Debug.Log("Metica initialized.");
```

**Key differences:**
- There is no `OnSdkInitializedEvent` — use the callback or `await`.
- `SdkConfiguration` fields are now accessed individually via `MeticaSdk.Ads.Max.*` (see [Privacy](#step-2--privacy--consent)).
- `IsInitialized()` → `MeticaSdk.Ads.Max.IsSuccessfullyInitialized()`.
- `MaxSdk.SetVerboseLogging(bool)` → `MeticaSdk.SetLogEnabled(bool)` ✅ (confirmed available in 2.2.5).
- `MeticaInitResponse` contains `SmartFloors` (user group: `TRIAL`/`HOLDOUT`) — new MeticaSdk feature, not present in MaxSdk.

---

## Step 2 — Privacy / Consent

### Before (MaxSdk)

```csharp
MaxSdk.SetHasUserConsent(hasConsent);
MaxSdk.SetDoNotSell(doNotSell);

bool consent   = MaxSdk.HasUserConsent();
bool consentSet = MaxSdk.IsUserConsentSet();

var config = MaxSdk.GetSdkConfiguration();
var geo    = config.ConsentFlowUserGeography;
var state  = config.ConsentDialogState;
string cc  = config.CountryCode;

MaxSdk.ShowMediationDebugger();
// Show CMP for returning user:
MaxCmpService.ShowCmpForExistingUser(error => { });
```

### After (MeticaSdk)

```csharp
MeticaSdk.Ads.SetHasUserConsent(hasConsent);
MeticaSdk.Ads.SetDoNotSell(doNotSell);

bool consent    = MeticaSdk.Ads.Max.HasUserConsent();
bool consentSet = MeticaSdk.Ads.Max.IsUserConsentSet();

var geo   = MeticaSdk.Ads.Max.GetConsentFlowUserGeography();
var state = MeticaSdk.Ads.Max.ConsentDialogState();
string cc = MeticaSdk.Ads.Max.CountryCode();

MeticaSdk.Ads.Max.ShowMediationDebugger();
MeticaSdk.Ads.Max.ShowCmpForExistingUser();
```

**Not available in MeticaSdk:**
- `MaxSdk.IsDoNotSell()` — no read-back of DoNotSell flag.
- `MaxSdk.IsDoNotSellSet()` — no check-if-set.
- `MaxSdk.GetSdkConfiguration()` as a single struct — fields split across individual `Max.*` methods.

---

## Step 3 — Banners

### Before (MaxSdk)

```csharp
// Subscribe
MaxSdkCallbacks.Banner.OnAdLoadedEvent       += OnBannerLoaded;
MaxSdkCallbacks.Banner.OnAdLoadFailedEvent   += OnBannerLoadFailed;
MaxSdkCallbacks.Banner.OnAdClickedEvent      += OnBannerClicked;
MaxSdkCallbacks.Banner.OnAdRevenuePaidEvent  += OnBannerRevenuePaid;

// Create & show
MaxSdk.CreateBanner(bannerAdUnitId, MaxSdkBase.AdViewPosition.BottomCenter);
MaxSdk.SetBannerBackgroundColor(bannerAdUnitId, Color.black);
MaxSdk.ShowBanner(bannerAdUnitId);

// Callbacks
private void OnBannerLoaded(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    Debug.Log($"Banner loaded. Network: {adInfo.NetworkName}, Revenue: {adInfo.Revenue}");
}
private void OnBannerLoadFailed(string adUnitId, MaxSdkBase.ErrorInfo error)
{
    Debug.LogError($"Banner failed [{error.Code}]: {error.Message}");
}
private void OnBannerRevenuePaid(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    LogRevenue(adInfo.Revenue, adInfo.RevenuePrecision, adInfo.NetworkName);
}
```

### After (MeticaSdk)

```csharp
using Metica.Ads;

// Subscribe
MeticaAdsCallbacks.Banner.OnAdLoadSuccess  += OnBannerLoaded;
MeticaAdsCallbacks.Banner.OnAdLoadFailed   += OnBannerLoadFailed;
MeticaAdsCallbacks.Banner.OnAdClicked      += OnBannerClicked;
MeticaAdsCallbacks.Banner.OnAdRevenuePaid  += OnBannerRevenuePaid;

// Create & show
var cfg = new MeticaAdViewConfiguration(MeticaAdViewPosition.BottomCenter);
MeticaSdk.Ads.CreateBanner(bannerAdUnitId, cfg);
MeticaSdk.Ads.SetBannerBackgroundColor(bannerAdUnitId, "#000000"); // hex string, not Color
MeticaSdk.Ads.ShowBanner(bannerAdUnitId);

// Callbacks — adUnitId is now in meticaAd.adUnitId
private void OnBannerLoaded(MeticaAd meticaAd)
{
    Debug.Log($"Banner loaded. Network: {meticaAd.networkName}, Revenue: {meticaAd.revenue}");
}
private void OnBannerLoadFailed(MeticaAdError error)
{
    Debug.LogError($"Banner failed: {error.message}");  // no error code enum
}
private void OnBannerRevenuePaid(MeticaAd meticaAd)
{
    LogRevenue(meticaAd.revenue ?? 0, meticaAd.networkName);
    // NOTE: RevenuePrecision not available in MeticaAd
}
```

**Additional banner methods — rename map:**

| MaxSdk | MeticaSdk |
|--------|-----------|
| `MaxSdk.HideBanner(id)` | `MeticaSdk.Ads.HideBanner(id)` |
| `MaxSdk.DestroyBanner(id)` | `MeticaSdk.Ads.DestroyBanner(id)` |
| `MaxSdk.LoadBanner(id)` | `MeticaSdk.Ads.LoadBanner(id)` |
| `MaxSdk.StartBannerAutoRefresh(id)` | `MeticaSdk.Ads.StartBannerAutoRefresh(id)` |
| `MaxSdk.StopBannerAutoRefresh(id)` | `MeticaSdk.Ads.StopBannerAutoRefresh(id)` |
| `MaxSdk.SetBannerPlacement(id, p)` | `MeticaSdk.Ads.SetBannerPlacement(id, p)` |
| `MaxSdk.SetBannerWidth(id, w)` | `MeticaSdk.Ads.SetBannerWidth(id, w)` |
| `MaxSdk.SetBannerExtraParameter(id, k, v)` | `MeticaSdk.Ads.SetBannerExtraParameter(id, k, v)` |
| `MaxSdk.SetBannerLocalExtraParameter(id, k, v)` | `MeticaSdk.Ads.SetBannerLocalExtraParameter(id, k, v)` |
| `MaxSdk.SetBannerCustomData(id, d)` | `MeticaSdk.Ads.SetBannerCustomData(id, d)` |

**Workaround — dynamic banner position update:**

`UpdateBannerPosition` is not available. To reposition a banner at runtime:

```csharp
// MaxSdk (before):
MaxSdk.UpdateBannerPosition(bannerAdUnitId, MaxSdkBase.AdViewPosition.TopCenter);

// MeticaSdk workaround — destroy and recreate:
MeticaSdk.Ads.DestroyBanner(bannerAdUnitId);
var newCfg = new MeticaAdViewConfiguration(MeticaAdViewPosition.TopCenter);
MeticaSdk.Ads.CreateBanner(bannerAdUnitId, newCfg);
MeticaSdk.Ads.ShowBanner(bannerAdUnitId);
```

**Not available:**
- `MaxSdk.GetBannerLayout(id)` — no layout rect query.
- `OnAdExpandedEvent` / `OnAdCollapsedEvent` — no expand/collapse state callbacks.

---

## Step 4 — MRECs

The MREC API is identical in structure to banners. The only change is the capitalisation: **`MRec` → `Mrec`** everywhere.

### Rename map

| MaxSdk | MeticaSdk |
|--------|-----------|
| `MaxSdk.CreateMRec(id, cfg)` | `MeticaSdk.Ads.CreateMrec(id, cfg)` |
| `MaxSdk.LoadMRec(id)` | `MeticaSdk.Ads.LoadMrec(id)` |
| `MaxSdk.ShowMRec(id)` | `MeticaSdk.Ads.ShowMrec(id)` |
| `MaxSdk.HideMRec(id)` | `MeticaSdk.Ads.HideMrec(id)` |
| `MaxSdk.DestroyMRec(id)` | `MeticaSdk.Ads.DestroyMrec(id)` |
| `MaxSdk.StartMRecAutoRefresh(id)` | `MeticaSdk.Ads.StartMrecAutoRefresh(id)` |
| `MaxSdk.StopMRecAutoRefresh(id)` | `MeticaSdk.Ads.StopMrecAutoRefresh(id)` |
| `MaxSdk.SetMRecPlacement(id, p)` | `MeticaSdk.Ads.SetMrecPlacement(id, p)` |
| `MaxSdk.SetMRecCustomData(id, d)` | `MeticaSdk.Ads.SetMrecCustomData(id, d)` |
| `MaxSdk.SetMRecExtraParameter(id, k, v)` | `MeticaSdk.Ads.SetMrecExtraParameter(id, k, v)` |
| `MaxSdk.SetMRecLocalExtraParameter(id, k, v)` | `MeticaSdk.Ads.SetMrecLocalExtraParameter(id, k, v)` |
| `MaxSdkCallbacks.MRec.OnAdLoadedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadSuccess` |
| `MaxSdkCallbacks.MRec.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadFailed` |
| `MaxSdkCallbacks.MRec.OnAdClickedEvent` | `MeticaAdsCallbacks.Mrec.OnAdClicked` |
| `MaxSdkCallbacks.MRec.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Mrec.OnAdRevenuePaid` |

Same workaround applies for position update (destroy + recreate). Same gaps apply (`GetMRecLayout`, expand/collapse events).

---

## Step 5 — Interstitials

Interstitials have the closest parity. Method signatures are identical; only event names changed.

### Before (MaxSdk)

```csharp
MaxSdkCallbacks.Interstitial.OnAdLoadedEvent       += OnInterstitialLoaded;
MaxSdkCallbacks.Interstitial.OnAdLoadFailedEvent   += OnInterstitialLoadFailed;
MaxSdkCallbacks.Interstitial.OnAdDisplayedEvent    += OnInterstitialDisplayed;
MaxSdkCallbacks.Interstitial.OnAdDisplayFailedEvent += OnInterstitialDisplayFailed;
MaxSdkCallbacks.Interstitial.OnAdHiddenEvent       += OnInterstitialHidden;
MaxSdkCallbacks.Interstitial.OnAdClickedEvent      += OnInterstitialClicked;
MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent  += OnInterstitialRevenuePaid;

MaxSdk.LoadInterstitial(interstitialAdUnitId);

private void ShowInterstitialIfReady()
{
    if (MaxSdk.IsInterstitialReady(interstitialAdUnitId))
        MaxSdk.ShowInterstitial(interstitialAdUnitId, "main_menu");
}

private void OnInterstitialDisplayed(string adUnitId, MaxSdkBase.AdInfo adInfo) { }
private void OnInterstitialDisplayFailed(string adUnitId, MaxSdkBase.ErrorInfo error,
                                          MaxSdkBase.AdInfo adInfo) { }
private void OnInterstitialHidden(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    // reload
    MaxSdk.LoadInterstitial(adUnitId);
}
```

### After (MeticaSdk)

```csharp
using Metica.Ads;

MeticaAdsCallbacks.Interstitial.OnAdLoadSuccess  += OnInterstitialLoaded;
MeticaAdsCallbacks.Interstitial.OnAdLoadFailed   += OnInterstitialLoadFailed;
MeticaAdsCallbacks.Interstitial.OnAdShowSuccess  += OnInterstitialDisplayed;    // renamed
MeticaAdsCallbacks.Interstitial.OnAdShowFailed   += OnInterstitialDisplayFailed; // renamed
MeticaAdsCallbacks.Interstitial.OnAdHidden       += OnInterstitialHidden;
MeticaAdsCallbacks.Interstitial.OnAdClicked      += OnInterstitialClicked;
MeticaAdsCallbacks.Interstitial.OnAdRevenuePaid  += OnInterstitialRevenuePaid;

MeticaSdk.Ads.LoadInterstitial(interstitialAdUnitId);

private void ShowInterstitialIfReady()
{
    if (MeticaSdk.Ads.IsInterstitialReady(interstitialAdUnitId))
        MeticaSdk.Ads.ShowInterstitial(interstitialAdUnitId, "main_menu");
}

// adUnitId is in meticaAd.adUnitId; error and ad parameters swapped vs MaxSdk
private void OnInterstitialDisplayed(MeticaAd meticaAd) { }
private void OnInterstitialDisplayFailed(MeticaAd meticaAd, MeticaAdError error) { }
private void OnInterstitialHidden(MeticaAd meticaAd)
{
    MeticaSdk.Ads.LoadInterstitial(meticaAd.adUnitId);
}
```

**Note — `OnAdDisplayFailedEvent` parameter order changed:**

```csharp
// MaxSdk:    Action<string adUnitId, ErrorInfo error, AdInfo adInfo>
// MeticaSdk: Action<MeticaAd meticaAd, MeticaAdError error>
```

---

## Step 6 — Rewarded Ads

### Before (MaxSdk)

```csharp
MaxSdkCallbacks.Rewarded.OnAdLoadedEvent          += OnRewardedLoaded;
MaxSdkCallbacks.Rewarded.OnAdLoadFailedEvent      += OnRewardedLoadFailed;
MaxSdkCallbacks.Rewarded.OnAdDisplayedEvent       += OnRewardedDisplayed;
MaxSdkCallbacks.Rewarded.OnAdDisplayFailedEvent   += OnRewardedDisplayFailed;
MaxSdkCallbacks.Rewarded.OnAdHiddenEvent          += OnRewardedHidden;
MaxSdkCallbacks.Rewarded.OnAdClickedEvent         += OnRewardedClicked;
MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent  += OnRewardReceived;
MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent     += OnRewardedRevenuePaid;

MaxSdk.LoadRewardedAd(rewardedAdUnitId);

private void ShowRewardedIfReady()
{
    if (MaxSdk.IsRewardedAdReady(rewardedAdUnitId))
        MaxSdk.ShowRewardedAd(rewardedAdUnitId, "level_complete");
}

private void OnRewardReceived(string adUnitId, MaxSdkBase.Reward reward, MaxSdkBase.AdInfo adInfo)
{
    // reward.Label = "coins", reward.Amount = 50
    GrantReward(reward.Label, reward.Amount);
}
```

### After (MeticaSdk)

```csharp
using Metica.Ads;

MeticaAdsCallbacks.Rewarded.OnAdLoadSuccess  += OnRewardedLoaded;
MeticaAdsCallbacks.Rewarded.OnAdLoadFailed   += OnRewardedLoadFailed;
MeticaAdsCallbacks.Rewarded.OnAdShowSuccess  += OnRewardedDisplayed;   // renamed
MeticaAdsCallbacks.Rewarded.OnAdShowFailed   += OnRewardedDisplayFailed; // renamed
MeticaAdsCallbacks.Rewarded.OnAdHidden       += OnRewardedHidden;
MeticaAdsCallbacks.Rewarded.OnAdClicked      += OnRewardedClicked;
MeticaAdsCallbacks.Rewarded.OnAdRewarded     += OnRewardReceived;      // renamed
MeticaAdsCallbacks.Rewarded.OnAdRevenuePaid  += OnRewardedRevenuePaid;

MeticaSdk.Ads.LoadRewarded(rewardedAdUnitId);        // renamed

private void ShowRewardedIfReady()
{
    if (MeticaSdk.Ads.IsRewardedReady(rewardedAdUnitId))    // renamed
        MeticaSdk.Ads.ShowRewarded(rewardedAdUnitId, "level_complete"); // renamed
}

// ⚠️ CRITICAL: Reward.Label and Reward.Amount are NOT available.
// MeticaSdk fires OnAdRewarded with only MeticaAd — no reward type or amount.
private void OnRewardReceived(MeticaAd meticaAd)
{
    // You cannot read reward.Label or reward.Amount here.
    // Workaround: hardcode the reward in game logic, or configure it server-side.
    GrantReward("coins", 50); // hardcoded — adapt to your game's design
}
```

> **Action required**: If your game grants different rewards based on `reward.Label` or scales amounts from `reward.Amount`, you must redesign this logic. Options:
> - Configure a fixed reward per ad unit in your game code.
> - Fetch reward details from your own backend when `OnAdRewarded` fires.
> - Wait for MeticaSdk to expose reward data in a future release.

**Rewarded method rename map:**

| MaxSdk | MeticaSdk |
|--------|-----------|
| `LoadRewardedAd(id)` | `LoadRewarded(id)` |
| `IsRewardedAdReady(id)` | `IsRewardedReady(id)` |
| `ShowRewardedAd(id, p, d)` | `ShowRewarded(id, p, d)` |
| `SetRewardedAdExtraParameter(id, k, v)` | `SetRewardedAdExtraParameter(id, k, v)` |
| `SetRewardedAdLocalExtraParameter(id, k, v)` | `SetRewardedAdLocalExtraParameter(id, k, v)` |

---

## Step 7 — App Open Ads

**App Open Ads are not supported in MeticaSdk 2.2.5.** There is no replacement for any of the following:

```csharp
// These have NO MeticaSdk equivalent:
MaxSdk.LoadAppOpenAd(adUnitId);
MaxSdk.IsAppOpenAdReady(adUnitId);
MaxSdk.ShowAppOpenAd(adUnitId, placement, customData);
MaxSdk.SetAppOpenAdExtraParameter(adUnitId, key, value);
MaxSdk.SetAppOpenAdLocalExtraParameter(adUnitId, key, value);

MaxSdkCallbacks.AppOpen.OnAdLoadedEvent
MaxSdkCallbacks.AppOpen.OnAdLoadFailedEvent
MaxSdkCallbacks.AppOpen.OnAdDisplayedEvent
MaxSdkCallbacks.AppOpen.OnAdDisplayFailedEvent
MaxSdkCallbacks.AppOpen.OnAdClickedEvent
MaxSdkCallbacks.AppOpen.OnAdRevenuePaidEvent
MaxSdkCallbacks.AppOpen.OnAdHiddenEvent
MaxSdkCallbacks.AppOpen.OnExpiredAdReloadedEvent
```

**Options:**
1. Keep calling MaxSdk directly for app open ads — MeticaSdk wraps MaxSdk, both can coexist.
2. Disable app open ads until MeticaSdk adds support.

---

## Step 8 — Settings / SDK Configuration

### Before (MaxSdk)

```csharp
MaxSdk.SetMuted(true);
bool muted = MaxSdk.IsMuted();
MaxSdk.SetExtraParameter("key", "value");
MaxSdk.ShowMediationDebugger();
MaxSdk.ShowCreativeDebugger();
MaxSdk.SetVerboseLogging(true);
MaxSdk.SetTestDeviceAdvertisingIdentifiers(new[] { "YOUR-TEST-DEVICE-ID" });
```

### After (MeticaSdk)

```csharp
MeticaSdk.Ads.Max.SetMuted(true);
bool muted = MeticaSdk.Ads.Max.IsMuted();
MeticaSdk.Ads.Max.SetExtraParameter("key", "value");
MeticaSdk.Ads.Max.ShowMediationDebugger();
MeticaSdk.SetLogEnabled(true); // ✅ replaces MaxSdk.SetVerboseLogging — call before Initialize

// ❌ Not available in MeticaSdk — call MaxSdk directly if needed:
MaxSdk.ShowCreativeDebugger();
MaxSdk.SetTestDeviceAdvertisingIdentifiers(new[] { "YOUR-TEST-DEVICE-ID" });
```

---

## Step 9 — Error Handling

MaxSdk's `ErrorInfo` is a rich struct with typed codes. MeticaSdk's `MeticaAdError` is a simple message string with an optional ad unit ID.

### Before (MaxSdk)

```csharp
private void OnInterstitialLoadFailed(string adUnitId, MaxSdkBase.ErrorInfo error)
{
    switch (error.Code)
    {
        case MaxSdkBase.ErrorCode.NoFill:
            ScheduleRetry(TimeSpan.FromSeconds(30));
            break;
        case MaxSdkBase.ErrorCode.NetworkError:
            ScheduleRetry(TimeSpan.FromSeconds(5));
            break;
        default:
            Debug.LogError($"[{error.Code}] {error.Message}. " +
                           $"Mediated: {error.MediatedNetworkErrorCode} — {error.MediatedNetworkErrorMessage}");
            break;
    }
}
```

### After (MeticaSdk)

```csharp
private void OnInterstitialLoadFailed(MeticaAdError error)
{
    // No typed error code — only a message string.
    // Implement a simple retry strategy without branching on error type.
    Debug.LogError($"Interstitial load failed ({error.adUnitId}): {error.message}");
    ScheduleRetry(TimeSpan.FromSeconds(30));
}
```

---

## Step 10 — Ad Info / Revenue Reporting

### Before (MaxSdk)

```csharp
private void OnRevenuePaid(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    Analytics.LogAdRevenue(new AdRevenueEvent
    {
        AdUnitId       = adInfo.AdUnitIdentifier,
        NetworkName    = adInfo.NetworkName,
        AdFormat       = adInfo.AdFormat,
        Revenue        = adInfo.Revenue,
        Precision      = adInfo.RevenuePrecision,  // "exact", "estimated", etc.
        CreativeId     = adInfo.CreativeIdentifier,
        DspName        = adInfo.DSPName,
        WaterfallName  = adInfo.WaterfallInfo?.Name,
    });
}
```

### After (MeticaSdk)

```csharp
private void OnRevenuePaid(MeticaAd meticaAd)
{
    Analytics.LogAdRevenue(new AdRevenueEvent
    {
        AdUnitId    = meticaAd.adUnitId,
        NetworkName = meticaAd.networkName,
        AdFormat    = meticaAd.adFormat,
        Revenue     = meticaAd.revenue ?? 0,   // nullable — guard against null
        CreativeId  = meticaAd.creativeId,
        // ❌ No RevenuePrecision, DSPName, WaterfallInfo available
    });
}
```

> **iOS note**: On iOS, `MeticaAd.revenue` may be `null` despite the underlying AppLovin SDK providing a value. This is a known bug in MeticaSdk 2.2.5 caused by `JsonUtility` not supporting `double?`. The Android path is unaffected. Track the fix in the MeticaSdk changelog.

---

## Step 11 — Callback Signature Migration Reference

All MaxSdk callbacks follow `Action<string adUnitId, AdInfo>`. All MeticaSdk callbacks follow `Action<MeticaAd>`. The `adUnitId` is accessed as `meticaAd.adUnitId`.

```csharp
// BEFORE — MaxSdk pattern
void OnLoaded(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    if (adUnitId == bannerAdUnitId) { /* ... */ }
}

// AFTER — MeticaSdk pattern
void OnLoaded(MeticaAd meticaAd)
{
    if (meticaAd.adUnitId == bannerAdUnitId) { /* ... */ }
}
```

### Full event rename table

| MaxSdkCallbacks | MeticaAdsCallbacks | Signature change |
|---|---|---|
| `Banner.OnAdLoadedEvent` | `Banner.OnAdLoadSuccess` | `(string, AdInfo)` → `(MeticaAd)` |
| `Banner.OnAdLoadFailedEvent` | `Banner.OnAdLoadFailed` | `(string, ErrorInfo)` → `(MeticaAdError)` |
| `Banner.OnAdClickedEvent` | `Banner.OnAdClicked` | `(string, AdInfo)` → `(MeticaAd)` |
| `Banner.OnAdRevenuePaidEvent` | `Banner.OnAdRevenuePaid` | `(string, AdInfo)` → `(MeticaAd)` |
| `MRec.OnAdLoadedEvent` | `Mrec.OnAdLoadSuccess` | `(string, AdInfo)` → `(MeticaAd)` |
| `MRec.OnAdLoadFailedEvent` | `Mrec.OnAdLoadFailed` | `(string, ErrorInfo)` → `(MeticaAdError)` |
| `MRec.OnAdClickedEvent` | `Mrec.OnAdClicked` | `(string, AdInfo)` → `(MeticaAd)` |
| `MRec.OnAdRevenuePaidEvent` | `Mrec.OnAdRevenuePaid` | `(string, AdInfo)` → `(MeticaAd)` |
| `Interstitial.OnAdLoadedEvent` | `Interstitial.OnAdLoadSuccess` | `(string, AdInfo)` → `(MeticaAd)` |
| `Interstitial.OnAdLoadFailedEvent` | `Interstitial.OnAdLoadFailed` | `(string, ErrorInfo)` → `(MeticaAdError)` |
| `Interstitial.OnAdDisplayedEvent` | `Interstitial.OnAdShowSuccess` | `(string, AdInfo)` → `(MeticaAd)` |
| `Interstitial.OnAdDisplayFailedEvent` | `Interstitial.OnAdShowFailed` | `(string, ErrorInfo, AdInfo)` → `(MeticaAd, MeticaAdError)` |
| `Interstitial.OnAdHiddenEvent` | `Interstitial.OnAdHidden` | `(string, AdInfo)` → `(MeticaAd)` |
| `Interstitial.OnAdClickedEvent` | `Interstitial.OnAdClicked` | `(string, AdInfo)` → `(MeticaAd)` |
| `Interstitial.OnAdRevenuePaidEvent` | `Interstitial.OnAdRevenuePaid` | `(string, AdInfo)` → `(MeticaAd)` |
| `Rewarded.OnAdLoadedEvent` | `Rewarded.OnAdLoadSuccess` | `(string, AdInfo)` → `(MeticaAd)` |
| `Rewarded.OnAdLoadFailedEvent` | `Rewarded.OnAdLoadFailed` | `(string, ErrorInfo)` → `(MeticaAdError)` |
| `Rewarded.OnAdDisplayedEvent` | `Rewarded.OnAdShowSuccess` | `(string, AdInfo)` → `(MeticaAd)` |
| `Rewarded.OnAdDisplayFailedEvent` | `Rewarded.OnAdShowFailed` | `(string, ErrorInfo, AdInfo)` → `(MeticaAd, MeticaAdError)` |
| `Rewarded.OnAdHiddenEvent` | `Rewarded.OnAdHidden` | `(string, AdInfo)` → `(MeticaAd)` |
| `Rewarded.OnAdClickedEvent` | `Rewarded.OnAdClicked` | `(string, AdInfo)` → `(MeticaAd)` |
| `Rewarded.OnAdReceivedRewardEvent` | `Rewarded.OnAdRewarded` | `(string, Reward, AdInfo)` → `(MeticaAd)` ⚠️ |
| `Rewarded.OnAdRevenuePaidEvent` | `Rewarded.OnAdRevenuePaid` | `(string, AdInfo)` → `(MeticaAd)` |

---

## Known Gaps — No Direct Replacement

The following MaxSdk features have **no MeticaSdk equivalent** in version 2.2.5. Either retain direct MaxSdk calls (both SDKs coexist) or accept the functionality is unavailable.

| Feature | MaxSdk API | Workaround |
|---------|------------|------------|
| App Open Ads | All `AppOpen.*` methods and callbacks | Call MaxSdk directly |
| Reward amount/label | `Reward.Label`, `Reward.Amount` in `OnAdReceivedRewardEvent` | Hardcode or fetch from own backend |
| Dynamic banner/MREC repositioning | `UpdateBannerPosition`, `UpdateMRecPosition` | Destroy + recreate at new position |
| Typed error codes | `ErrorInfo.Code` (ErrorCode enum) | Simple retry; no code-based branching |
| Revenue precision | `AdInfo.RevenuePrecision` | Omit from analytics or assume "estimated" |
| Waterfall info | `AdInfo.WaterfallInfo`, `ErrorInfo.WaterfallInfo` | Unavailable |
| User ID for S2S | `MaxSdk.SetUserId(string)` | Call MaxSdk directly if S2S reward validation needed |
| MAX segmentation | `MaxSdk.SetSegmentCollection(MaxSegmentCollection)` | Call MaxSdk directly |
| Test device IDs | `MaxSdk.SetTestDeviceAdvertisingIdentifiers(string[])` | Call MaxSdk directly in `#if UNITY_EDITOR` / debug builds |
| Verbose logging | `MaxSdk.SetVerboseLogging(bool)` | ✅ Use `MeticaSdk.SetLogEnabled(bool)` |
| Creative debugger | `MaxSdk.ShowCreativeDebugger()` | Call MaxSdk directly |
| Banner/MREC expand/collapse | `OnAdExpandedEvent`, `OnAdCollapsedEvent` | Not detectable |
| Expired ad reload | `OnExpiredAdReloadedEvent` (Interstitial, Rewarded) | Not detectable |
| Ad layout rect | `GetBannerLayout()`, `GetMRecLayout()` | Not queryable |
| Available mediation networks | `GetAvailableMediatedNetworks()` | Call MaxSdk directly |
| Safe area insets | `GetSafeAreaInsets()` | Call MaxSdk directly or use Unity's `Screen.safeArea` |
| App state change | `OnApplicationStateChangedEvent` | Use Unity's `OnApplicationPause(bool)` in a MonoBehaviour |

---

## Quick Find-and-Replace Cheatsheet

Use these as a starting point for a global search-and-replace pass. Review each substitution — some require parameter adjustments.

| Find (MaxSdk) | Replace (MeticaSdk) |
|---|---|
| `MaxSdk.InitializeSdk(` | `MeticaSdk.Initialize(` |
| `MaxSdk.IsInitialized()` | `MeticaSdk.Ads.Max.IsSuccessfullyInitialized()` |
| `MaxSdk.SetHasUserConsent(` | `MeticaSdk.Ads.SetHasUserConsent(` |
| `MaxSdk.SetDoNotSell(` | `MeticaSdk.Ads.SetDoNotSell(` |
| `MaxSdk.HasUserConsent()` | `MeticaSdk.Ads.Max.HasUserConsent()` |
| `MaxSdk.IsUserConsentSet()` | `MeticaSdk.Ads.Max.IsUserConsentSet()` |
| `MaxSdk.ShowMediationDebugger()` | `MeticaSdk.Ads.Max.ShowMediationDebugger()` |
| `MaxSdk.SetMuted(` | `MeticaSdk.Ads.Max.SetMuted(` |
| `MaxSdk.IsMuted()` | `MeticaSdk.Ads.Max.IsMuted()` |
| `MaxSdk.SetVerboseLogging(` | `MeticaSdk.SetLogEnabled(` |
| `MaxSdk.CreateBanner(` | `MeticaSdk.Ads.CreateBanner(` |
| `MaxSdk.ShowBanner(` | `MeticaSdk.Ads.ShowBanner(` |
| `MaxSdk.HideBanner(` | `MeticaSdk.Ads.HideBanner(` |
| `MaxSdk.DestroyBanner(` | `MeticaSdk.Ads.DestroyBanner(` |
| `MaxSdk.LoadBanner(` | `MeticaSdk.Ads.LoadBanner(` |
| `MaxSdk.StartBannerAutoRefresh(` | `MeticaSdk.Ads.StartBannerAutoRefresh(` |
| `MaxSdk.StopBannerAutoRefresh(` | `MeticaSdk.Ads.StopBannerAutoRefresh(` |
| `MaxSdk.SetBannerPlacement(` | `MeticaSdk.Ads.SetBannerPlacement(` |
| `MaxSdk.SetBannerBackgroundColor(` | `MeticaSdk.Ads.SetBannerBackgroundColor(` ⚠️ change Color→hex |
| `MaxSdk.CreateMRec(` | `MeticaSdk.Ads.CreateMrec(` |
| `MaxSdk.ShowMRec(` | `MeticaSdk.Ads.ShowMrec(` |
| `MaxSdk.HideMRec(` | `MeticaSdk.Ads.HideMrec(` |
| `MaxSdk.DestroyMRec(` | `MeticaSdk.Ads.DestroyMrec(` |
| `MaxSdk.LoadMRec(` | `MeticaSdk.Ads.LoadMrec(` |
| `MaxSdk.StartMRecAutoRefresh(` | `MeticaSdk.Ads.StartMrecAutoRefresh(` |
| `MaxSdk.StopMRecAutoRefresh(` | `MeticaSdk.Ads.StopMrecAutoRefresh(` |
| `MaxSdk.LoadInterstitial(` | `MeticaSdk.Ads.LoadInterstitial(` |
| `MaxSdk.IsInterstitialReady(` | `MeticaSdk.Ads.IsInterstitialReady(` |
| `MaxSdk.ShowInterstitial(` | `MeticaSdk.Ads.ShowInterstitial(` |
| `MaxSdk.LoadRewardedAd(` | `MeticaSdk.Ads.LoadRewarded(` |
| `MaxSdk.IsRewardedAdReady(` | `MeticaSdk.Ads.IsRewardedReady(` |
| `MaxSdk.ShowRewardedAd(` | `MeticaSdk.Ads.ShowRewarded(` |
| `MaxSdkBase.AdViewPosition.` | `Metica.Ads.MeticaAdViewPosition.` |
| `MaxSdkCallbacks.Banner.OnAdLoadedEvent` | `MeticaAdsCallbacks.Banner.OnAdLoadSuccess` |
| `MaxSdkCallbacks.Banner.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Banner.OnAdLoadFailed` |
| `MaxSdkCallbacks.Banner.OnAdClickedEvent` | `MeticaAdsCallbacks.Banner.OnAdClicked` |
| `MaxSdkCallbacks.Banner.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Banner.OnAdRevenuePaid` |
| `MaxSdkCallbacks.MRec.OnAdLoadedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadSuccess` |
| `MaxSdkCallbacks.MRec.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadFailed` |
| `MaxSdkCallbacks.MRec.OnAdClickedEvent` | `MeticaAdsCallbacks.Mrec.OnAdClicked` |
| `MaxSdkCallbacks.MRec.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Mrec.OnAdRevenuePaid` |
| `MaxSdkCallbacks.Interstitial.OnAdLoadedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdLoadSuccess` |
| `MaxSdkCallbacks.Interstitial.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdLoadFailed` |
| `MaxSdkCallbacks.Interstitial.OnAdDisplayedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdShowSuccess` |
| `MaxSdkCallbacks.Interstitial.OnAdDisplayFailedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdShowFailed` ⚠️ param order |
| `MaxSdkCallbacks.Interstitial.OnAdHiddenEvent` | `MeticaAdsCallbacks.Interstitial.OnAdHidden` |
| `MaxSdkCallbacks.Interstitial.OnAdClickedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdClicked` |
| `MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Interstitial.OnAdRevenuePaid` |
| `MaxSdkCallbacks.Rewarded.OnAdLoadedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdLoadSuccess` |
| `MaxSdkCallbacks.Rewarded.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdLoadFailed` |
| `MaxSdkCallbacks.Rewarded.OnAdDisplayedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdShowSuccess` |
| `MaxSdkCallbacks.Rewarded.OnAdDisplayFailedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdShowFailed` ⚠️ param order |
| `MaxSdkCallbacks.Rewarded.OnAdHiddenEvent` | `MeticaAdsCallbacks.Rewarded.OnAdHidden` |
| `MaxSdkCallbacks.Rewarded.OnAdClickedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdClicked` |
| `MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent` | `MeticaAdsCallbacks.Rewarded.OnAdRewarded` ⚠️ reward data lost |
| `MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Rewarded.OnAdRevenuePaid` |
| `MaxSdkBase.AdInfo` | `MeticaAd` |
| `MaxSdkBase.ErrorInfo` | `MeticaAdError` |
| `.AdUnitIdentifier` | `.adUnitId` |
| `.NetworkName` | `.networkName` |
| `.AdFormat` | `.adFormat` |
| `.Revenue` | `.revenue ?? 0` |
| `.CreativeIdentifier` | `.creativeId` |
| `.LatencyMillis` | `.latency` |
| `.NetworkPlacement` | `.placementTag` |
