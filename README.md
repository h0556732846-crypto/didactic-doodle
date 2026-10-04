# Chord Follow App - project scaffold

An Android app that shows a song's chords and waits for you to play the correct chord on
your Korg PA 600 (connected via USB) before advancing to the next one. Includes a "simple"
mode that only requires the basic triad, not the full/exact chord voicing.

## How the pieces fit together

```
Chord ai app (existing, third-party - does the actual chord recognition)
      |
      | export to PDF, or view on screen and copy/type the text
      v
  PDF file  --------------------->  PdfChordImporter (PdfBox-Android text extraction)
  or pasted text  ----------------> ChordTextParser (regex chord-token extraction)
                                                    |
                                                    v
                                          SongProgressionEngine
                                           (waits for a chord match)
                                                    ^
                                                    |
                                            ChordMatcher (pitch-class matching,
                                            FULL or SIMPLE mode)
                                                    ^
                                                    |
                                          UsbMidiManager (raw USB Host reader,
                                          talks to the PA 600 directly)
```

## Why chord detection isn't built into this app

You already use **Chord ai** for chord recognition, and it's a closed third-party app with no
available source code - there's no way to add a feature inside it. Instead, this app imports
the chords Chord ai already found, via one of two paths:

1. **PDF import** (`PdfChordImporter`) - if Chord ai exports a chord sheet as PDF, this app
   extracts the text with PdfBox-Android (Android's built-in `PdfRenderer` can only rasterize
   pages to images, not extract text, so a real PDF text library is needed) and pulls out the
   chord symbols with `ChordTextParser`.
2. **Paste text** (also via `ChordTextParser`) - a fallback if PDF export isn't available or
   doesn't extract cleanly: copy the chord sheet as text and paste it into the app.

**Both extraction paths are approximate**: a regex over plain text can't perfectly distinguish
a chord symbol from a lyric word that happens to look like one, and PDF text extraction
doesn't preserve visual layout, so unusual chord-sheet formats may not extract in the right
order. **Review the imported chord list before relying on it for practice**.

There's also a standalone `backend_chord_detection/detect_chords.py` script, kept from an
earlier version of this plan (automatic chord detection directly from audio, via librosa) -
you likely don't need it now that Chord ai is doing that job.

## Why USB-MIDI is implemented from scratch instead of using android.media.midi

`android.media.midi` (Android's official MIDI API) only exists from **Android 6.0 (API 23)**
onward. Since the target device runs **Android 5.0.1 (API 21)**, `UsbMidiManager` talks
directly to the USB Host API and parses the raw 4-byte USB-MIDI event packets itself. This
works from API 12 onward, so it's compatible with the target device.

## Building the APK without a local Android Studio / network setup

If your local network blocks the Google/Maven domains Gradle needs (this project was built
alongside troubleshooting exactly that - a content-filtered connection blocking
`dl.google.com`, `maven.google.com`, etc.), this repo includes a GitHub Actions workflow
(`.github/workflows/build-apk.yml`) that builds the APK entirely on GitHub's own servers,
which have full internet access. You only need a browser:

1. Create a GitHub account (free) and a new repository.
2. Upload this entire project folder (drag-and-drop the folder onto GitHub's "Add file >
   Upload files" page works and preserves the folder structure).
3. Make sure the default branch is named `main` (GitHub names it this by default now) - the
   workflow triggers on pushes to `main`.
4. Go to the repository's **Actions** tab. The workflow should start automatically after the
   upload; if not, click "Build APK" in the left sidebar, then "Run workflow".
5. Once it finishes (green checkmark), click into the run, scroll to **Artifacts**, and
   download `chord-follow-debug-apk` - that's a zip containing the installable `.apk`.
6. Transfer the `.apk` to your phone (e.g. via a cloud drive or USB file transfer) and install
   it - you'll need to allow "install from unknown sources" since it's not from the Play Store.

This produces a **debug build**, which is fine for your own use and testing but isn't signed
for Play Store distribution (not relevant here since you're installing it directly).

## Before this runs on your actual PA 600

1. **Confirm the USB vendor/product ID.** `res/xml/device_filter.xml` currently only filters
   by Korg's vendor ID (0x0A4B / 2635). Connect the PA 600, log `device.productId` from
   `UsbMidiManager.findAndConnectDevice()`, and add it to the filter for a cleaner permission
   prompt experience.
2. **Check the PA 600's USB MIDI settings.** Some Korg keyboards have a menu setting for
   which MIDI channels/data go out over USB vs. the 5-pin DIN ports - make sure "local
   control" / USB MIDI output is enabled for the keyboard's own playing (not just
   arranger/style data).
3. **You'll need a USB OTG cable** (or the phone needs native USB-C host support) to connect
   the PA 600's USB port to the phone.

## What's scaffolded vs. what's left to build

Done (in this scaffold):
- USB-MIDI note on/off reading (`UsbMidiManager`)
- Pitch-class based chord matching, full vs. simple mode (`ChordMatcher`)
- "Wait for the right chord, then advance" state machine (`SongProgressionEngine`)
- Chord symbol parsing (`ChordFactory`) and text/PDF/JSON song loading
- Minimal UI with PDF import, paste-text import, and current/next chord display
- Cloud build via GitHub Actions, bypassing any local network/toolchain blockers

Not yet built (recommended next steps):
1. **Visual match feedback** - use `onHeldNotesChanged()` in `MainActivity` to color the
   chord text (e.g. green when matched) for a much more usable practice experience.
2. **Persisting songs** - currently everything is in-memory; add local file storage for a
   song library.
3. **Testing on real hardware** - this scaffold hasn't been run against a physical PA 600,
   so expect to debug the USB endpoint detection and packet parsing against the real device.

## Opening this project locally (if your network allows it)

This is a standard Gradle-based Android project. Open the `ChordFollowApp` folder in
Android Studio, let it sync, and run on a device (a physical device is required to test the
USB-MIDI parts - the emulator can't simulate a USB MIDI keyboard).
