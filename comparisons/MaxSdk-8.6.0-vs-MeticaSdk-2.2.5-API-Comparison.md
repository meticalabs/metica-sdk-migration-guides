# MaxSdk vs MeticaSdk - Public API Comparison

| SDK | Version |
|-----|---------|
| **AppLovin MAX Unity Plugin** | 8.6.0 |
| **MeticaSdk Unity Plugin** | 2.2.5 |

> Generated on: 2026-03-12
> MaxSdk source: `SDK comparison/AppLovin SDK builds/AppLovin-MAX-Unity-Plugin-8.6.0-Android-13.6.0-iOS-13.6.0/`
> MeticaSdk source: `SDK comparison/Metica SDK builds/MeticaSDK-2.2.5/` (extracted from MeticaSDK-2.2.5.unitypackage)

---

## Summary Findings

MeticaSdk 2.2.5 covers the core ad operations for Banners, MRECs, Interstitials, and Rewarded formats with near-complete method parity. However, App Open Ads are entirely absent, several debug/testing utilities are missing, callback signatures differ structurally, and data richness in ad info and error objects is reduced compared to MaxSdk.

### Overall Parity

| Category | Parity |
|----------|--------|
| Banners | ⚠️ Partial — core load/show/hide covered; no `UpdateBannerPosition`, no `GetBannerLayout`, no expand/collapse callbacks |
| MRECs | ⚠️ Partial — same gaps as Banners: no `UpdateMRecPosition`, no `GetMRecLayout`, no expand/collapse callbacks |
| Interstitials | ✅ Full — all core methods present; minor rename (`OnAdDisplayed` → `OnAdShowSuccess`) |
| Rewarded Ads | ⚠️ Partial — core methods covered; `OnAdRewarded` drops reward amount/label data present in MaxSdk's `Reward` struct |
| Callbacks / Events | ⚠️ Partial — all core events covered; missing `OnAdExpanded/Collapsed`, `OnAdReviewCreativeIdGenerated`, `OnExpiredAdReloaded`, `OnApplicationStateChanged`; signature changed from `Action<string, AdInfo>` to `Action<MeticaAd>` |
| Privacy / Consent | ✅ Full — `SetHasUserConsent`, `SetDoNotSell` present; getter methods accessible via `Max` accessor |
| App Open Ads | ❌ Not supported — entire ad format absent |
| Debugging / Testing | ⚠️ Partial — `ShowMediationDebugger` + `SetLogEnabled` available; no `ShowCreativeDebugger`, no test device IDs, no logging getter |
| Segmentation | ❌ Not supported — `MaxSdk.SetSegmentCollection` has no MeticaSdk equivalent |
| Ad Info / Error data richness | ⚠️ Partial — `MeticaAd` covers `adUnitId`, `networkName`, `adFormat`, `revenue`, `creativeId`, `latency`, `placementTag`; missing `RevenuePrecision`, `WaterfallInfo`, `DSPName/Id`, `PlacementName`; `MeticaAdError` has only `message` string — no error codes, no waterfall info |

### Key API Pattern Differences

