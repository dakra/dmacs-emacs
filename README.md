# dmacs

This branch is GNU Emacs `master` with a few experimental commits on top.
The commits are macOS (Cocoa/NS) only and are rebased onto `master` from
time to time, so expect the history of this branch to be rewritten.

The upstream README is in the file `README`.

## Commits on top of master

In order, oldest first:

- **Keep the image transform across canvas-refresh on NS**:
  a canvas image keeps its `:scale` (and smoothing) after
  `canvas-refresh`.
- **Add a Metal GPU display backend for macOS**:
  a GPU renderer for Cocoa frames, imported from the
  [emacs-gpu](https://github.com/tanrax/emacs-gpu) fork by Andros
  Fenollosa (Metal driver only).  It adds cursor animations, scroll
  easing and buffer cross-fades, configured through `lisp/gpu.el`.
- **Metal: add a smear cursor animation**:
  the `smear` value of `gpu-cursor-animation`, which stretches the
  cursor from the old cell into the new one.
- **Metal: support translucent backgrounds**:
  the `alpha-background`, `ns-alpha-elements` and `ns-background-blur`
  frame parameters.
- **Metal: fix crashes, scrambled text and cursor glitches**:
  fixes for the three commits above.  Deleting a child frame and starting
  `emacs --daemon` no longer crash, text no longer turns into wrong
  glyphs when the glyph atlas fills up or a font is reopened, switching
  windows plays the cursor animation once, scrolling no longer leaves
  a stale cursor box behind, canvas images update after
  `canvas-refresh`, a black cursor keeps its color, and
  `ns-alpha-default` applies behind text.

## Building

```sh
./autogen.sh
./configure --with-ns
make
make install    # creates nextstep/Emacs.app
```

The Metal backend is part of every Cocoa build; there is no configure
option for it.

## Building with Homebrew emacs-plus

The [emacs-plus](https://github.com/d12frosted/homebrew-emacs-plus)
formula `emacs-plus@32` builds Emacs `master` and can apply this branch
as a patch.

Generate the patch from a checkout of this branch:

```sh
git diff $(git merge-base master dmacs) dmacs -- . ':(exclude)README.md' \
  > ~/.config/emacs-plus/dmacs-emacs-32.patch
shasum -a 256 ~/.config/emacs-plus/dmacs-emacs-32.patch
```

List it in `~/.config/emacs-plus/build.yml` with the hash that `shasum`
printed:

```yaml
patches:
  - dmacs:
      "32":
        url: ~/.config/emacs-plus/dmacs-emacs-32.patch
        sha256: <hash>
```

Then build:

```sh
brew install emacs-plus@32      # or: brew reinstall emacs-plus@32
```

Regenerate the patch and update the hash whenever the branch changes.
The formula always builds the current upstream `master`, so the patch
can stop applying when `master` moves ahead of this branch; rebase the
branch onto `master` and regenerate it then.

The formula runs `autogen.sh` before it applies patches, so a patch
cannot change `configure.ac`.  That is why the Metal backend is wired
into `src/Makefile.in` instead of having a configure option.

## Using the GPU backend

The backend is enabled at startup.  Set the environment variable
`EMACS_GPU_DISABLE=1` to start with the stock renderer; translucent
backgrounds are only available with the GPU backend.

`M-x gpu-status` shows the current state, and `M-x customize-group RET
gpu` lists the options.  For example:

```elisp
(with-eval-after-load 'gpu
  (setopt gpu-cursor-animation 'smear
          gpu-cursor-effects-while-typing t))

;; Translucent, blurred background with opaque text.
(add-to-list 'default-frame-alist '(alpha-background . 0.8))
(add-to-list 'default-frame-alist '(ns-background-blur . 20))
```

`ns-alpha-elements` is a list that selects which backgrounds become
translucent: `ns-alpha-all`, `ns-alpha-default`, `ns-alpha-glyphs` and
`ns-alpha-fringe`.  When it is unset, all of them are.

## Known limitations

- Changing `ns-alpha-elements` on an existing frame takes effect after
  `M-x redraw-display`.
- The title bar becomes translucent together with the frame.
- While the smear cursor is moving it covers the text it passes over.
