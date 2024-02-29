# Setting up Cargo for pre-commit

`--manifest-path` is only needed if your project is in a subdirectory.

```yaml
    hooks:
      - id: cargo-fmt
        name: cargo fmt
        entry: cargo fmt --manifest-path src-tauri/Cargo.toml --
        language: system
        types: [rust]
      - id: cargo-clippy
        name: cargo clippy
        entry: cargo clippy --manifest-path src-tauri/Cargo.toml --fix --allow-dirty --allow-staged --all-targets --all-features -- -Dwarnings
        language: system
        types: [rust]
        pass_filenames: false
```

If there are some warnings emitted by libraries that don't cause clippy to return a nonzero exit code (for an example, see [`diesel.md`](/general-notes/libraries/rust/diesel.md)), you can simulate that manually by creating `clippy.sh`:

```sh
#!/usr/bin/bash

clippy_output=$(cargo clippy --manifest-path src-tauri/Cargo.toml --fix --allow-dirty --allow-staged --all-targets --all-features -- -Dwarnings)

if [[ $clippy_output =~ "warning" ]]; then
  exit 0;
else
  exit 1;
fi
```

and then defining a pre-commit hook that calls that instead (make sure to `chmod +x` the script as needed)

```yaml
      - id: cargo-clippy
        name: cargo clippy
        entry: src-tauri/clippy.sh
        language: system
        types: [rust]
        pass_filenames: false
```

You can make the if statement shorter with

```bash
if [[ $clippy_output == *"warning"* ]]; then
  exit 1;
fi
```

To check for warnings specific to your project and not any forks (as was needed in [`chat.md`](/zamm-notes/chat.md)), do:

```bash
...
zamm_output=$(echo "$clippy_output" | awk '/Checking zamm /{flag=1; next} flag')

if [[ $zamm_output == *"warning"* ]]; then
  exit 1;
fi

```

To avoid cluttering the output every time you run `git commit`, you can do:

```bash
clippy_output=$(cargo clippy --manifest-path src-tauri/Cargo.toml --fix --allow-dirty --allow-staged --all-targets --all-features -- -Dwarnings 2>&1)
zamm_output=$(echo "$clippy_output" | awk '/Checking zamm /{flag=1; next} flag')
echo "$zamm_output"
```

To test that this script works, you can insert

```rust
let v = [1, 2, 3, 4];
v.first();
```

anywhere into your code. This should trigger the clippy warning

```
warning: unused return value of `core::slice::<impl [T]>::first` that must be used
  --> src\setup\db.rs:16:5
   |
16 |     v.first();
   |     ^^^^^^^^^
   |
   = note: `-D unused-must-use` implied by `-D warnings`
   = help: to override `-D warnings` add `#[allow(unused_must_use)]`  
help: use `let _ = ...` to ignore the resulting value
   |
16 |     let _ = v.first();
   |     +++++++

warning: `zamm` (bin "zamm" test) generated 1 warning
warning: `zamm` (bin "zamm") generated 1 warning (1 duplicate)        
    Finished dev [unoptimized + debuginfo] target(s) in 10.25s        
```

To check the last exit code in PowerShell to make sure that it's correctly detecting the warning, do

```powershell
$ echo $LastExitCode
1
```

[Note](https://stackoverflow.com/a/334893) that this is instead the `%ERRORLEVEL%` environment variable in cmd, and for Bash it is `$?`.

## Moving to Python script for Windows

Because Windows doesn't support Bash by default, we rename `clippy.sh` to `clippy.py` and replace the contents with

```python
import subprocess
import sys

clippy_command = "cargo clippy --color never --manifest-path src-tauri/Cargo.toml --fix --allow-dirty --allow-staged --all-targets --all-features -- -Dwarnings"
clippy_process = subprocess.Popen(
    clippy_command, shell=True, stderr=subprocess.STDOUT, stdout=subprocess.PIPE
)
clippy_output = clippy_process.communicate()[0].decode()

checking_zamm_output = clippy_output.split("Checking zamm")[1]
zamm_output = "\n".join(checking_zamm_output.split("\n")[1:])

print(zamm_output)

if "warning" in zamm_output:
    sys.exit(1)
```

Note that we add `--color never` to ensure that color codes don't pollute the output and prevent the "Checking zamm" string from being found.

We then edit `.pre-commit-config.yaml`:

```yaml
repos:
  ...
  - repo: local
    hooks:
      ...
      - id: cargo-clippy
        ...
        entry: python src-tauri/clippy.py
        ...
```

Unfortunately, this fails on CI with

```
Traceback (most recent call last):
  File "/home/runner/work/zamm/zamm/src-tauri/clippy.py", line 10, in <module>
    checking_zamm_output = clippy_output.split("Checking zamm")[1]
IndexError: list index out of range
```

Interestingly enough, when we debug the output on CI, we find that CI does not produce "Checking zamm v0.1.1" in its output:

```
...
warning: use of deprecated field `types::chat::ChatCompletionFunctions::parameters`
   --> /home/runner/work/zamm/zamm/forks/async-openai/async-openai/src/types/impls.rs:505:13
    |
505 |             parameters: value.1,
    |             ^^^^^^^^^^^^^^^^^^^

warning: `async-openai` (lib) generated 19 warnings (run `cargo clippy --fix --lib -p async-openai` to apply 1 suggestion)
warning: unused return value of `core::slice::<impl [T]>::first` that must be used
  --> src/setup/db.rs:16:5
   |
16 |     v.first();
   |     ^^^^^^^^^
   |
   = note: `-D unused-must-use` implied by `-D warnings`
   = help: to override `-D warnings` add `#[allow(unused_must_use)]`
help: use `let _ = ...` to ignore the resulting value
   |
16 |     let _ = v.first();
   |     +++++++

warning: `zamm` (bin "zamm") generated 1 warning
warning: `zamm` (bin "zamm" test) generated 1 warning (1 duplicate)
    Finished dev [unoptimized + debuginfo] target(s) in 10.89s
```

As such, we edit the Python script to look for the last warning we know to exist:

```python
SEPARATOR = "warning: `async-openai` (lib) generated "

...

separator_line = clippy_output.split(SEPARATOR)[1]
zamm_output = "\n".join(separator_line.split("\n")[1:])
```