- **Namespace/accessor shift**: MaxSdk exposes everything as `MaxSdk.*` static methods. MeticaSdk routes through `MeticaSdk.Ads.*` (for ad operations), `MeticaSdk.Ads.Max.*` (for AppLovin-specific calls like `ShowMediationDebugger`, `IsMuted`, `HasUserConsent`), and `MeticaAdsCallbacks.*` (for events).
- **Initialization**: `MaxSdk.InitializeSdk(string[] adUnitIds)` takes an optional ad-unit ID array and fires `MaxSdkCallbacks.OnSdkInitializedEvent`. MeticaSdk uses `MeticaSdk.Initialize(MeticaInitConfig, Action<MeticaInitResponse>)` or `InitializeAsync` with a config object — richer but structurally different.
- **Callback signatures**: All MaxSdk callbacks are `Action<string adUnitId, MaxSdkBase.AdInfo>` (or `Action<string, ErrorInfo>`). MeticaSdk callbacks are `Action<MeticaAd>` or `Action<MeticaAdError>` — the ad unit ID is embedded in `MeticaAd.adUnitId` rather than passed as a separate argument. Code that pattern-matches on the adUnitId parameter must be refactored.
- **Event naming**: MaxSdk uses `OnAdLoadedEvent`/`OnAdDisplayedEvent`; MeticaSdk uses `OnAdLoadSuccess`/`OnAdShowSuccess`. Show failures are `OnAdDisplayFailedEvent` → `OnAdShowFailed`.
- **Background color**: `SetBannerBackgroundColor(string, Color)` takes a `UnityEngine.Color` in MaxSdk; MeticaSdk takes a `string hexColorCode`.
- **Rewarded method names**: `LoadRewardedAd` / `IsRewardedAdReady` / `ShowRewardedAd` → `LoadRewarded` / `IsRewardedReady` / `ShowRewarded`.
- **Settings access**: `MaxSdk.SetMuted`, `IsMuted`, `SetExtraParameter` become `MeticaSdk.Ads.Max.SetMuted`, `IsMuted`, `SetExtraParameter`.

### Critical Gaps for Migration

1. **App Open Ads** — `LoadAppOpenAd`, `IsAppOpenAdReady`, `ShowAppOpenAd`, and all associated callbacks are absent. Games using app-open format must keep MaxSdk or find an alternative.

2. **Rewarded reward data lost** — `MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent` passes `MaxSdkBase.Reward` (with `Label` and `Amount`). MeticaSdk's `OnAdRewarded` passes only `MeticaAd` with no reward fields. Any game logic that reads `reward.Label` or `reward.Amount` to decide what to grant will break silently.

3. **No dynamic position update for banners/MRECs** — `UpdateBannerPosition` and `UpdateMRecPosition` are missing. Games that reposition banners/MRECs at runtime (e.g. on screen orientation change, or when other UI elements shift) have no equivalent — they must destroy and recreate the ad view instead.

4. **No user ID or segmentation** — `MaxSdk.SetUserId(string)` (used for S2S reward validation and analytics) and `MaxSdk.SetSegmentCollection(MaxSegmentCollection)` (for targeting segments) have no MeticaSdk equivalents. S2S reward validation and MAX segmentation features cannot be used.

5. **No test device / logging controls** — `SetTestDeviceAdvertisingIdentifiers`, `SetVerboseLogging`, `SetCreativeDebuggerEnabled`, `SetExceptionHandlerEnabled` are absent. QA workflows that rely on test ads on real devices or verbose MAX logs require direct MaxSdk calls.

6. **Reduced error diagnostics** — `MeticaAdError` has only a `message: string`. MaxSdk's `ErrorInfo` includes a typed `ErrorCode` enum, `MediatedNetworkErrorCode`, `MediatedNetworkErrorMessage`, and full `WaterfallInfo`. Error handling code that branches on `ErrorInfo.Code` must be completely rewritten with string parsing or omitted.

---

## 1. MaxSdk Functions That Can Be Replaced With MeticaSdk

### Initialization

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.InitializeSdk(string[] adUnitIds = null)` | `MeticaSdk.Initialize(MeticaInitConfig, Action<MeticaInitResponse>)` | Config object replaces ad unit ID array; callback replaces `OnSdkInitializedEvent` |
| `MaxSdk.InitializeSdk(string[] adUnitIds)` (async pattern) | `MeticaSdk.InitializeAsync(MeticaInitConfig)` | Async variant available |
| `MaxSdk.IsInitialized()` | `MeticaSdk.Ads.Max.IsSuccessfullyInitialized()` | Moved to `Max` accessor under `Ads` |

### Privacy

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.SetHasUserConsent(bool)` | `MeticaSdk.Ads.SetHasUserConsent(bool)` | Same semantics, different namespace |
| `MaxSdk.HasUserConsent()` | `MeticaSdk.Ads.Max.HasUserConsent()` | Read-back via `Max` accessor |
| `MaxSdk.IsUserConsentSet()` | `MeticaSdk.Ads.Max.IsUserConsentSet()` | Via `Max` accessor |
| `MaxSdk.SetDoNotSell(bool)` | `MeticaSdk.Ads.SetDoNotSell(bool)` | Same semantics |
| `MaxSdk.GetSdkConfiguration()` (ConsentFlowUserGeography) | `MeticaSdk.Ads.Max.GetConsentFlowUserGeography()` | Partial — decomposed |
| `MaxSdk.GetSdkConfiguration()` (CountryCode) | `MeticaSdk.Ads.Max.CountryCode()` | Partial — decomposed |
| `MaxSdk.GetSdkConfiguration()` (ConsentDialogState) | `MeticaSdk.Ads.Max.ConsentDialogState()` | Partial — decomposed |
| CMP flow for existing user (via AppLovin CMP service) | `MeticaSdk.Ads.Max.ShowCmpForExistingUser()` | Available via `Max` accessor |

