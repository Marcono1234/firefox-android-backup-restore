# Backup & restore of Firefox app data

This tutorial explains how one can create and restore a backup of the full
Android app data, fully offline.

While the Firefox app itself does not support `adb backup` due to
`allowBackups="false"` ([bug 1808763](https://bugzilla.mozilla.org/show_bug.cgi?id=1808763)),
we can still access all data by connecting through a Firefox-specific debugger.

Repository: https://github.com/Rob--W/firefox-android-backup-restore

## Requirements

All you need is `adb`, and a desktop Firefox instance to use `about:debugging`:

- Tutorial with screenshots of using `about:debugging`: https://extensionworkshop.com/documentation/develop/developing-extensions-for-firefox-for-android/#debug-your-extension
- Documentation of `about:debugging`: https://firefox-source-docs.mozilla.org/devtools-user/about_colon_debugging/index.html#connecting-to-a-remote-device
- Optional, adb over Wi-Fi (instead of USB): https://firefox-source-docs.mozilla.org/devtools-user/about_colon_debugging/index.html#about-colon-debugging-connecting-to-android-over-wi-fi

## Relevant files

The data of Android apps are usually stored in internal storage, private to the
app, inaccessible to other apps and adb. External storage at `/sdcard/` can be
accessed through `adb`, but we need to use a special subdirectory to make sure
that the app can access it without requiring special storage permissions.

Examples of paths for Firefox (app ID `org.mozilla.firefox`):

- Private app data: `/data/user/0/org.mozilla.firefox/`
- Public directory: `/sdcard/Android/data/org.mozilla.firefox/`

The private app data directory contains several directories and files, the most
important ones being:

- `shared_prefs/` - Firefox UI settings and customizations.
- `files/` - Firefox profile directory, with website data (cookies etc).
- `databases/` - Tabs, collections, autofill, logins & passwords, and more.

The following are also relevant for consistency, but not critical:

- `cache/` - Caches, including thumbnails and favicons.
- `nimbus_data/` - Feature flags. Optional, but can change behavior.
- `no_backup/` - Includes `androidx.work` database, e.g. add-on update checks.
- `glean_data/` - Telemetry data (very old profiles may also have `telemetry/`).

## Backup

Optionally before performing the backup clear the app cache of the Firefox app
to reduce the size of the backup.

1. To backup, prepare to receive the backup data in the terminal:
    ```sh
    adb reverse tcp:12101 tcp:12101
    nc -l -s 127.0.0.1 -p 12101 > firefox-android-backup.tar.gz
    ```

    On MacOS you might have to use `nc -l 127.0.0.1 12101 > ...` ([issue #3](https://github.com/Rob--W/firefox-android-backup-restore/issues/3)).\
    On Windows you can use [ncat](https://nmap.org/ncat/) with `ncat -l 127.0.0.1 -p 12101 > ...`.

3. Open the Firefox app on your phone, and visit `about:blank` (any webpage works).
4. On your PC, open `about:debugging` in Firefox
5. Set up a debug connection to the Android app, as described in the [requirements section](#requirements)
6. On your PC, for the debugged Android app, scroll down to "Main Process", click on "Inspect" and open the "Console" tab.
7. Copy the contents of [`snippets_for_firefox_debugging.js`](snippets_for_firefox_debugging.js).
8. Paste the code you copied into the Console run it.
9. In the Console run the following to make sure the script is functional:
    ```js
    fab_sanity_check();
    ```
10. Then type and run:
    ```js
    fab_backup_create();
    ```

    This copies [relevant files](#relevant-files) to `firefox-android-backup.tar.gz`.
    Wait until "Backup successfully created and transferred!" appears on the Console.

    If the script fails, saying it cannot find `nc`, see [issue #2](https://github.com/Rob--W/firefox-android-backup-restore/issues/2).
11. Close the reverse connection you opened earlier for the transfer:
     ```sh
     adb reverse --remove tcp:12101
     ```
12. Optionally to remove the temporary backup file and the log file run:
    ```js
    fab_cleanup();
    ```

## Restore

The steps below will replace [relevant files](#relevant-files) with the backup.

1. To restore the profile from the backup, put the backup archive on the device:
    ```sh
    adb shell mkdir /sdcard/Android/data/org.mozilla.firefox/
    adb push firefox-android-backup.tar.gz /sdcard/Android/data/org.mozilla.firefox/
    ```
2. Open the Firefox app on your phone, and visit `about:blank` (any webpage works).
3. On your PC, open `about:debugging` in Firefox
4. Set up a debug connection to the Android app, as described in the [requirements section](#requirements)
5. On your PC, for the debugged Android app, scroll down to "Main Process", click on "Inspect" and open the "Console" tab.
6. Copy the contents of [`snippets_for_firefox_debugging.js`](snippets_for_firefox_debugging.js).
7. Paste the code you copied into the Console run it.
8. Then type `fab_backup_restore();` and run it. This has no visible output, unless
  an error occurs. Upon successful completion, the app is killed to prevent the
  old app instance from corrupting the restored data.\
  The logs can be viewed with:
    ```sh
    adb shell cat /sdcard/Android/data/org.mozilla.firefox/firefox-android-backup.log
    ```
9. When you are done, remove the archive and log file with:
    ```js
    fab_cleanup();
    ```
