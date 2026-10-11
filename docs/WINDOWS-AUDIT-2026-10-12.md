# Filee Windows audit — 12 October 2026

Scope: focused source audit of upstream commit `7189d70c78fd782d903e28585dbeeed8c84d77ca`, followed by small safety fixes. Upstream main identifies itself as 1.10.0; the latest published release at inspection was v1.9.0. This is a fork build of main, not that upstream release binary.

## Changes made

| Finding | Trigger and result | Fix and evidence |
|---|---|---|
| P1: overwriting another batch input | A JPG and PNG are selected, a custom output name matches the PNG, and Overwrite replaces the original PNG. | Shared output allocation protects all batch inputs under every conflict policy. Regression verifies original bytes/text and numbered output. |
| P1: two outputs claim one name | Two inputs use the same custom output name with Overwrite. Both claim the path, causing lost output or a sharing violation. | Reserved output paths always get a numbered alternative. Regression checks distinct files and both contents while still replacing an unrelated existing output. |
| P1: archive extraction overwrites an input inside its destination | Extract `a.zip` into existing `a/` while `a/original.zip` is also an input. The archive contains `original.zip`. | Folder destinations containing batch inputs or reserved outputs get a numbered alternative. Real 7-Zip regression verifies original archive bytes remain intact. |
| P2: failed output reservations never expire | An engine allocates an output and fails before recording it in the job result. A retry unnecessarily uses a numbered name. | Reservations record their owning job and all that job's allocations are released in `finally`. Regression covers Rename, Overwrite and Skip. |
| P2: Windows device names and trailing characters | Custom patterns such as `CON`, `nul.notes`, `COM¹`, or a name ending in alternating dots/spaces can fail or address a device. | Names get a safe prefix and trailing dots/spaces are removed together. Archive readers reuse the same device-name classifier. Table tests include ordinary names and Unicode. |

The tests reproduced each original failure before its fix. The independent code review found the reservation and extraction gaps; both were reproduced and corrected before the follow-up review cleared them.

## Remaining enhancement opportunities

These are intentionally separate follow-ups; findings marked as risks have not been reproduced on the desktop.

| Priority | Area / source | Concrete next change | Why / verification needed |
|---|---|---|---|
| P1 | Output safety: `Magick/ImageEncoder.cs`, `Magick/MagickImageConverter.cs`, `Media/FfmpegConverter.cs` | Write conversions to a temporary file on the destination volume, then replace only after successful completion. Reuse the archive packer's staging pattern. | Several converters write directly to the final destination. Under Overwrite, failures can damage an earlier output. Inject write/encode failures and cancellation; confirm the previous output survives. |
| P2 | Drag UX: `App/Services/RadialController.cs`, `App/Views/RadialWindow.axaml.cs` | Tie the delayed close to the gesture that scheduled it. Add a rapid repeated-drag test plus a real Explorer drag/drop smoke check at different display scaling levels. | The 250 ms close callback examines current shared state. A previous drag's callback may close a new drag's window. This is a source-level race risk, not a reproduced desktop failure. Copy-only drag effects already protect originals. |
| P2 | Windows reliability: `Platform.Windows/WindowsPlatformServices.cs` | Validate active-tab selection on Windows 11 with Unicode paths, several Explorer tabs and network drives. | Selection currently takes the first matching HWND with selected items. The source documents this heuristic; automated headless tests cannot establish which Explorer tab is active. |
| P2 | Performance: `Core/Conversion/JobQueue.cs`, `Engines/Magick/MagickImageConverter.cs` | Measure a large-image batch; cap concurrent decoded-image memory if necessary. | Image parallelism defaults to processor count, while native image work can also use threads. No application-wide image memory budget was found in the inspected paths. Benchmark peak memory and responsiveness before choosing a limit. |
| P2 | IPC reliability: `App/Services/SingleInstance.cs` | Add a per-connection read deadline and bounded request size to the named-pipe argument receiver. | Pipes are restricted to the current user, but a client that connects and never finishes JSON can hold the only server connection indefinitely. Test an incomplete client followed by a valid launch. |
| P2 | Conversion testing / CI | Exercise FFmpeg media tests routinely; retain real archives, Unicode paths and cancellation cases. Add Windows desktop smoke checks separately from headless screenshots. | Missing optional tools cause explicit skips. Headless UI and pure gesture tests do not prove native Explorer gestures, shell extension registration or elevation behavior. |
| P3 | Build maintenance | Replace four obsolete Avalonia `TextBox.Watermark` attributes with `PlaceholderText`; resolve the nullable warning in `MediaTests.cs`. | Release build reported five existing warnings and no errors. They are outside the safety patch. |

