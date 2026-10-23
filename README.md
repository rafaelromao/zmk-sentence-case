# ZMK Sentence Case

A standalone Zephyr/ZMK module providing `&sentence_space`, a replacement for
`&kp SPACE`.

Repository: https://github.com/rafaelromao/zmk-sentence-case

## Usage

Add this directory to `ZMK_EXTRA_MODULES`, then include:

```dts
#include <behaviors.dtsi>
#include <behaviors/sentence_case.dtsi>
```

Use `&sentence_space` wherever sentence-aware space is wanted, including in
macros or the tap binding of a hold-tap. Other `&kp SPACE` bindings remain plain
spaces. For example:

```dts
&sentence_space {
    prior-idle-ms = <2000>;
    release-after-ms = <1000>;
};
```

### Bind it to your space key

In your existing keymap, replace the space binding:

```diff
- &kp SPACE
+ &sentence_space
```

For example, this illustrative six-key layer can type `a. B` without manually
pressing Shift:

```dts
/ {
    keymap {
        compatible = "zmk,keymap";
        default_layer {
            bindings = <&kp A &kp B &kp DOT &kp QMARK &kp EXCL &sentence_space>;
        };
    };
};
```

Keep your keyboard's existing number and order of bindings; the six-key layer
above only illustrates where the behavior goes. Keep `&kp SPACE` on any keys
where you do not want automatic capitalization.

### What typing looks like

Here, `Space` means the key bound to `&sentence_space`. Examples assume normal
host capitalization settings, no held modifiers, and the default timeouts.

| Keys typed | Text produced | Why |
| --- | --- | --- |
| `a . Space b a` | `a. Ba` | Only the first letter after the sentence boundary is capitalized. |
| `a ? Space b` | `a? B` | Question marks also arm capitalization. |
| `a ! Space b` | `a! B` | Exclamation marks do too. |
| `a Space b` | `a b` | An ordinary word boundary does not arm it. |
| `a . b` | `a.b` | Punctuation alone is not enough; sentence-aware space is required. |
| `. Space 1 a` | `. 1a` | A number consumes the one-shot without becoming a symbol. |
| `. Space Space a` | `.  a` | The extra space consumes the one-shot; `a` stays lowercase. |
| `. Space , a` | `. ,a` | A non-letter consumes it without adding Shift. |
| `.`, wait 3 seconds, `Space a` | `. a` | The punctuation is older than `prior-idle-ms`. |
| `. Space`, wait 2 seconds, `a` | `. a` | The one-shot has exceeded `release-after-ms`. |

For `. Space b a`, the sequence is:

1. `.` becomes the most recently emitted key press.
2. `Space` emits a normal space and arms capitalization. No Shift is held yet.
3. `b` consumes the one-shot and is emitted with Left Shift, producing `B`.
4. `a` is emitted normally, producing `a`.

Opening a layer alone does not consume the one-shot: `. Space`, hold your number
layer, then press `1`, produces `. 1`. The digit consumes it, so a subsequent
letter is not automatically capitalized. Backspace or an arrow key also consumes
it; the behavior tracks emitted key presses, not the text or cursor in your editor.

Space presses and releases behave like `&kp SPACE`. If the latest key press was
`.`, `!`, or `?` within `prior-idle-ms`, the space press arms a lazy, one-shot
Left Shift. The next keycode press consumes it; only keyboard letters A–Z get
shifted. Numbers, symbols, navigation, cancel, consumer keys, and modifiers
consume it without an added shift. Releases and layer-only behaviors do not
consume it. Another space consumes it without rearming. It expires after
`release-after-ms`; there is no held modifier to release on timeout.

Punctuation matching uses strict modifier semantics:
plain dot or Left-Shift question/exclamation, without extra modifiers. Shift+dot
(`>`) and shortcuts are not sentence terminators. The shift is attached to the
letter's keycode event as `LS(A)`, not `A`.
Only the first character of a macro is affected.

### Sticky shift and accents

`zmk,behavior-sentence-morph` chooses its first binding normally and its second
while capitalization is pending. It remembers the chosen binding through
release, even if that press consumes the one-shot. Both child bindings include
their own parameters; the morph itself takes none.

Concurrent keys sharing an instance retain independent branches, matched by
position (and split source). `CONFIG_ZMK_BEHAVIOR_SENTENCE_MORPH_MAX_HELD`
sets the per-instance capacity (default 8). If capacity is exhausted, only that
new press is rejected; releasing it cannot release another position's binding.

For example, ignore a redundant sticky-shift tap:

```dts
shift_guard: shift_guard {
    compatible = "zmk,behavior-sentence-morph";
    #binding-cells = <0>;
    bindings = <&sk LSHFT>, <&none>;
};
```

Use `&shift_guard` directly or as the tap behavior of a hold-tap.
With this binding, `. Space shift_guard a` produces `. A`: the redundant shift
tap emits nothing while sentence capitalization is pending. Outside that state,
`shift_guard a` behaves like ordinary sticky Shift and produces `A`.

#### Shift pressed after `. Space`

Sentence capitalization does not invert or cancel a manually applied Shift.
The result depends on which behavior your Shift key uses:

| Keys typed after `. Space` | Text produced | What happens |
| --- | --- | --- |
| Tap `shift_guard`, type `a b` | `. Ab` | The guarded one-shot Shift tap is ignored; sentence case capitalizes `a` only. |
| Tap `shift_guard` twice, type `a b` | `. Ab` | Both redundant taps are ignored while sentence case is pending. |
| Hold ordinary Shift, type `a b`, release Shift | `. AB` | The Shift press consumes sentence case; held Shift capitalizes both letters normally. |
| Tap an unguarded one-shot Shift, type `a b` | `. Ab` | With the standard non-lazy `&sk LSHFT`, its Shift press consumes sentence case; the manual one-shot capitalizes `a`. |
| Press and release ordinary Shift, then type `a b` | `. ab` | The Shift press consumes sentence case, and no Shift remains held for either letter. |

Here, ordinary Shift is `&kp LSHFT`, an unguarded one-shot is `&sk LSHFT`, and
`shift_guard` is the sentence-morph binding shown above. Use `&shift_guard` if
you want a redundant one-shot Shift tap to leave sentence capitalization armed.
This guarding is opt-in: the module does not automatically intercept every
Shift binding in your keymap. Examples assume the letters are typed before the
relevant sentence-case or manual one-shot timeout.

For dead-key accent macros, select a shifted-accent binding with
another sentence morph. That binding must leave the dead key unshifted and
shift the vowel; a stock lazy sticky shift would otherwise modify the dead key.

## Tests

The tests compile the real C behaviors into native ZMK and check its HID events;
they do not mock or reimplement the state machine. In a configured ZMK west
workspace:

```sh
python3 /path/to/zmk-sentence-case/tests/run.py \
  --zmk-app /path/to/zmk/app \
  --build-dir /path/to/sentence-tests
```

Build and event logs are saved under the build directory. The test fixture is
self-contained and requires only ZMK and this module.

## Reference implementation

For an example of using these behaviors in a complete keyboard configuration,
see [rafaelromao/keyboards](https://github.com/rafaelromao/keyboards).