### Banners

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.CreateBanner(string, AdViewConfiguration)` | `MeticaSdk.Ads.CreateBanner(string, MeticaAdViewConfiguration)` | Same; `MeticaAdViewConfiguration` mirrors `AdViewConfiguration` |
| `MaxSdk.LoadBanner(string)` | `MeticaSdk.Ads.LoadBanner(string)` | Identical |
| `MaxSdk.ShowBanner(string)` | `MeticaSdk.Ads.ShowBanner(string)` | Identical |
| `MaxSdk.HideBanner(string)` | `MeticaSdk.Ads.HideBanner(string)` | Identical |
| `MaxSdk.DestroyBanner(string)` | `MeticaSdk.Ads.DestroyBanner(string)` | Identical |
| `MaxSdk.StartBannerAutoRefresh(string)` | `MeticaSdk.Ads.StartBannerAutoRefresh(string)` | Identical |
| `MaxSdk.StopBannerAutoRefresh(string)` | `MeticaSdk.Ads.StopBannerAutoRefresh(string)` | Identical |
| `MaxSdk.SetBannerPlacement(string, string)` | `MeticaSdk.Ads.SetBannerPlacement(string, string?)` | Same |
| `MaxSdk.SetBannerWidth(string, float)` | `MeticaSdk.Ads.SetBannerWidth(string, float)` | Same |
| `MaxSdk.SetBannerBackgroundColor(string, Color)` | `MeticaSdk.Ads.SetBannerBackgroundColor(string, string hexColorCode)` | Parameter type changed: `UnityEngine.Color` → hex string |
| `MaxSdk.SetBannerExtraParameter(string, string, string)` | `MeticaSdk.Ads.SetBannerExtraParameter(string, string, string?)` | Same |
| `MaxSdk.SetBannerLocalExtraParameter(string, string, object)` | `MeticaSdk.Ads.SetBannerLocalExtraParameter(string, string, object?)` | Same |
| `MaxSdk.SetBannerCustomData(string, string)` | `MeticaSdk.Ads.SetBannerCustomData(string, string?)` | Same |

### MRECs

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.CreateMRec(string, AdViewConfiguration)` | `MeticaSdk.Ads.CreateMrec(string, MeticaAdViewConfiguration)` | Renamed: `MRec` → `Mrec` |
| `MaxSdk.LoadMRec(string)` | `MeticaSdk.Ads.LoadMrec(string)` | Renamed |
| `MaxSdk.ShowMRec(string)` | `MeticaSdk.Ads.ShowMrec(string)` | Renamed |
| `MaxSdk.HideMRec(string)` | `MeticaSdk.Ads.HideMrec(string)` | Renamed |
| `MaxSdk.DestroyMRec(string)` | `MeticaSdk.Ads.DestroyMrec(string)` | Renamed |
| `MaxSdk.StartMRecAutoRefresh(string)` | `MeticaSdk.Ads.StartMrecAutoRefresh(string)` | Renamed |
| `MaxSdk.StopMRecAutoRefresh(string)` | `MeticaSdk.Ads.StopMrecAutoRefresh(string)` | Renamed |
| `MaxSdk.SetMRecPlacement(string, string)` | `MeticaSdk.Ads.SetMrecPlacement(string, string?)` | Renamed |
| `MaxSdk.SetMRecCustomData(string, string)` | `MeticaSdk.Ads.SetMrecCustomData(string, string?)` | Renamed |
| `MaxSdk.SetMRecExtraParameter(string, string, string)` | `MeticaSdk.Ads.SetMrecExtraParameter(string, string, string?)` | Renamed |
| `MaxSdk.SetMRecLocalExtraParameter(string, string, object)` | `MeticaSdk.Ads.SetMrecLocalExtraParameter(string, string, object?)` | Renamed |

