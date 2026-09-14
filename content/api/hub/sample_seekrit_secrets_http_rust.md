title = "HTTP component with encrypted secrets"
template = "render_hub_content_body"
date = "2026-09-14T00:00:00Z"
content-type = "text/html"
tags = ["rust", "http", "secrets", "component dependencies"]

[extra]
author = "seekrit"
type = "hub_document"
category = "Sample"
language = "Rust"
created_at = "2026-09-14T00:00:00Z"
last_updated = "2026-09-14T00:00:00Z"
spin_version = ">=v3.0"
summary = "A Rust HTTP component that reads an API key at request time from a dependency component, with no credential in the manifest, the source, or the built Wasm."
url = "https://seekrit.dev/docs/guides/spin"
keywords = "secrets, api keys, http, rust, component dependencies, wasi-config, variables, vault"

---

A worked example of using [component
dependencies](https://spinframework.dev/v3/writing-apps#using-component-dependencies)
to keep credentials out of a Spin component. The app has one HTTP handler that
needs an API key. Instead of reading it from a variable, it imports
`seekrit:secrets/store` from a dependency that fetches and decrypts on its
behalf.

## spin.toml

```toml
spin_manifest_version = 2

[application]
name = "seekrit-spin-example"
version = "0.1.0"

[variables]
seekrit_token = { required = true }

[[trigger.http]]
route = "/..."
component = "app"

[component.app]
source = "target/wasm32-wasip2/release/seekrit_spin_example.wasm"
allowed_outbound_hosts = ["https://api.seekrit.dev"]
dependencies_inherit_configuration = true

[component.app.variables]
seekrit_token = "\{{ seekrit_token }}"

[component.app.dependencies]
"seekrit:secrets/store" = { version = "0.3.0", package = "seekritdev:secrets-spin", registry = "ghcr.io" }

[component.app.build]
command = "cargo build --target wasm32-wasip2 --release"
watch = ["src/**/*.rs", "Cargo.toml"]
```

## wit/world.wit

```wit
package seekrit:spin-example;

world app {
  import seekrit:secrets/store@0.1.0;
}
```

## src/lib.rs

```rust
use spin_sdk::http::{IntoResponse, Request, Response};
use spin_sdk::http_service;

// Bindings for the interface this component imports. Spin plugs the dependency
// into it at load time, so nothing here names a package or a registry.
spin_sdk::wit_bindgen::generate!({
    path: "wit",
    world: "app",
    runtime_path: "::spin_sdk::wit_bindgen::rt",
    generate_all,
});

use seekrit::secrets::store;

#[http_service]
async fn handle_seekrit_example(_req: Request) -> anyhow::Result<impl IntoResponse> {
    // Names carry no values, so this is safe to return.
    let names = match store::list_names() {
        Ok(names) => names,
        Err(e) => return Ok(text(503, &format!("{e:?}"))),
    };

    // The value stays inside this component; report its length, not the value.
    let api_key = match store::get("STRIPE_API_KEY") {
        Ok(Some(v)) => format!("{} bytes", v.len()),
        Ok(None) => "not in scope".to_string(),
        Err(e) => return Ok(text(503, &format!("{e:?}"))),
    };

    Ok(text(200, &format!("in scope: {}\nSTRIPE_API_KEY: {api_key}\n", names.join(", "))))
}

fn text(status: u16, body: &str) -> impl IntoResponse {
    Response::builder()
        .status(status)
        .header("content-type", "text/plain")
        .body(body.to_string())
}
```

## Running it

```bash
export SPIN_VARIABLE_SEEKRIT_TOKEN="skt_..."
spin build
spin up
curl localhost:3000
```

```
in scope: DATABASE_URL, STRIPE_API_KEY
STRIPE_API_KEY: 32 bytes
```

For a deployment, point `seekrit_token` at a variables provider in
`runtime-config.toml` — Vault or Azure Key Vault — and nothing in the component
changes.

## What is worth copying from this

The pattern generalises past secrets. A dependency component is a way to give
one component a capability without giving it the credential the capability
needs, and three manifest details make or break it:

- `dependencies_inherit_configuration = true`, because Spin grants a dependency
  none of the parent's permissions by default. Without it the dependency can
  neither call out nor read a variable.
- The dependency package must import the exact `wasi:config` version Spin
  registers (`0.2.0-draft-2024-09-27`). wasmtime gives pre-release versions no
  semver-compatible matching, so a component built against `0.2.0-draft` is a
  different import name and fails at instantiation rather than at a call site.
- Secrets belong in `[component.*.variables]`, not `component.*.environment` —
  manifest expressions are not supported in `environment`, so it can only hold a
  literal.

One caveat if the dependency's exported interface shares a short name with one
of its own imports, as this one does (`seekrit:secrets/store` and
`wasi:config/store`): `spin_sdk::dependencies!()` selects by bare name and
generates the wrong one. Declaring the import in your own `wit/` and calling
`spin_sdk::wit_bindgen::generate!`, as above, works regardless.

Verified against Spin 4.1.0.

Full walkthrough:
[seekrit.dev/docs/guides/spin](https://seekrit.dev/docs/guides/spin).
