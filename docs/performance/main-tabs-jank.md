# Main-tab jank A/B experiment

This branch is diagnostic, not a confirmed performance fix.

## Source comparison

- Fork baseline: `00c14f4c26e2cee39b2e966887f2edccd8de8893`.
- Official Telegram: `f2908b14133bbffbf7ab04f641ecb5bfaf533242` (12.10.6, build 7112).
- Both destroy the profile fragment after navigating away and update the glass
  background during transitions. These behaviors alone are not fork regressions.
- Unlike the official implementation, the fork's `ViewPagerFixed.removePage`
  recursively walks the outgoing page and calls `RecyclerView.ItemAnimator.endAnimations`.
  The main pager also clears detached page children; that remains unchanged.

## Variants

- A-baseline: existing item-animation cleanup.
- B-experiment: skip that cleanup for the main-tab pager only.
- Both: staging optimization, arm64 native libraries, identical tracing, same package
  `com.exteraless.janktest`, and the same signing key within one workflow run.
- All other ViewPagerFixed consumers retain their cleanup behavior.
- Flags default to false outside the diagnostic workflow. No release behavior changes.
- Trace sections: `MainTabs.bindView`, `ViewPager.endItemAnimations`,
  `ViewPager.removePage`, and `MainTabs.clearPageChildren`.

The isolated package does not replace the installed release. It needs a separate
login and the same relevant appearance settings. Firebase configuration is locally
adapted only to generate resources for the test package; the package is not registered
with Firebase. Push delivery, Maps, and production signing are not test targets.
Do not use this diagnostic APK as a daily client.

## Build

The `Main tabs jank A-B` workflow runs on pushes to `perf/main-tabs-jank`.
It builds both APKs sequentially so their signing key is shared, and uploads
`main-tabs-jank-A-B-arm64`. It never creates a release or posts to Telegram.
Use A and B from the same run; different runs may generate different signing keys.
Existing `LOCAL_PROPERTIES` secrets are optional; no secrets are committed.

## Windows measurement (no Android Studio)

1. Download Android SDK Platform Tools from Google's Android developer site.
2. Enable USB debugging, connect the device, and approve the computer's RSA prompt.
3. Download and extract the two-APK artifact. Copy `main-tabs.pbtxt` into the
   Platform Tools directory along with the APKs.
4. In PowerShell, run `./adb.exe devices`. The device must show `device`, not
   `unauthorized`. Run `./adb.exe install ./A-baseline.apk`.
5. Open the separate test app, log in, wait for initial synchronization, and match
   appearance, refresh rate, and plugin settings. Keep plugins disabled for the
   initial test. Record cold first navigation separately from repeated navigation.
6. Warm up with three cycles: chats, contacts, profile, chats. Return to chats.
7. Start a measurement:

```powershell
./adb.exe shell dumpsys gfxinfo com.exteraless.janktest reset
./adb.exe push ./main-tabs.pbtxt /data/local/tmp/main-tabs.pbtxt
./adb.exe shell perfetto --background --txt -c /data/local/tmp/main-tabs.pbtxt -o /data/misc/perfetto-traces/main-tabs-A.pftrace
```

8. Perform ten cycles by tapping tabs, with about one second between taps.
   Do not swipe or record the screen during this pass. Wait until 65 seconds
   after starting the trace, then collect:

```powershell
./adb.exe shell dumpsys gfxinfo com.exteraless.janktest framestats | Out-File -Encoding utf8 ./A-gfxinfo.txt
./adb.exe pull /data/misc/perfetto-traces/main-tabs-A.pftrace ./main-tabs-A.pftrace
```

9. Update the test app with `./adb.exe install -r ./B-experiment.apk`.
   Repeat steps 6-8 with B filenames. Settings and login should survive.
   Never uninstall the production app or use adb clear against it.
10. Repeat A-B-A with the paired APKs. Keep refresh rate, display brightness,
    power-saving settings, temperature, and charging state consistent.

Check B for missing/duplicated rows, stuck row animations, scroll-position resets,
profile changes not updating, and rapid alternating tab taps. Repeat separately
with glass effects off/on and with the user's plugins enabled only after the
plugin-free test. If the original problem disappears with plugins disabled,
record that before attributing it to navigation.

## Interpretation

Open traces locally in Perfetto or send the trace and gfxinfo files for analysis.
They can contain process names and system scheduling metadata; review before sharing.
Compare frame durations and per-transition jank, not just aggregate percentages.
60 Hz allows roughly 16.7 ms per frame; 120 Hz roughly 8.3 ms.
Look for long cleanup slices aligned with delayed frames in A and absent in B.
If B does not consistently improve repeated transitions, do not ship the change.
Static tests and successful APK compilation cannot establish smoothness on a device.