### Interstitials

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.LoadInterstitial(string)` | `MeticaSdk.Ads.LoadInterstitial(string)` | Identical |
| `MaxSdk.IsInterstitialReady(string)` | `MeticaSdk.Ads.IsInterstitialReady(string)` | Identical |
| `MaxSdk.ShowInterstitial(string, string?, string?)` | `MeticaSdk.Ads.ShowInterstitial(string, string?, string?)` | Identical |
| `MaxSdk.SetInterstitialExtraParameter(string, string, string)` | `MeticaSdk.Ads.SetInterstitialExtraParameter(string, string, string?)` | Same |
| `MaxSdk.SetInterstitialLocalExtraParameter(string, string, object)` | `MeticaSdk.Ads.SetInterstitialLocalExtraParameter(string, string, object?)` | Same |

### Rewarded Ads

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.LoadRewardedAd(string)` | `MeticaSdk.Ads.LoadRewarded(string)` | Renamed: `RewardedAd` → `Rewarded` |
| `MaxSdk.IsRewardedAdReady(string)` | `MeticaSdk.Ads.IsRewardedReady(string)` | Renamed |
| `MaxSdk.ShowRewardedAd(string, string?, string?)` | `MeticaSdk.Ads.ShowRewarded(string, string?, string?)` | Renamed |
| `MaxSdk.SetRewardedAdExtraParameter(string, string, string)` | `MeticaSdk.Ads.SetRewardedAdExtraParameter(string, string, string?)` | Same |
| `MaxSdk.SetRewardedAdLocalExtraParameter(string, string, object)` | `MeticaSdk.Ads.SetRewardedAdLocalExtraParameter(string, string, object?)` | Same |

