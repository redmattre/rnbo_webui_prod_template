# RNBO + JUCE VST — Web UI Starter Pack

This file is a companion to `rnbo_juce_starter_pack_prompt_instructions.md`.
It covers the same project structure but uses a **web-based UI** (HTML / CSS / JS)
served through JUCE's `WebBrowserComponent` instead of a native C++ editor.
Use this file when the plugin UI should be built with frontend code.

---

## 0. Ground Rules
- You have to STRICTLY follow this rules and procedures
- Always check if you followed every rule and procedure
- If a problem takes too long ask the user if he made the right prompt (to save tokens)

## 1. Working Style

- Do the work, don't just explain it.
- Use very small step by step updates
- Keep updates concise
- The C++ side of the project (processor + editor) is nearly boilerplate.
  Touch it only to add / remove relay declarations and their attachments.
- All visual design lives in `src/webui/` (HTML / CSS / JS). No JUCE paint() needed.
- Do not use Projucer unless explicitly chosen.
- Do not use system or external JUCE installations — use only the one in `thirdparty/juce/` inside the workspace.
- Keep the RNBO export and the JUCE/plugin layer clearly separated.

---

## 2. Three-Layer Architecture

```
RNBO export            WebBrowserAudioEditor (C++)          src/webui/ (JS)
─────────────    →    ──────────────────────────────    →   ─────────────────
RangedAudio-          WebSliderRelay   "paramName"          getSliderState("paramName")
Parameter             WebSliderParameterAttachment           state.setNormalisedValue(x)
                                                            state.valueChangedEvent
```

- The **relay name string** in C++ and the **name passed to `getSliderState`** in JS
  must be **identical** and must match the RNBO parameter name exactly.
- `WebSliderParameterAttachment` handles bidirectional sync automatically
  (host automation, DAW undo, parameter feedback) — no manual suppression needed.

---

## 3. Versioning Discipline

- Every build, even tiny changes, MUST become a new iteration (PluginName_N).
- Never rebuild and redeploy under the same plugin version/name once a prior build has already been produced.
- Keep naming/version bumps in sync with build output and deployed bundle name.
- Make sure everytime you make a new build that for some reason it's fresh and not a copy of a previous version

---

## 4. CMake Configure & Build Rules

**Configure** (always use WEBVIEW mode):
```sh
cmake -S . -B build -G Ninja -DRNBO_EDITOR_MODE=WEBVIEW
```

**Build order — respect this sequence every time:**
```sh
# 1. Bundle the web UI into dist/ first
cd src/webui && npm run build && cd ../..

# 2. Build the plugin (dist/ gets compiled into the binary via juce_add_binary_data)
cmake --build build
```

Never pipe `cmake --build` to `head`, `grep`, or any short-circuiting command —
SIGPIPE will kill the linker and produce an empty/broken bundle.
Use `cmake --build build > /tmp/build.log 2>&1` to capture logs safely.

---

## 5. Deploy Pattern

Use this pattern, replacing the plugin name/version as needed and also replacing the username based on user's desktop path (ask the user if you don't know):

```sh
ditto build/RNBOAudioPlugin_artefacts/Debug/VST3/PluginName_N.vst3 \
      /Users/pierpaoloovarini/Desktop/VST3/PluginName_N.vst3

codesign --force --deep --sign - \
         /Users/pierpaoloovarini/Desktop/VST3/PluginName_N.vst3

codesign --verify --deep --strict --verbose=4 \
         /Users/pierpaoloovarini/Desktop/VST3/PluginName_N.vst3
```

Keep `NEEDS_MIDI_INPUT TRUE` and `NEEDS_MIDI_OUTPUT TRUE` in Plugin.cmake for
instrument plugins — DAWs may refuse to open them otherwise.

---

## 6. C++ Parameter Linkage Pattern

### 6a. WebBrowserAudioEditor.h — one relay per parameter

Declare relays **before** `_webComponent` (initialization order matters):

```cpp
// One WebSliderRelay per continuous RNBO parameter
WebSliderRelay _freqRelay { "Frequency" };
WebSliderRelay _volRelay  { "Volume" };
// Use WebToggleButtonRelay for boolean parameters
```

Pass each relay to the `WebBrowserComponent::Options` chain:
```cpp
SinglePageBrowser _webComponent {
    WebBrowserComponent::Options{}
        ...
        .withOptionsFrom (_freqRelay)
        .withOptionsFrom (_volRelay)
        ...
};
```

Declare one attachment per relay **after** `_webComponent`:
```cpp
WebSliderParameterAttachment _freqAttachment;
WebSliderParameterAttachment _volAttachment;
```

### 6b. WebBrowserAudioEditor.cpp — initialize attachments in constructor

