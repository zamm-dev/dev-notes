# Releasing a new version

We edit `src-tauri/Cargo.toml` to bump the version number:

```toml
[package]
name = "zamm"
version = "0.1.0"
...
```

We also edit `src-tauri\tauri.conf.json` in the same way:

```json
{
  ...,
  "package": {
    "productName": "zamm",
    "version": "0.1.0"
  },
  ...
}
```

`src-tauri/Cargo.lock` should update automatically on the next build to reflect the new version number. `webdriver/screenshots/baseline/desktop_wry/welcome-screen-800x600.png` will also have to be updated with the output of the CI test to reflect the new version number displayed onscreen.
