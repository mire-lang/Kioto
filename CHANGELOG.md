# kioto changelog

> **Re-versioned (2026-07-03):** Old 3.x tags mapped to semver:
> - 3.11.12 → 1.0.0 (base)
> - 3.11.13 → 1.1.0 (SDL3 module)

## 1.1.6 — 2026-07-07

### Changed
- **`thread::spawn` / `thread::join`**: `core/async/mod.mire` now uses the new
  namespace syntax (`thread::spawn` / `thread::join`) instead of legacy flat
  builtins (`thread_spawn` / `thread_join`). This aligns with builtin
  modularization in Avenys 3.11.44+.

## 1.1.5 — 2026-07-06

### New features

- **fs::walk**: Recursive directory traversal. Returns all file paths under a
  directory, recursively including subdirectories. Skips `.` and `..` entries.

## 1.1.4 — 2026-07-06

### Naming cleanup

- **Removed duplicate functions**: `lists::append` (alias for `push`),
  `lists::delete` (alias for `remove`), `strings::strip` (alias for `trim`).
- **Removed broken extern**: `proc::err` no longer declares non-existent
  `pal_proc_err`. Function now returns empty string with `@[deprecated]`.

### 1.1.3 — 2026-07-06

### Core cleanup

- **Removed stubs**: `gpu::available()` (hardcoded false), `lists::map/filter/fold`
  (returning dummies), `proc::on()` (empty body).
- **Namespace separators**: Fixed `basic.` → `basic::` and `strings.` → `strings::`
  in `math/complex.mire` and `core/term/mod.mire`.

### Tests

- **Extended test suite**: Tests split into modular files (`maybe.mire`, `result.mire`,
  `iter.mire`, `json.mire`, `async.mire`, `fs.mire`, `env.mire`, `proc.mire`,
  `math.mire`, `math_complex.mire`, `cli.mire`) for easier navigation.

## 1.1.2 — 2026-07-05

### Fixed

- **strings::substr:** All calls now compute length into a variable before
  passing to `substr`, avoiding a Mire runtime bug where inline
  `strings::len()` as a function argument corrupts concatenated strings.
  Fix applied across `ext/maybe`, `ext/result`, `ext/json`, `ext/ws`,
  `core/async`, `core/cli`, `core/net/http` (12+ functions affected).
- **async::channel:** Replaced undefined `proc_run("date +%s")` with
  `time::unix_ms()` — the `proc_run` function never existed in kioto's scope.
- **async::value, async::error_msg:** Fixed inline `strings::len()` bug
  in the same pattern.

### Added

- **Test suite:** 35 tests across `tests/test_smoke.mire` and
  `tests/test_ext.mire`. Run with `mire test` in the kioto directory.

## 1.1.1 — 2026-07-04

### Fixed

- **net::event:** `accept_one` bounded to 50 retries instead of infinite loop,
  returns -1 when exhausted.
- **net::http:** Removed unnecessary variable aliases in `get`, `post`,
  `build_get_request`, `build_post_request` — leftover use-after-move workarounds
  no longer needed.
- **net::http:** Fixed `if/if` patterns to `if/else` in `http_send` and `http_close`.

## 1.1.0 — 2026-07-03

### New modules

#### sdl3 — SDL3 video + rendering (`SDL_CreateWindow` + Wayland fix)
```mire
load kioto::sdl3

if sdl3::init_video() {
    set win = sdl3::create_window("My App" 640 480)
    set rend = sdl3::create_renderer(win)
    sdl3::fill_screen(rend 100 150 200)   # present first buffer → window appears
    sdl3::delay(3000)
    sdl3::destroy_renderer(rend)
    sdl3::destroy_window(win)
    sdl3::quit()
}
```
Full SDL3 FFI: `SDL_Init`, `SDL_CreateWindow`, `SDL_CreateRenderer`,
`SDL_SetRenderDrawColor`, `SDL_RenderClear`, `SDL_RenderPresent`, `SDL_Delay`.
Wrapper functions handle the Wayland requirement of committing a buffer before the
window is mapped by the compositor (Hyprland, GNOME, etc.).

### Changes

- **sdl3:** New `ext/sdl3` module with correct `:bool` return types for SDL3 C `_Bool`
  functions (`SDL_Init`, `SDL_SetRenderDrawColor`, `SDL_RenderClear`, `SDL_RenderPresent`).
- **sdl3:** Fixed `create_window` parameter type to `:str` (not `:&str`).
- **sdl3:** Window test requires renderer + `fill_screen` for Wayland buffer commit.
- **sdl2:** Fixed `create_window` parameter type from `:&str` to `:str`.
- **Compiler:** Removed debug `eprintln!` lines from `typeck.rs`.

### Design notes

- **SDL3 `:bool` vs SDL2 `:i64`:** SDL3 C API returns `_Bool` for success/failure
  functions; SDL2 returns `int` (0 = success). Module return types must match exactly
  for correct x86_64 ABI (`al` register vs full `rax`).
- **Wayland needs a buffer:** `SDL_CreateWindow` creates the surface but Wayland
  compositors only map it after the first `wl_surface.commit()` with a valid buffer
  attached. Always call `SDL_CreateRenderer` + `SDL_RenderPresent` before waiting.