```cpp
WebBrowserAudioEditor::WebBrowserAudioEditor (RNBO::JuceAudioProcessor* const p,
                                              RNBO::CoreObject& rnboObject)
    : AudioProcessorEditor (p)
    , _audioProcessor (p)
    , _rnboObject (rnboObject)
    , _freqAttachment (findParameter(p, "Frequency"), _freqRelay, nullptr)
    , _volAttachment  (findParameter(p, "Volume"),    _volRelay,  nullptr)
{
    addChildComponent (_webComponent);
    _webComponent.goToURL (kDevServerAddress);
    setSize (500, 400);  // adjust to match web UI size
}
```

**Rule:** add/remove exactly one relay declaration + one attachment per RNBO parameter.
Nothing else in the C++ editor needs to change for parameter additions.

---

## 7. JS Parameter Binding Pattern

```js
import { getSliderState, getToggleState } from 'juce-framework-frontend';

function bindSlider(rnboName, inputElement, displayElement) {
    const state = getSliderState(rnboName);  // name must match relay name in C++

    // Receive changes from the plugin / DAW automation
    state.valueChangedEvent.addListener(() => {
        inputElement.value = state.getNormalisedValue();        // 0..1
        displayElement.textContent = state.getScaledValue().toFixed(2);  // real units
    });

    // Required for DAW undo / automation recording
    inputElement.addEventListener('mousedown',  () => state.sliderDragStarted());
    inputElement.addEventListener('mouseup',    () => state.sliderDragEnded());
    inputElement.addEventListener('touchstart', () => state.sliderDragStarted(), { passive: true });
    inputElement.addEventListener('touchend',   () => state.sliderDragEnded());

    // Send changes to the plugin
    inputElement.addEventListener('input', () => {
        state.setNormalisedValue(parseFloat(inputElement.value));
        displayElement.textContent = state.getScaledValue().toFixed(2);
    });
}
```

Key rules:
- `getNormalisedValue()` / `setNormalisedValue()` always work in **0..1**.
- `getScaledValue()` returns the real unit value (Hz, dB, etc.) — use this for display.
- Always fire `sliderDragStarted` / `sliderDragEnded` or DAW automation recording breaks.
- For toggles: use `getToggleState(name)` → `getValue()` / `setValue(bool)`.

---

## 8. Dev Server Workflow

The plugin always **tries localhost:3000 first**. If nothing is listening there it
falls back to the compiled binary automatically (via `pageLoadHadNetworkError`).

**During development (hot reload, no recompile needed):**
```sh
cd src/webui
npm run dev          # Vite dev server on port 3000 with hot reload
```
Open the plugin in a DAW or standalone — changes to HTML/CSS/JS appear live.

**For release (bundle into binary):**
```sh
cd src/webui
npm run build        # outputs dist/index.html + dist/index.js
cd ../..
cmake --build build  # dist/ gets compiled in via juce_add_binary_data
```

`vite.config.js` must output a single `dist/index.html` + `dist/index.js`
(no hashed filenames). The existing config in the template already does this.

---

## 9. Audio / DSP Lessons

- DSP is the responsibility of the RNBO patch — don't add JUCE-side processing
  unless explicitly necessary.
- Parameter smoothing (to eliminate clicks): smooth the value **into** RNBO, not
  around it. Use `AudioProcessorParameter::Listener` + `std::atomic` to track the
  target on the message thread; advance a block-level `SmoothedValue` in
  `processBlock` and push the smoothed value to `_rnboObject.setParameterValue()`
  before calling the parent's `processBlock`. Suppress RNBO parameter feedback via
  `handleParameterEvent` override to prevent the smoother target from being
  overwritten by intermediate values.
- `WebSliderParameterAttachment` already handles bidirectional sync — the processor
  should not interfere with the parameter feedback loop for parameters that are not
  being smoothed.

---

## 10. Project Setup Checklist

Before starting:
- [ ] Verify Node.js is installed (`node --version`)
- [ ] Run `npm install` in `src/webui/` (once per project)
- [ ] Confirm RNBO export is in `export/` with correct parameter names and ranges
- [ ] Note all parameter names exactly as exported (case-sensitive — relay names must match)
- [ ] Set plugin identity in Plugin.cmake: `PRODUCT_NAME`, `PLUGIN_CODE`, `COMPANY_NAME`
- [ ] Set `IS_SYNTH TRUE` / `NEEDS_MIDI_INPUT TRUE` / `NEEDS_MIDI_OUTPUT TRUE` for instruments

When given a new RNBO export:
1. List all parameters (name, min, max, default, type: continuous / toggle)
2. Add one relay + attachment per parameter in `WebBrowserAudioEditor.h/.cpp`
3. Build the HTML structure in `src/webui/index.html`
4. Bind each parameter in `src/webui/main.js` using `getSliderState` / `getToggleState`
5. `npm run build` → `cmake --build build` → ditto → codesign → verify