### Callbacks / Events

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| **Banner Events** | | |
| `MaxSdkCallbacks.Banner.OnAdLoadedEvent` `Action<string, AdInfo>` | `MeticaAdsCallbacks.Banner.OnAdLoadSuccess` `Action<MeticaAd>` | adUnitId embedded in `MeticaAd.adUnitId` |
| `MaxSdkCallbacks.Banner.OnAdLoadFailedEvent` `Action<string, ErrorInfo>` | `MeticaAdsCallbacks.Banner.OnAdLoadFailed` `Action<MeticaAdError>` | Error model simplified |
| `MaxSdkCallbacks.Banner.OnAdClickedEvent` `Action<string, AdInfo>` | `MeticaAdsCallbacks.Banner.OnAdClicked` `Action<MeticaAd>` | Same semantics |
| `MaxSdkCallbacks.Banner.OnAdRevenuePaidEvent` `Action<string, AdInfo>` | `MeticaAdsCallbacks.Banner.OnAdRevenuePaid` `Action<MeticaAd>` | Same semantics |
| **MREC Events** | | |
| `MaxSdkCallbacks.MRec.OnAdLoadedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadSuccess` | Same semantics, renamed |
| `MaxSdkCallbacks.MRec.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Mrec.OnAdLoadFailed` | Same semantics |
| `MaxSdkCallbacks.MRec.OnAdClickedEvent` | `MeticaAdsCallbacks.Mrec.OnAdClicked` | Same semantics |
| `MaxSdkCallbacks.MRec.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Mrec.OnAdRevenuePaid` | Same semantics |
| **Interstitial Events** | | |
| `MaxSdkCallbacks.Interstitial.OnAdLoadedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdLoadSuccess` | Renamed |
| `MaxSdkCallbacks.Interstitial.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdLoadFailed` | Renamed |
| `MaxSdkCallbacks.Interstitial.OnAdDisplayedEvent` `Action<string, AdInfo>` | `MeticaAdsCallbacks.Interstitial.OnAdShowSuccess` `Action<MeticaAd>` | Renamed (`Displayed` → `ShowSuccess`) |
| `MaxSdkCallbacks.Interstitial.OnAdDisplayFailedEvent` `Action<string, ErrorInfo, AdInfo>` | `MeticaAdsCallbacks.Interstitial.OnAdShowFailed` `Action<MeticaAd, MeticaAdError>` | Renamed; parameter order changed |
| `MaxSdkCallbacks.Interstitial.OnAdHiddenEvent` | `MeticaAdsCallbacks.Interstitial.OnAdHidden` | Renamed |
| `MaxSdkCallbacks.Interstitial.OnAdClickedEvent` | `MeticaAdsCallbacks.Interstitial.OnAdClicked` | Renamed |
| `MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Interstitial.OnAdRevenuePaid` | Renamed |
| **Rewarded Events** | | |
| `MaxSdkCallbacks.Rewarded.OnAdLoadedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdLoadSuccess` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdLoadFailedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdLoadFailed` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdDisplayedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdShowSuccess` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdDisplayFailedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdShowFailed` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdHiddenEvent` | `MeticaAdsCallbacks.Rewarded.OnAdHidden` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdClickedEvent` | `MeticaAdsCallbacks.Rewarded.OnAdClicked` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent` | `MeticaAdsCallbacks.Rewarded.OnAdRevenuePaid` | Renamed |
| `MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent` `Action<string, Reward, AdInfo>` | `MeticaAdsCallbacks.Rewarded.OnAdRewarded` `Action<MeticaAd>` | ⚠️ Reward amount/label **not passed** — see Critical Gaps |
| **SDK-level Events** | | |
| `MaxSdkCallbacks.OnSdkInitializedEvent` `Action<SdkConfiguration>` | `MeticaSdk.Initialize(config, Action<MeticaInitResponse> callback)` | Not an event; replaced by init callback pattern |