## Security and licensing observations

- External process arguments use `ProcessStartInfo.ArgumentList` with shell execution disabled. Timeouts/cancellation terminate child process trees. General process output capture is unbounded; cap retained logs if long-running tools produce excessive output.
- Engine downloads have pinned versions and SHA-256 checks. The SDK used for this work came from Microsoft and passed its published SHA-512 check. Checksums establish artifact identity relative to the fetched manifest, not a malware certification.
- Archive code already rejects traversal, NTFS alternate-stream syntax in its own readers, encrypted archives and excessive entry counts; it checks declared disk requirements and removes extracted links. This audit does not establish immunity to hard-link, junction, hostile same-user races or malformed native-library inputs.
- The current NuGet vulnerability query reported no vulnerable direct or transitive packages in the configured feed. That is an advisory-feed result, not a complete security audit.
- Filee is MIT licensed. Bundled libraries and separate engine programs retain their own licenses. Preserve `LICENSE`, `THIRD-PARTY-NOTICES.md` and each engine's license. FFmpeg and Pandoc carry GPL terms; 7-Zip carries LGPL and the unRAR restriction. Public redistribution requires checking the corresponding source obligations; this task prepares a local personal build.

Windows reserved-name behavior follows [Microsoft's filename rules](https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file). Upstream source, release assets and licensing are available in the [original repository](https://github.com/KnifeLemon/Filee).

## Scope limits

The changes protect lexical paths in the current batch and active allocations in its shared queue. They do not detect hard-link aliases, resolve arbitrary junctions, or protect inputs of another queued batch before it starts. Renaming a colliding output preserves every current batch input; it does not make all converter writes transactional. Optional LibreOffice, calibre and Ghostscript integrations remain outside the locally bundled engine set.

## Validation

- Full Release solution tests: **922 passed, 19 skipped, 0 failed** (941 total). Core: 173/2; engines: 525/11; app: 188/2; CLI: 18/1; platform: 18/3 (passed/skipped).
- Full solution formatting verification and whitespace checks passed. Release build and Windows x64 self-contained app/CLI publishing succeeded. The existing nullable and four Avalonia deprecation warnings remain.
- Optional engines used locally: rhwp 0.8.7, Pandoc 3.12, FFmpeg 9.0.2, 7-Zip 26.03; downloads verified against the repository manifest. The 7-Zip MSI was unpacked into the workspace, not installed system-wide.
- The personal portable build `1.10.0-chee.1` opened a visible Windows desktop window and logged successful global input hook startup. A second launch forwarded a generated Unicode/spaced filename to its donut toolbar; clicking JPG converted it and added a successful history entry.
- Published CLI smoke checks converted PNG to WEBP, TXT to PDF, PDF to PNG, WAV to MP3 and ZIP to a folder. FFprobe decoded the generated WEBP and MP3; extracted content matched the source; the source image hash was unchanged.
- Skips include optional LibreOffice/calibre/Ghostscript dependencies, two unbuilt native Explorer-extension checks, and tests for other platforms/case-sensitive filesystems. Explorer Ctrl-drag, native shell-extension registration, UAC installation and startup-at-sign-in were not manually validated.
- This portable development build does not register Explorer menus or sign-in startup. Its updater points at the fork; no release exists there yet. It is unsigned and was not published as a GitHub binary release.
