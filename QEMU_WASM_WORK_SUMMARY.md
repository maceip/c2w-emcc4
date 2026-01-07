# QEMU-Wasm and c2w-emcc4 Build & Deployment Summary

This document summarizes the changes and setup required to build and run the `container2wasm` (c2w) project with Emscripten (`--to-js`) support, using a local patched version of QEMU.

## Work Completed

### 1. Source Synchronization & Patches
- **Local QEMU:** The build uses a local QEMU source located in `~/c2w-emcc4/qemu-wasm-local/`.
- **Subprojects:** Applied manual patches to `qemu-wasm-local/subprojects/berkeley-softfloat-3` and `berkeley-testfloat-3` by copying `meson.build` from their respective `packagefiles/` directories.

### 2. Binary Modifications (`c2w` Go code)
- **File:** `cmd/c2w/main.go`
- **Change:** Modified the `build` function to automatically inject a `--build-context qemu-wasm-src=...` argument pointing to the local `qemu-wasm-local` directory. This ensures the Docker build uses the local patched QEMU instead of trying to pull it from a registry.

### 3. Dockerfile Fixes
- **Python Dependencies:** Added `python3-tomli` to the build stage as it's required for QEMU's `configure` script on newer Python versions.
- **VirtFS (host_data):** Fixed a startup crash where QEMU expected a `/tmp/wasi2` path for the `host_data` mount tag. Added logic to create this directory in the build stages.
- **Worker JS Fallback:** Added a `touch` command for `qemu-system-x86_64.worker.js` to ensure the `COPY` instruction doesn't fail if the file isn't generated (though it usually is).

### 4. Browser Environment (`index.html`)
- **Headers:** Configured the Apache server (`xterm-pty.conf`) to serve `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp`.
- **Resource Loading:** Updated `index.html` to load `xterm` and `xterm-pty` from a local `vendor/` directory to avoid COEP blocks on third-party CDNs.
- **Initialization:** 
    - **Fixed `xterm.css` Path:** Corrected the link to point to `./vendor/xterm.css` ensuring the terminal renders correctly.
    - **PTY Configuration:** Configured the PTY slave using `termios` flags (clearing `ECHO`, `ICANON`, etc.) to ensure compatibility with QEMU's serial console.
    - **Input Handling:** Removed `window.Module.stdin = () => null` which was blocking user input.
    - **Output Redirection:** Added `Module.print` and `Module.printErr` redirection to `xterm.writeln`. This ensures that boot logs and other stdout/stderr messages are visible in the terminal even if the PTY pipe is delayed.
    - **Filesystem:** Added `mod.FS.mkdirTree('/tmp/wasi2')` in `preRun` to satisfy QEMU's filesystem expectations.
    - **Debugging:** Exposed `window.term` and `window.master` to the global scope for easier debugging.

### 5. Networking Setup
- **Proxy:** Running `c2w-net --listen-ws 0.0.0.0:8889` (Plain WebSocket).
- **Port Alignment:** Updated `arg-module.js` and the `?net=delegate=...` URL parameter to use port `8889`.

---

## Instructions for the Next Agent

To continue development or verify the build:

### 1. Build the Binary
```bash
cd ~/c2w-emcc4
make
```

### 2. Run the Conversion
Convert a container image (e.g., Alpine) to Wasm/JS:
```bash
./out/c2w --dockerfile ./Dockerfile --to-js --target-arch=amd64 \
  --assets /home/pooppoop/git/agentbox \
  --build-arg LINUX_LOGLEVEL=7 \
  --build-arg INIT_DEBUG=true \
  --build-arg VM_CORE_NUMS=4 \
  alpine:3.20 /tmp/out-js/htdocs/
```

### 3. Fix Glue Code & Assets (Post-Build)
The Emscripten glue code expects the original filename. Ensure the entry point exists and correct index.html is used:
```bash
# Fix JS filename
cp /tmp/out-js/htdocs/out.js /tmp/out-js/htdocs/qemu-system-x86_64.js

# Ensure we use the patched index.html (if starting fresh, copy it from source/backup)
# cp /home/pooppoop/index.html /tmp/out-js/htdocs/index.html
```

### 4. Start the Networking Proxy
```bash
cd ~/c2w-emcc4
./out/c2w-net --listen-ws 0.0.0.0:8889 &
```

### 5. Serve and Access
Start the web server (ensure port 8083 is free, or adjust):
```bash
docker run --rm -d -p 8083:80 \
  -v "/tmp/out-js/htdocs:/usr/local/apache2/htdocs/:ro" \
  -v "/tmp/out-js/xterm-pty.conf:/usr/local/apache2/conf/extra/xterm-pty.conf:ro" \
  --entrypoint=/bin/sh httpd -c 'echo "Include conf/extra/xterm-pty.conf" >> /usr/local/apache2/conf/httpd.conf && httpd-foreground'
```

Access the terminal at:
`http://127.0.0.1:8083/?net=delegate=ws://127.0.0.1:8889`

Wait 10-20 seconds for the VM to boot. You should see boot logs ("mounting wasi0...", "Hello from console!") and then be able to interact with the shell.