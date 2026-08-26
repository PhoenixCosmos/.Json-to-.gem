Works in both Windows and Linux.
# build_plugin.rb

Generates a valid OpenC3/COSMOS plugin (gem) from a QMK/VIA-style keyboard
JSON definition (e.g. exported from a VIA config like `gem_01.json`).

It scaffolds a complete plugin directory — target folder, `plugin.txt`,
command/telemetry stub definitions, gemspec, and Rakefile — ready to be
packaged into a `.gem` and installed into an OpenC3 instance.

## Usage

1. Place the script and your keyboard JSON file in the same directory.
2. Edit the hardcoded input filename in the script if needed:
   ```ruby
   json_data = JSON.parse(File.read('gem_01.json'))
   ```
3. Run it:
   ```bash
   ruby build_plugin.rb
   ```
4. This creates a folder named `openc3-cosmos-<keyboard_name>/` containing:
   - `targets/<TARGET_NAME>/plugin.txt` — target/interface definition
   - `targets/<TARGET_NAME>/cmd_tlm/cmd.txt` and `tlm.txt` — command/telemetry stubs
   - `targets/<TARGET_NAME>/lib/` — empty, for custom target code
   - `openc3-cosmos-<keyboard_name>.gemspec`
   - `Rakefile`
5. Build the gem and install it into OpenC3:
   ```bash
   cd openc3-cosmos-<keyboard_name>
   gem build openc3-cosmos-<keyboard_name>.gemspec
   openc3.sh cli load openc3-cosmos-<keyboard_name>-1.0.0.gem
   (or openc3.bat cli load openc3-cosmos-<keyboard_name>-1.0.0.gem)
   ```
   (adjust the load command for however your OpenC3 instance is invoked —
   `openc3cli`, a running container, etc.)

## Dependencies

- **Ruby** (3.0+) with the standard library (`json`, `fileutils`,
  `rubygems/package_task`) — no external gems required to *run* the script.
- **RubyGems** (`gem build`) to package the generated `.gemspec` into a
  `.gem` file.
- **An OpenC3/COSMOS instance** (tested against **openc3 7.2.0**) to
  install and run the resulting plugin. Behavior on other major versions
  is not verified — see "Version sensitivity" below.
- Input: a keyboard definition JSON with at minimum a top-level `"name"`
  field. `vendorId`/`productId` are read from real VIA exports but not
  currently used by the script.

## Known limitations / potential problems

These were found by trial-and-error against a live OpenC3 7.2.0 instance;
several are OpenC3-version-specific and could shift again on other
versions.

- **Target name collisions.** The target name is derived purely from
  `json_data['name']` (uppercased, spaces → underscores). Two different
  JSON files with the same `"name"` (e.g. two variants of "Akko
  Keyboard") will generate the *same* target name and the second install
  will fail with `RuntimeError: ..._openc3_targets:<NAME> already exists`.
  OpenC3 enforces globally unique target names per scope — this script
  does **not** disambiguate variants. If you need multiple keyboards with
  the same display name to coexist, fold `vendorId`/`productId` (or
  another unique field) into the target name before generating.
- **Re-installing/updating a target.** To replace an already-installed
  target, uninstall/remove the existing plugin first rather than loading
  a new gem that reuses the same target name.
- **Gem/plugin naming convention.** The gem name and plugin directory
  must be prefixed `openc3-cosmos-` (not the older `cosmos-` prefix from
  legacy Ball Aerospace COSMOS). Using the wrong prefix does not fail
  cleanly — it surfaces later as an opaque `NoMethodError: undefined
  method 'strip' for nil` during install.
- **Gemspec metadata.** OpenC3 7.x expects App Store–style metadata
  (`s.metadata` with `openc3_store_title`, `openc3_store_description`,
  `openc3_store_keywords`, etc.) along with standard fields (`license`,
  `homepage`, `email`, `required_ruby_version`). A bare-bones gemspec
  missing these can also surface as the same opaque `strip`-on-`nil`
  error rather than a clear validation message. The placeholder values
  in the generated gemspec (`you@example.com`, `https://example.com`,
  generic keywords) should be replaced with real values before
  publishing anywhere beyond a local instance.
- **`plugin.txt` keyword syntax is strict and version-specific:**
  - `TARGET` requires **two** parameters — `TARGET <FOLDER NAME> <TARGET NAME>`
    — not just one.
  - `INTERFACE` must reference the interface file **with its `.rb`
    extension** (`tcpip_client_interface.rb`), not the bare name.
  - `APPEND_PARAMETER` (commands) does **not** take a leading bit-offset —
    format is `APPEND_PARAMETER <NAME> <BIT SIZE> <TYPE> <MIN> <MAX> <DEFAULT> <DESCRIPTION>`.
  - `APPEND_ITEM` (telemetry) takes **no** min/max/default at all —
    format is `APPEND_ITEM <NAME> <BIT SIZE> <TYPE> <DESCRIPTION>`.
  - Getting any of these wrong produces a parse-time error naming the
    exact line and expected usage, so check `Stderr` output carefully.
- **Target folder casing.** The folder under `targets/` must be named in
  **uppercase**, matching the target name OpenC3 uppercases internally.
  A lowercase folder (e.g. `targets/akko_keyboard`) will fail deploy with
  `RuntimeError: No target files found at .../targets/AKKO_KEYBOARD/**/*`
  even though the files exist under the lowercase path.
- **Hardcoded interface connection.** The generated `INTERFACE` line
  always points at `127.0.0.1 9000 9000` via `tcpip_client_interface.rb`
  regardless of the actual keyboard's connection method (most keyboards
  are USB HID devices, not TCP). This stub is a structural placeholder —
  for a real integration you'll need a custom interface (Ruby or Python)
  that actually talks to the device, likely via HID, and reference that
  file (with extension) in `plugin.txt` instead.
- **Command/telemetry stubs are placeholders.** Only a single generic
  `REQUEST_ID` field is generated for both command and telemetry; the
  rich `menus`/`content` structure in the source JSON (brightness,
  effect, color, etc.) is not translated into real packet definitions.
- **Version sensitivity.** All of the above syntax/behavior was
  confirmed against `openc3-7.2.0`. Earlier or later OpenC3 versions may
  have different `plugin.txt` requirements, gemspec expectations, or
  error messages.
- **Hardcoded input filename.** The script always reads `gem_01.json`
  from the current directory; it does not accept a filename argument.
- **No overwrite protection.** Re-running the script for the same
  keyboard name will silently overwrite files in an existing
  `openc3-cosmos-<name>/` directory.