- **`SDL_Delay` pumps Wayland events** but does not attach a buffer.

## 1.0.0 — 2025-06-25

### New modules

#### json — Full JSON parser, navigator, and builder (`e1a7d5a`)
```mire
set data = json::get(response, "user.name")
set age  = json::get(response, "user.age")    # "30"
set ok   = json::is_valid("{...}")
set t    = json::type_of(data, "items")       # "object" | "array" | "string" | ...
set keys = json::keys(obj, "")                # newline-separated key names
set n    = json::len("[1,2,3]", "")           # 3
set out  = json::object(pairs_vec)            # {"k":"v",...}
set out  = json::array(items_vec)             # [a,b,c]
set safe = json::quoted("hello \"world\"")    # escaped + quoted
```
Supports dot-path navigation (`user.tags.0.name`), array indexing, nested objects,
all JSON types (string, number, bool, null, object, array), and string escape/unescape.

#### maybe — Option type (`071d7d3`)
```mire
set m = maybe::some("hello")
if maybe::is_some(m) { use dasu(maybe::unwrap(m)) }
set v = maybe::unwrap_or(maybe::none(), "fallback")
set m2 = maybe::map(m, "mapped")
set m3 = maybe::and_then(m, next_maybe)
```
Tagged-string encoding (`"some:value"` / `"none"`).

#### result — Result type (`071d7d3`)
```mire
set r = result::ok("success")
set e = result::err("fail")
if result::is_ok(r) { use dasu(result::unwrap(r)) }
set v2 = result::unwrap_or(e, "default")
set em = result::unwrap_err(e)
set r2 = result::map(r, "mapped")
set r3 = result::map_err(e, "better error")
```
Tagged-string encoding (`"ok:value"` / `"err:message"`).

#### iter — Iterator utilities (`071d7d3`)
```mire
set parts = strings::split("a\nb\nc", "\n")
set n  = iter::count(parts)       # 3
set f  = iter::first(parts)       # "a"
set l  = iter::last(parts)        # "c"
set el = iter::nth(parts, 1)      # "b"
if iter::contains(parts, "b") { ... }
set idx = iter::index_of(parts, "b")
set rng = iter::range(0, 5)       # "0\n1\n2\n3\n4"
```
Operates on `vec[str]` (the standard Mire vector type).

#### ws — WebSocket client + server (`392067b`)
```mire
# Client (ws:// and wss:// with auto TLS)
set fd = ws::connect("ws://echo.example.com/")
ws::send::text(fd, "hello")
set msg = ws::recv::all(fd)
ws::close(fd)

# Server
ws::server::accept(client_fd)
set msg = ws::server::recv(client_fd, 65536)
ws::server::send_text(client_fd, "echo: " + msg)
ws::server::close(client_fd)
```
Client frames automatically masked per RFC 6455. Server frames unmasked.
Server PAL per RFC 6455 SS5.2–5.3 (handshake, frame encode/decode, XOR masking,
base64 encoding).

#### net::http server — Request parser + response builder (`757abc2`)
```mire
set method = net::http::req_method(raw)       # "GET" / "POST"
set path   = net::http::req_path(raw)         # "/api/data"
set qs     = net::http::req_query(raw)        # "a=1&b=2"
set host   = net::http::req_header(raw, "Host")
set body   = net::http::req_body(raw)         # after \r\n\r\n
set ct     = net::http::server_mime("style.css")  # MIME type
set resp   = net::http::resp_200(body, "text/html")
set notf   = net::http::resp_404()
set rd     = net::http::resp_302("/new-location")
net::http::serve_file(fd, "index.html")
```

### Changes

- **ws:** Server-side PAL functions added (`pal_ws_server_accept/send_text/recv/close`)
  with corresponding LLVM IR declarations. Client PAL rewired to use existing
  `pal_net_*` / `pal_tls_*` functions. (`418f48f` in mire)
- **ws:** Fixed `owl.toml` to export `server` submodule. (`d5cde03`)
- **maybe/result/iter:** Replaced dict-based stubs with tagged-string implementations
  that avoid LLVM codegen edge cases.
- **json:** Moved from planned `core/json` to `ext/json` to match export path.
  All helpers use `&str` borrows to avoid Mire ownership conflicts.
- **http server:** Flattened into `net::http` module to avoid 3-level module
  nesting (`kioto::net::http::server`). Functions prefixed (`server_mime`,
  `req_method`, `resp_200`, etc.) to avoid conflicts with existing client API.

### Design notes

- **Module nesting > 2 levels** requires `owl.toml` in every intermediate directory.
  When possible, flatten to 2 levels or use prefixed function names.
- **`&str` everywhere:** Mire's borrow checker is conservative — passing owned `str`
  to helper functions that consume it prevents further use of the value in the caller.
- **`str mut` building:** Long inline string concatenation can trigger codegen issues.
  Using intermediate variables or delegating to builder functions is more reliable.
- **No binary data in Mire strings:** Mire `str` is UTF-8/C-string. WebSocket frame
  construction requires C PAL functions for binary headers and XOR masking.
- **`rt_vec_get_str` spacing:** `rt_vec_get_str(v(i+1))` is parsed as function call.
  Use `rt_vec_get_str(v, i+1)` or `rt_vec_get_str(v(i+1))` (no space before `(`).
