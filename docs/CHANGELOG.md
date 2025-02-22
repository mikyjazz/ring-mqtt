## v5.3.0
The primary goal of this update is to address issues with camera/doorbell/intercom notifications that have impacted many users due to changes in the Ring API for push notifications.  This version uses a new upstream ring-client-api that persist the FCM token and hardware ID across restarts which will hopefully address these issues, however, it's important to note that addressing this will likely require users to re-authenticate following the instructions below:

**Steps to fix notifications**
If you have cameras/doorbells/intercoms and are not receiving notifications you will need to follow these steps to re-establish authentication with the Ring API:

1. Stop the addon and verify that it is no longer running
2. In the official Ring App or using the Ring web based dashboard go to the Control Center
3. Click on Authorized Client Devices
4. In the list of authorized devices find and remove all devices associated with ring-mqtt, these devices will have names like the following:
   - ring-mqtt
   - ring-mqtt-addon
   - Device name not found
   - Unknown device
5. Once you have removed all of these devices restart the addon.
6. Review the addon logs and it should show that the existing token is invalid and you need to use the web UI to create a new one
7. Use the addon web UI to authenticate with Ring and re-establish the connection with the Ring API
8. Notifications should now be working!


**New Features**
- Added support to enable/disable motion detection for cameras

**Fixed Bugs**
- Use persistent FCM tokens so that push notifications survive restarts
- Remove doubled-up devices in Ring Control Center, including one "unknown device" when authenticating
- Fix random crash of go2rtc process which impacted some users (fixed by bumping go2rtc to v1.5.0)

**Dependency Updates**
- ring-client-api v11.8.0-beta.0
- werift v0.18.3
- rxjs v7.8.1
- go2rtc v1.5.0
- s6-overlay v3.1.5.0

## v5.2.2
**Fixed Bugs**
- Update ring-client-api to v11.7.5 which should fix issues with web token generation caused by changes in Ring API.

## v5.2.1
**Fixed Bugs**
- Update ring-client-api to v11.7.4 which should fix an issue with motion snapshots not working due to a change in push notification data sent by the Ring API.
- Suppress spurious "Lost subscription to ding" messages in log which led to confusion for users.

**Other Changes**
- Tamper sensors now use "tamper" device class in Home Assistant vs the generic "problem" device class used previously.

**Dependency Updates**
- Bump go2rtc to v1.3.1

## v5.2.0
**New Features**
- Basic support for Ring Intercom, the following features are supported:
  - Ding state - Simple binary sensor, stays "on" for 20 seconds after ding
  - Lock state - Unlock command is supported and also triggers on unlock from Ring app.  Stays in unlocked state for 5 seconds before reverting to locked state.
  - Battery status
  - Wifi status
- Keypad proximity sensor is now exposed as a motion sensor (only tested with Keypad Gen2 model)
- Implement improved Home Assistant behavior for MQTT thermostats when switching between auto and heat/cool modes.  Requires Home Assistant 2023.3 release or later, but provides much improved behavior which should mirror that of thermostats connected directly via Z-wave.
- Based on requests, camera motion and ding event "on" duration can now be configured on a per-device basis.  For now, the default duration remains 180 seconds, which is based on the ding expire time property sent as part of the ding event in the Ring API and also aligned with first-generation motion sensors which would stay in "on" state for 180 seconds after any detected motion.  However, many users have requested a shorter "on" duration and second-generation motion sensors now use 20 seconds, so this new feature provides flexibility for those users.  Based on feedback, future versions may use the shorter "on" duration by default.

**Fixed Bugs**
- Fixed an issue with generic binary sensors which caused them to fail automatic discovery in Home Assistant.

**Dependency Updates**
- Bump go2rtc to v1.2.0
- Bump werift to v0.18.2 with some WebRTC stun negotiation improvements.  May fix livestream failures for users with more complex network setups.
- Bump ring-client-api to v11.7.2
- Bump s6-overlay to v3.1.4.1

## v5.1.3
**Fixed Bugs**
- Don't crash on codec mismatch.  This is caused by the fact that Ring has started rolling out support for the HEVC/H.265 video encoding format on some devices and cameras, however, this format still has many issues and incompatibilities in downstream browsers and devices.  For now the suggestion for ring-mqtt users is to enable [Legacy Video Mode](https://support.ring.com/hc/en-us/articles/4417503172116-Legacy-Video-Mode-) for any cameras that are using this codec as default.

**Other Changes**
- Include RTX as part of WebRTC codec negotiation which can improve robustness and reduce artifacting of the livestream in cases where there is minor UDP packet loss.
- Disable WebRTC port on internal go2rtc instance.

**Dependency Updates**
- Bump go2rtc version to v1.1.2
- Bump s6-overlay to v3.1.3.0

## v5.1.2 (re-publish of v5.1.1)
**Fixed Bugs**  
- Fix crash on Chime Pro (1st Generation) models

## v5.1.0
After several releases focused on stability and minor bug fixes this release includes significant internal changes and a few new features.

**!!!!! WARNING !!!!!**  
Starting with 5.1.x all backwards compatibiltiy with prior 4.x style configuration options has been removed.  Upgrades from 4.x versions are still possible but will require manual conversion of any legacy configuration methods and options.  Upgrades from 5.0.x versions should not require any changes.

**New Features**  
- Added ability to refine event stream to only motion events where a person is detected
- Option to select transcoded vs raw video for event stream (this also changes the URL for scripting automatic download of recordings):
  - Raw video (default) - This video is exactly as it was recorded by the camera and is the same as previous versions of ring-mqtt
  - Transcoded video - This is the same as selecting to share/download video from the Ring app or web dashboard.  This video includes the Ring logo and timestamps and may include supplemental pre-roll video for supported devices.  Note that switching from a raw to transcoded event selection can take 10-15 seconds as transcoded videos are created by Ring on-demand so ring-mqtt must wait for the Ring servers to process the video and return the URL.
- New camera models should now display with correct model/features
- Improved support for cameras with dual batteries.  The BatteryLevel attribute always reports the level of currently active battery but the level of both batteries is individually available via the batteryLife and batteryLife2 attributes.
- Switch to enable/disable the Chime Pro nightlight.  The current nightlight on/off state can be determined via attribute.

**Other Changes**  
- Reduced average live stream startup time by several hundred milliseconds with the following changes:
  - Switched from rtsp-simple-server to go2rtc as the core streaming engine which provides slightly faster stream startup and opens the door for future feature enhancements.  This switch is expected to be transparent for users, but please report any issues.
  - Cameras now allocate a dedicated worker thread for live streaming vs previous versions which used a pool of worker threads based on the number of processor cores detected.  This simplifies the live stream code and leads to faster stream startup and, hopefully, more reliable recovery from various error conditions.
  - The recommended configuration for streaming setup in Home Assistant is now to use the go2rtc addon with RTSPtoWebRTC integration.  This provides fast stream startup and shutdown and low-latency live vieweing (typically <1 second latency).

**Dependency Updates**  
- Replaced problematic colors package with chalk for colorizing debug log output
- Bump ring-client-api to 11.7.1 (various fixes and support for newer cameras)
- Bump other dependent packages to latest versions
- Migrated project codebase from CommonJS to ESM.  As this project is not a library this should have zero impact for users, but it does ease ongoing maintenance by enabling the ability to pull in newer versions of various dependent packages that have also moved to pure ESM.

### Changes prior to v5.1.0 are tracked in the [historical changelog](https://github.com/tsightler/ring-mqtt/blob/main/docs/CHANGELOG-HIST.md)