### Settings / Configuration

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.SetMuted(bool)` | `MeticaSdk.Ads.Max.SetMuted(bool)` | Moved to `Max` accessor |
| `MaxSdk.IsMuted()` | `MeticaSdk.Ads.Max.IsMuted()` | Moved to `Max` accessor |
| `MaxSdk.SetExtraParameter(string, string)` | `MeticaSdk.Ads.Max.SetExtraParameter(string, string?)` | Moved to `Max` accessor |
| `MaxSdk.SetVerboseLogging(bool)` | `MeticaSdk.SetLogEnabled(bool)` | Renamed; call before `Initialize` |

### Debugging / Testing

| MaxSdk | MeticaSdk | Notes |
|--------|-----------|-------|
| `MaxSdk.ShowMediationDebugger()` | `MeticaSdk.Ads.Max.ShowMediationDebugger()` | Moved to `Max` accessor |
| AppLovin CMP consent flow (via `MaxCmpService`) | `MeticaSdk.Ads.Max.ShowCmpForExistingUser()` | Available |

### Ad Info Models

| MaxSdk (`MaxSdkBase.AdInfo`) | MeticaSdk (`MeticaAd`) | Notes |
|------------------------------|------------------------|-------|
| `AdUnitIdentifier` (string) | `adUnitId` (string) | Same |
| `NetworkName` (string) | `networkName` (string?) | Same |
| `AdFormat` (string) | `adFormat` (string?) | Same |
| `Revenue` (double) | `revenue` (double?) | MaxSdk non-nullable; MeticaSdk nullable |
| `CreativeIdentifier` (string) | `creativeId` (string?) | Renamed |
| `LatencyMillis` (long) | `latency` (long?) | Renamed (unit implicit) |
| `NetworkPlacement` (string) | `placementTag` (string?) | Renamed |

### Error Models

| MaxSdk (`MaxSdkBase.ErrorInfo`) | MeticaSdk (`MeticaAdError`) | Notes |
|---------------------------------|-----------------------------|-------|
| `Message` (string) | `message` (string) | Same |
| — | `adUnitId` (string?) | MeticaSdk adds adUnitId to error |

---

## 2. MaxSdk Functionality Missing From MeticaSdk

### Ad Formats

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| **App Open Ads** | `MaxSdk.LoadAppOpenAd(string)`, `IsAppOpenAdReady(string)`, `ShowAppOpenAd(string, string?, string?)`, `SetAppOpenAdExtraParameter(string, string, string)`, `SetAppOpenAdLocalExtraParameter(string, string, object)` | Entire ad format not supported; all 5 `MaxSdkCallbacks.AppOpen.*` events also absent |

### Banner Features

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Dynamic position update | `MaxSdk.UpdateBannerPosition(string, AdViewPosition)`, `UpdateBannerPosition(string, float, float)` | Banner position can only be set at creation; runtime repositioning requires destroy+recreate |
| Layout query | `MaxSdk.GetBannerLayout(string)` → `Rect` | Cannot query absolute screen position of banner |
| Expand/collapse events | `MaxSdkCallbacks.Banner.OnAdExpandedEvent`, `OnAdCollapsedEvent` | Cannot detect banner expand/collapse state |
| Ad Review Creative ID | `MaxSdkCallbacks.Banner.OnAdReviewCreativeIdGeneratedEvent` | No creative review workflow |

### MREC Features

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Dynamic position update | `MaxSdk.UpdateMRecPosition(string, AdViewPosition)`, `UpdateMRecPosition(string, float, float)` | Same gap as banner |
| Layout query | `MaxSdk.GetMRecLayout(string)` → `Rect` | Cannot query position |
| Expand/collapse events | `MaxSdkCallbacks.MRec.OnAdExpandedEvent`, `OnAdCollapsedEvent` | No expand/collapse state |
| Ad Review Creative ID | `MaxSdkCallbacks.MRec.OnAdReviewCreativeIdGeneratedEvent` | No creative review workflow |

### Interstitial Features

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Expired ad reloaded event | `MaxSdkCallbacks.Interstitial.OnExpiredAdReloadedEvent` `Action<string, AdInfo, AdInfo>` | Cannot detect auto-reload of expired interstitial |
| Ad Review Creative ID | `MaxSdkCallbacks.Interstitial.OnAdReviewCreativeIdGeneratedEvent` | No creative review |

### Rewarded Ad Features

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Reward amount and label | `MaxSdkCallbacks.Rewarded.OnAdReceivedRewardEvent` → `Action<string, MaxSdkBase.Reward, AdInfo>` (`Reward.Label`, `Reward.Amount`) | MeticaSdk `OnAdRewarded` passes only `MeticaAd`; reward type/amount data is lost |
| Expired ad reloaded event | `MaxSdkCallbacks.Rewarded.OnExpiredAdReloadedEvent` `Action<string, AdInfo, AdInfo>` | Cannot detect auto-reload |
| Ad Review Creative ID | `MaxSdkCallbacks.Rewarded.OnAdReviewCreativeIdGeneratedEvent` | No creative review |

### Privacy / Consent

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| DoNotSell getter | `MaxSdk.IsDoNotSell()` | No read-back of DoNotSell flag |
| DoNotSell set-check | `MaxSdk.IsDoNotSellSet()` | Cannot check if flag was explicitly set |
| Full SdkConfiguration | `MaxSdk.GetSdkConfiguration()` → `SdkConfiguration` | No single structured config object; individual fields accessed via `Max.*` |

### Debugging / Testing

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Creative debugger | `MaxSdk.ShowCreativeDebugger()` | Not exposed in MeticaSdk |
| Test device identifiers | `MaxSdk.SetTestDeviceAdvertisingIdentifiers(string[])` | Cannot enable test ads on real devices via MeticaSdk |
| Verbose logging getter | `MaxSdk.IsVerboseLoggingEnabled()` | No getter; `MeticaSdk.SetLogEnabled(bool)` setter is available |
| Creative debugger enable | `MaxSdk.SetCreativeDebuggerEnabled(bool)` | Not accessible |
| Exception handler toggle | `MaxSdk.SetExceptionHandlerEnabled(bool)` | Not accessible |

### SDK Information

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Ad value lookup | `MaxSdk.GetAdValue(string adUnitId, string key)` → `string` | Cannot query arbitrary ad key-value data |
| Available networks list | `MaxSdk.GetAvailableMediatedNetworks()` → `List<MediatedNetworkInfo>` | Cannot enumerate mediation networks at runtime |
| Safe area insets | `MaxSdk.GetSafeAreaInsets()` → `SafeAreaInsets` | Cannot query native safe area for custom layout |

### Settings

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| Verbose logging getter | `MaxSdk.IsVerboseLoggingEnabled()` | Setter `MeticaSdk.SetLogEnabled(bool)` is available; no getter |
| Creative debugger | `MaxSdk.SetCreativeDebuggerEnabled(bool)` | No equivalent |
| Exception handler | `MaxSdk.SetExceptionHandlerEnabled(bool)` | No equivalent |
| Test device IDs | `MaxSdk.SetTestDeviceAdvertisingIdentifiers(string[])` | No equivalent |

### Event Tracking

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| AppLovin event tracking | `MaxSdk.TrackEvent(string name, IDictionary<string, string> parameters)` | MeticaSdk has its own `MeticaSdk.Events` subsystem; no bridge to MaxSdk's event tracking |

### Segmentation

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| MAX segment collection | `MaxSdk.SetSegmentCollection(MaxSegmentCollection)` | No equivalent; `MaxSegmentCollection` and associated targeting features unavailable |
| User identifier for S2S | `MaxSdk.SetUserId(string)` | No user ID forwarding to MAX; affects S2S reward validation and per-user analytics |

### Callback Differences

| Missing Feature | MaxSdk API | Details |
|-----------------|------------|---------|
| App lifecycle event | `MaxSdkCallbacks.OnApplicationStateChangedEvent` `Action<bool isPaused>` | Cannot receive app pause/resume notifications via MeticaSdk |
| Per-adUnit callback ID | All MaxSdk callbacks `Action<string adUnitId, AdInfo>` | MeticaSdk callbacks omit adUnitId as separate parameter; it's in `MeticaAd.adUnitId` |

### Data Models

| Missing Feature | MaxSdk Type | Details |
|-----------------|-------------|---------|
| Revenue precision | `MaxSdkBase.AdInfo.RevenuePrecision` (string) | Whether revenue is exact, estimated, or publisher-defined |
| Waterfall info | `MaxSdkBase.AdInfo.WaterfallInfo` (`WaterfallInfo`) | Full mediation waterfall with network response details per network |
| DSP identification | `MaxSdkBase.AdInfo.DSPName`, `DSPIdentifier` | Demand-side platform name/ID not forwarded |
| Placement name | `MaxSdkBase.AdInfo.PlacementName` | Ad placement name not in MeticaAd |
| Typed error code | `MaxSdkBase.ErrorInfo.Code` (`ErrorCode` enum) | MeticaAdError only has a `message` string; no structured error code for branching |
| Mediated network error | `MaxSdkBase.ErrorInfo.MediatedNetworkErrorCode`, `MediatedNetworkErrorMessage` | Underlying network error details not surfaced |
| Waterfall info on error | `MaxSdkBase.ErrorInfo.WaterfallInfo` | Load failure waterfall trace unavailable |
| Reward data | `MaxSdkBase.Reward` (`Label`, `Amount`) | Only passed in rewarded callback; no MeticaSdk equivalent |
| Mediated network list | `MaxSdkBase.MediatedNetworkInfo` | Returned by `GetAvailableMediatedNetworks()`; unavailable |
| Safe area insets | `MaxSdkBase.SafeAreaInsets` | Not exposed |
| Segment collection | `MaxSegmentCollection` | No equivalent type or API |
