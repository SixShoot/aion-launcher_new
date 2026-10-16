# Aion Launcher

Launcher for our Aion 4.8 server. One button instead of `start.bat`: it finds the game folder,
checks that the client is patched, and launches it with the correct server address.

This is **v0** — the very first working cut. It already does the following:

- finds the game folder (from `launcher.json`, from the remembered path, or next to itself);
- checks `bin32\version.dll` and `bin64\version.dll` to see if they exist and whether they are swapped;
- optionally downloads the official patch `version.dll` (build v1.5.0) and places it in
  `bin32` and `bin64`, verifying the archive checksum;
- launches `bin64\aion.bin -ip:<server-address> -port:2106 -loginex` (with fallback to `bin32`);
- opens the news page in a browser;
- displays the server status (“Online · 12 players”) if the status address is configured.

What it still **does not** do: update itself, pull settings from the website, or show a list of
news inside the window. Those are planned for the next slices (v1/v2).

---

## Installation (for players)

1. Download `AionLauncher.exe` and `launcher.json` from the
   [Releases](https://github.com/BenNiks1/aion-launcher/releases) page.
2. Place **both files in the root of the game folder** — the folder where `bin32`, `bin64`, `Data`,
   and `L10N` are located.
   Then the launcher will find the client automatically, with no manual configuration needed.
3. Run `AionLauncher.exe`.
4. If it says “Client is not patched,” click “Install patch” and wait for it to finish.
5. Click “PLAY.” The account is created automatically on first login: simply choose a login and
   password on the login screen.

The launcher can also be placed somewhere else — in that case, on first launch click “Browse…”
and select the game folder. The choice is remembered.

### Windows may show a warning — this is normal

The file is not signed with a certificate (it costs money, and we do not buy one at server startup).
On first launch, SmartScreen may say “Windows protected your PC”:
click **“More info” → “Run anyway”**.

To make sure the file you downloaded is ours, compare the checksum — it is published in the
release description and in the `SHA256SUMS.txt` file. In PowerShell:

```powershell
Get-FileHash .\AionLauncher.exe -Algorithm SHA256
```

Download the launcher only from the Releases page of this repository.

### Configuring `launcher.json`

The file sits next to the exe and is edited with a regular Notepad:

```json
{
  "gameDir": "",
  "serverHost": "127.0.0.1",
  "loginPort": 2106,
  "newsUrl": "",
  "statusUrl": "",
  "prefer32Bit": false,
  "patchZipUrl": "https://github.com/beyond-aion/aion-version-dll/releases/download/v1.5.0/version-dll.zip",
  "patchZipSha256": "3c0034fa381a248a73b3034f47fdac40dc8b8665079b1a067920b0c519014670"
}
```

| Key | Meaning |
| --- | --- |
| `gameDir` | Game folder. Empty means the launcher will find it automatically (remembered path, then the folder next to the exe) |
| `serverHost` | Address of our login server: IPv4 or domain name |
| `loginPort` | Login server port, default `2106` |
| `newsUrl` | News link; empty disables the button |
| `statusUrl` | Link to a `status.json` like `{"online": true, "players": 12}`; empty hides the status line |
| `prefer32Bit` | `true` means launch the 32-bit client (`bin32\aion.bin`) |
| `patchZipUrl` | Archive containing `version.dll` (only `https://`) |
| `patchZipSha256` | Archive checksum; installation is canceled if it does not match |

If the file is missing or damaged, the launcher uses default values — the “PLAY” button still works.

### If something goes wrong

| What you see | What to do |
| --- | --- |
| “Folder not found” | Click “Browse…” and select the folder that contains `bin32`, `bin64`, `Data`, and `L10N` |
| “Client is not patched” | Click “Install patch” or manually unpack [version-dll.zip](https://github.com/beyond-aion/aion-version-dll/releases) |
| “There is the same `version.dll` in both `bin32` and `bin64`” | These are different files. Extract the archive again; do not copy one file into both folders |
| “Server unavailable” | The server is down or you have no network; you can still try logging in to the game |
| Game does not start | Check that the game path has no Cyrillic characters and that your antivirus did not remove `version.dll` |

---

## For developers

Requires .NET SDK 8 or newer. The game client is not required for building.

```bash
dotnet build AionLauncher.sln                                  # build (WinForms also builds on Linux)
dotnet test tests/AionLauncher.Core.Tests                       # unit tests, cross-platform
dotnet publish src/AionLauncher/AionLauncher.csproj \
  -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -o publish
```

### Structure

| Project | What it is |
| --- | --- |
| `src/AionLauncher.Core` | All logic: config, game folder discovery, patch check, launch arguments, patch installation, etc. |
| `src/AionLauncher` | WinForms window, `net8.0-windows`. Only the UI |
| `tests/AionLauncher.Core.Tests` | xUnit for `AionLauncher.Core`. Runs on Linux and in CI |

The logic is intentionally separated from the UI: everything that can be tested is in `Core` and is
tested on any OS. Only buttons remain in `AionLauncher`.

### Rules that must not be relaxed

The server address and (in v1) the manifest are external input. Therefore:

- client launch arguments are a whitelist: only `-ip:<host>`, `-port:<1..65535>`, and `-loginex`
  (`LaunchArgs`);
- only `bin64\aion.bin` or `bin32\aion.bin` may be launched, nothing else (`LaunchArgs`);
- any path from the archive or manifest must resolve **inside** the game folder: no `..`,
  absolute paths, drive letters, or UNC paths (`SafePath`);
- from the patch archive, only `bin32/version.dll` and `bin64/version.dll` are extracted;
- the archive is downloaded only via `https://`, and its SHA256 is checked against the expected value;
- client files are written via a temporary file and `File.Move` — an interrupted installation will not
  leave a truncated `version.dll` (`AtomicFile`);
- browser URLs are restricted to `http://` and `https://`.

Each of these rules is covered by tests. If a rule changes, the test must change first.

### CI

`.github/workflows/build.yml` runs on `windows-latest`: restore → build → test → `dotnet publish`
(self-contained, single-file, win-x64) → artifact `AionLauncher-win-x64`. On tag `v*`, a GitHub
Release is also created with `AionLauncher.exe`, `launcher.json`, and `SHA256SUMS.txt`.

## License

MIT — see [LICENSE](LICENSE). The client patch `version.dll` is taken from
[beyond-aion/aion-version-dll](https://github.com/beyond-aion/aion-version-dll) and is distributed
under its own license; we do not modify it.
