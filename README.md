# build-static-tmux

A script for building a static tmux binary for Linux, which should run on a wide
selection of distributions (tested on Ubuntu, SLES12, TencentOS).

## Versions

| Component  | Version  |
|------------|----------|
| tmux       | 3.6b     |
| musl       | 1.2.6    |
| ncurses    | 6.6      |
| libevent   | 2.1.12   |
| UPX        | 5.2.0    |

## Usage

```sh
# Build (with resume support — completed components are skipped on re-run)
./build-static-tmux.sh

# Force a full rebuild (clear all stamps and artifacts)
./build-static-tmux.sh -f

# Compress the resulting binary with UPX
./build-static-tmux.sh -c

# Dump build logs to stdout on error
./build-static-tmux.sh -d
```

## Features

### Component-level resume (断点续跑)

Each build component (musl, libevent, ncurses, tmux, strip, upx, gzip) writes
a stamp file to `${TMUX_STATIC_HOME}/.stamps/` upon successful completion.
On re-run, completed components are skipped with `[SKIP]`.

- Use `-f` to force a full rebuild, which clears all stamps and artifacts.
- To redo a single component, delete its stamp:
  ```sh
  rm ${TMUX_STATIC_HOME}/.stamps/<key>.done
  ```
- All stamp keys include version numbers — bumping a version in the script
  automatically invalidates the old stamp and triggers a rebuild.

Stamps produced by a typical run:

```
musl-1.2.6.done
libevent-2.1.12.done
ncurses-6.6.done
tmux-3.6b.done
post-strip-3.6b.done       # always
post-upx-3.6b-5.2.0.done   # only when -c is used
post-gzip-3.6b.done        # always
```

### Environment variables

| Variable           | Default                       | Description                  |
|--------------------|-------------------------------|------------------------------|
| `TMUX_STATIC_HOME` | `/tmp/tmux-static`            | Build / install prefix       |
| `USE_UPX`          | `0`                           | Set to `1` to compress with UPX |
| `DUMP_LOG_ON_ERROR`| `0`                           | Set to `1` to dump logs on error |
| `FORCE_REBUILD`    | `0`                           | Set to `1` (or use `-f`) to force rebuild |

### Notes

- **ncurses** uses `install.libs` and `install.includes` only — the terminfo
  database (`install.data`) is skipped because it requires write access to
  `/usr/share/terminfo`. tmux reads the system terminfo database at runtime
  via `--with-terminfo-dirs` baked in at configure time.
- **ncurses** is built with wide-character support (default). tmux links
  against `libncursesw` / `libtinfow` to ensure correct Unicode/CJK rendering.
- **ncurses download** uses versioned archive URLs
  (`https://invisible-island.net/archives/ncurses/ncurses-X.Y.tar.gz`).
  Changing `NCURSES_VERSION` in the script downloads the correct version.
- **Output files** are placed in `${TMUX_STATIC_HOME}/bin/`:
  - `tmux.linux-<arch>.gz` — standard binary
  - `tmux.linux-<arch>.stripped.gz` — stripped binary
  - `tmux.linux-<arch>.upx.gz` — UPX-compressed (only with `-c`)

## Coexisting with an older tmux

If you already have a running tmux server (e.g. 3.5a), you can run the new
3.6b server on a different socket without disrupting existing sessions:

```sh
# Install the new binary with a different name
gzip -dc /tmp/tmux-static/bin/tmux.linux-*.stripped.gz > ~/.local/bin/tmux-3.6b
chmod +x ~/.local/bin/tmux-3.6b

# Old sessions: use the default socket (default tmux binary)
tmux attach

# New sessions: use a separate socket
alias tmuxn='tmux-3.6b -L new'
tmuxn new -s work
tmuxn attach -t work
```

Once all old sessions exit naturally, the old server terminates. You can then
swap the binary:

```sh
mv ~/.local/bin/tmux      ~/.local/bin/tmux-3.5a.bak
mv ~/.local/bin/tmux-3.6b ~/.local/bin/tmux
```

## License

This project bundles the build script from
[mjakob-gh/build-static-tmux](https://github.com/mjakob-gh/build-static-tmux).
Original license applies.