# Terminal

## Recording terminal sessions

From a cursory online search, we find out about [termrec](http://angband.pl/termrec.html) for terminal recordings. Additionally, if we ever want to display these recordings online, there is a [JS tty-player](https://tty-player.chrismorgan.info/). As for the Rust ecosystem, there is [ttyrec-bin](https://crates.io/crates/ttyrec-bin), which features the Rust version of the `ttyrec` and `ttyplay` executables. However, we find out from the [main.rs](https://git.tozt.net/ttyrec-bin/tree/src/bin/ttyrec/main.rs?h=main#n211) implementation (at least as of commit `657318d`) that inputs are not recorded.

Next, we discover AsciiCinema, which also has a [Rust implementation](https://github.com/LegNeato/asciinema-rs). While most functionality in the Rust implementation are not part of a library, the part that is a part of [avt](https://docs.rs/avt/0.11.1/avt/) appears potentially useful for us in terms of rendering raw terminal output into neat strings. Meanwhile, the asciicast file format is [well documented](https://docs.asciinema.org/manual/asciicast/v2/), unlike the one for `ttyrec`.

### Database migration

As usual, we do

```bash
$ diesel migration generate add_asciicasts
Creating migrations/2024-06-27-125941_add_asciicasts/up.sql
Creating migrations/2024-06-27-125941_add_asciicasts/down.sql
```

By running the migrations with

```bash
$ diesel migration run
```

we get an automatically updated `src-tauri/src/schema.rs`:

```rs
...

diesel::table! {
    asciicasts (id) {
        id -> Text,
        timestamp -> Timestamp,
        command -> Text,
        shell -> Nullable<Text>,
        cast -> Text,
    }
}

...

diesel::allow_tables_to_appear_in_same_query!(
    ...,
    asciicasts,
    ...
);
```

Then we edit `src-tauri/migrations/2024-06-27-125941_add_asciicasts/up.sql`:

```sql
CREATE TABLE asciicasts (
  id VARCHAR PRIMARY KEY NOT NULL,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
  command VARCHAR NOT NULL,
  shell VARCHAR,
  cast TEXT NOT NULL
);
```

and `src-tauri/migrations/2024-06-27-125941_add_asciicasts/down.sql`:

```sql
DROP TABLE asciicasts;
```

We refactor the `Shell` struct (and associated tests) out of `src-tauri/src/commands/system.rs` into `src-tauri/src/models/shell.rs`:

```rust
use serde::{Deserialize, Serialize};
use specta::Type;

use std::env;

#[derive(Debug, Clone, Eq, PartialEq, Serialize, Deserialize, Type)]
pub enum Shell {
    Bash,
    Zsh,
    #[allow(clippy::enum_variant_names)]
    PowerShell,
}

pub fn get_shell() -> Option<Shell> {
    if let Ok(shell) = env::var("SHELL") {
        if shell.ends_with("/zsh") {
            return Some(Shell::Zsh);
        }
        if shell.ends_with("/bash") {
            return Some(Shell::Bash);
        }
    }

    if env::var("ZSH_NAME").is_ok() {
        return Some(Shell::Zsh);
    }
    if env::var("BASH").is_ok() {
        return Some(Shell::Bash);
    }

    #[cfg(target_os = "windows")]
    return Some(Shell::PowerShell);
    #[cfg(not(target_os = "windows"))]
    return None;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[ignore]
    #[test]
    fn test_can_determine_shell() {
        let shell = get_shell();
        println!(
            "Determined shell to be {:?} from env var {:?}",
            shell,
            env::var("SHELL")
        );
        assert!(shell.is_some());
    }
}

```

and we import that in `src-tauri/src/commands/system.rs`:

```rs
use crate::models::shell::{get_shell, Shell};
...
```

We also create `src-tauri/src/models/asciicasts.rs`:

```rust
use crate::models::llm_calls::EntityId;
use crate::schema::asciicasts;
use chrono::naive::NaiveDateTime;
use diesel::prelude::*;

#[derive(Queryable, Selectable, Debug, serde::Serialize, serde::Deserialize)]
#[diesel(table_name = asciicasts)]
pub struct AsciiCast {
    pub id: EntityId,
    pub timestamp: NaiveDateTime,
    pub command: String,
    pub shell: Option<String>,
    pub cast: String,
}

impl AsciiCast {
    #[allow(dead_code)]
    pub fn as_insertable(&self) -> NewAsciiCast {
        NewAsciiCast {
            id: &self.id,
            timestamp: &self.timestamp,
            command: &self.command,
            shell: self.shell.as_deref(),
            cast: &self.cast,
        }
    }
}

#[derive(Insertable)]
#[diesel(table_name = asciicasts)]
pub struct NewAsciiCast<'a> {
    pub id: &'a EntityId,
    pub timestamp: &'a NaiveDateTime,
    pub command: &'a str,
    pub shell: Option<&'a str>,
    pub cast: &'a str,
}

```

where we allow the dead code for now because we want to commit this before we start building functionality on top of the base data models.

We export these new modules in `src-tauri/src/models/mod.rs`:

```rs
...
pub mod asciicasts;
...
pub mod shell;
```

### Recording a new command

We see that, for the future, there are [ways](https://stackoverflow.com/a/49597789) to inject input into the process.

Meanwhile, to redirect stderr to stdout, [the `duct` crate](https://stackoverflow.com/a/41025699) appears to be the simplest solution. In case we need non-blocking solutions in the future, there's also [this alternative](https://stackoverflow.com/a/55565595).

We add `duct`, and `src-tauri/Cargo.toml` is updated automatically:

```toml
duct = "0.13.7"
```

We create a test script at `src-tauri/api/sample-terminal-sessions/interleaved.py` that will allow us to check that our output captures are working properly:

```python
#!/usr/bin/env python3

import sys

print("stdout")
sys.stdout.flush()
print("stderr", file=sys.stderr)
print("stdout")
```

We get some minimal code working in `src-tauri/src/commands/terminal.rs`. Note that:

1. The windows command for echo is [slightly different](https://superuser.com/a/1084354).
2. The windows output is also slightly different by including the carriage return `\r`
3. We need to [explicitly flush](https://unix.stackexchange.com/a/701896) the `stdout` buffer in order for the intended output to be captured properly

```rs
use super::errors::ZammResult;
use duct::cmd;

#[allow(dead_code)]
async fn capture_command_output(command: &str, args: &[&str]) -> ZammResult<String> {
    let output = cmd(command, args).stderr_to_stdout().read()?;
    Ok(output)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_capture_command_output() {
        #[cfg(target_os = "windows")]
        let output = capture_command_output("cmd", &["/C", "echo hello world"])
            .await
            .unwrap();
        #[cfg(not(target_os = "windows"))]
        let output = capture_command_output("echo", &["hello", "world"])
            .await
            .unwrap();
        assert_eq!(output, "hello world");
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        let output = capture_command_output(
            "python",
            &["api/sample-terminal-sessions/interleaved.py"],
        )
        .await
        .unwrap();

        #[cfg(target_os = "windows")]
        assert_eq!(output, "stdout\r\nstderr\r\nstdout");
        #[cfg(not(target_os = "windows"))]
        assert_eq!(output, "stdout\nstderr\nstdout");
    }
}

```

Finally, we expose this in `src-tauri/src/commands/mod.rs`:

```rs
...
mod terminal;

...
```

### Storing the recording in SQL

We add the `asciicast` crate.

```bash
$ cargo add asciicast
```

Then we edit `src-tauri/src/models/asciicasts.rs` to rename `AsciiCast` to `AsciiCastRow`, and have a new `AsciiCast` struct that can be serialized and deserialized from JSON:

```rs
...
use anyhow::anyhow;
use asciicast::{Entry, Header};
...
use diesel::backend::Backend;
use diesel::deserialize::{self, FromSql};
...
use diesel::serialize::{self, IsNull, Output, ToSql};
use diesel::sql_types::Text;
use diesel::sqlite::Sqlite;

#[derive(Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct AsciiCast {
    pub header: Header,
    pub entries: Vec<Entry>,
}

#[derive(...)]
#[diesel(table_name = asciicasts)]
pub struct AsciiCastRow {
    ...
}

impl AsciiCastRow {
    #[allow(dead_code)]
    pub fn as_insertable(&self) -> NewAsciiCast {
        ...
    }
}

impl ToSql<Text, Sqlite> for AsciiCast
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let header_str = serde_json::to_string(&self.header)?;
        let entries_str = self
            .entries
            .iter()
            .map(serde_json::to_string)
            .collect::<Result<Vec<String>, _>>()?;
        let json_str = format!("{}\n{}", header_str, entries_str.join("\n"));
        out.set_value(json_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for AsciiCast
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let json_str = String::from_sql(bytes)?;
        let mut lines = json_str.lines();
        let header_str = lines.next().ok_or(anyhow!("Empty cast"))?;
        let header: Header = serde_json::from_str(header_str)?;
        let entries = lines
            .map(serde_json::from_str)
            .collect::<Result<Vec<Entry>, _>>()?;
        Ok(AsciiCast { header, entries })
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::test_helpers::database::setup_database;
    use asciicast::{Entry, EventType, Header};

    use uuid::Uuid;

    #[test]
    fn test_ascii_cast_round_trip() {
        let mut conn = setup_database(None);

        let header = Header {
            version: 2,
            width: 80,
            height: 24,
            timestamp: None,
            duration: Some(1.0),
            idle_time_limit: None,
            command: Some("echo hello".to_string()),
            title: None,
            env: None,
        };
        let entries = vec![
            Entry {
                time: 0.0,
                event_type: EventType::Input,
                event_data: "echo hello".to_string(),
            },
            Entry {
                time: 1.0,
                event_type: EventType::Output,
                event_data: "hello".to_string(),
            },
        ];
        let cast = AsciiCast { header, entries };

        let row = AsciiCastRow {
            id: EntityId {
                uuid: Uuid::new_v4(),
            },
            timestamp: chrono::Utc::now().naive_utc(),
            command: "echo hello".to_string(),
            shell: None,
            cast: serde_json::to_string(&cast).unwrap(),
        };

        diesel::insert_into(asciicasts::table)
            .values(&row.as_insertable())
            .execute(&mut conn)
            .unwrap();

        let result = asciicasts::table.first::<AsciiCastRow>(&mut conn).unwrap();

        assert_eq!(result.id, row.id);
        assert_eq!(result.timestamp, row.timestamp);
        assert_eq!(result.command, row.command);
        assert_eq!(result.shell, row.shell);
        assert_eq!(result.cast, row.cast);
    }
}

```

### Storing OS instead of shell

Next, we realize that we actually want to keep track of the OS for now, rather than the shell that the command was executed under. We edit `src-tauri/migrations/2024-06-27-125941_add_asciicasts/up.sql`:

```sql
CREATE TABLE asciicasts (
  ...
  os VARCHAR,
  ...
);

```

`src-tauri/src/schema.rs` gets automatically updated when we rerun the migration:

```rs
diesel::table! {
    asciicasts (id) {
        ...
        os -> Nullable<Text>,
        ...
    }
}
```

We refactor the `OS` enum out of `src-tauri/src/commands/system.rs` and into `src-tauri/src/models/os.rs`, getting it to also derive `Copy` and `AsExpression`:


```rs
use diesel::backend::Backend;
use diesel::deserialize::{self, FromSql};
use diesel::expression::AsExpression;
use diesel::serialize::{self, IsNull, Output, ToSql};
use diesel::sql_types::Text;
use diesel::sqlite::Sqlite;
use serde::{Deserialize, Serialize};
use specta::Type;

#[derive(
    Debug, Clone, Copy, Eq, PartialEq, Serialize, Deserialize, AsExpression, Type,
)]
#[diesel(sql_type = Text)]
pub enum OS {
    Mac,
    Linux,
    Windows,
}

pub fn get_os() -> Option<OS> {
    #[cfg(target_os = "linux")]
    return Some(OS::Linux);
    #[cfg(target_os = "macos")]
    return Some(OS::Mac);
    #[cfg(target_os = "windows")]
    return Some(OS::Windows);
    #[cfg(not(any(
        target_os = "linux",
        target_os = "macos",
        target_os = "windows"
    )))]
    return None;
}

impl ToSql<Text, Sqlite> for OS
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let os_str = match self {
            OS::Mac => "Mac",
            OS::Linux => "Linux",
            OS::Windows => "Windows",
        };
        out.set_value(os_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for OS
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let os_str = String::from_sql(bytes)?;
        match os_str.as_str() {
            "Mac" => Ok(OS::Mac),
            "Linux" => Ok(OS::Linux),
            "Windows" => Ok(OS::Windows),
            _ => Err("Invalid OS string".into()),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_can_determine_os() {
        let os = get_os();
        println!("Determined OS to be {:?}", os);
        assert!(os.is_some());
    }
}

```

We expose this in `src-tauri/src/models/mod.rs`:

```rs
pub mod os;
```

and import this in the original `src-tauri/src/commands/system.rs`:

```rs
...
use crate::models::os::{get_os, OS};
...
```

Finally, we edit `src-tauri/src/models/asciicasts.rs`, renaming `AsciiCast` yet again to `AsciiCastData` and `AsciiCastRow` to `AsciiCast`, because that is ultimately how we want to present the data to the frontend. We also change the `shell` variable to hold `OS` data instead:

```rs
pub struct AsciiCast {
    ...
    pub os: Option<OS>,
    ...
}

impl AsciiCast {
    #[allow(dead_code)]
    pub fn as_insertable(&self) -> NewAsciiCast {
        NewAsciiCast {
            ...
            os: self.os,
            ...
        }
    }
}

#[derive(Insertable)]
#[diesel(table_name = asciicasts)]
pub struct NewAsciiCast<'a> {
    ...
    pub os: Option<OS>,
    ...
}

...

#[cfg(test)]
mod tests {
    ...

    #[test]
    fn test_ascii_cast_round_trip() {
        ...

        let row = AsciiCast {
            ...
            os: Some(OS::Mac),
            ...
        };

        ...
        assert_eq!(result.os, row.os);
        ...
    }
}
```

### Replaying recorded terminal sessions

We edit `src-tauri/src/models/asciicasts.rs` to refactor out the serialization/deserialization logic from the SQL functions, and to make saving and loading asciicasts to/from disk easier:

```rs
use crate::commands::errors::ZammResult;
...
use std::fmt;
use std::fmt::{Display, Formatter};

#[derive(..., Clone, ...)]
pub struct AsciiCastData {
    pub header: Header,
    pub entries: Vec<Entry>,
}

impl AsciiCastData {
    #[allow(dead_code)]
    pub fn new() -> Self {
        Self {
            header: Header {
                version: 2,
                width: 80,
                height: 24,
                timestamp: None,
                duration: None,
                idle_time_limit: None,
                command: None,
                title: None,
                env: None,
            },
            entries: Vec::new(),
        }
    }

    #[allow(dead_code)]
    pub fn load(file: &str) -> ZammResult<Self> {
        let contents = std::fs::read_to_string(file)?;
        AsciiCastData::parse(&contents)
    }

    #[allow(dead_code)]
    pub fn save(&self, file: &str) -> ZammResult<()> {
        let contents = format!("{}", self);
        std::fs::write(file, contents)?;
        Ok(())
    }

    pub fn parse(contents: &str) -> ZammResult<Self> {
        let mut lines = contents.lines();
        let header_str = lines.next().ok_or(anyhow!("Empty cast"))?;
        let header: Header = serde_json::from_str(header_str)?;
        let entries = lines
            .map(serde_json::from_str)
            .collect::<Result<Vec<Entry>, _>>()?;
        Ok(Self { header, entries })
    }
}

impl Display for AsciiCastData {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        let header = serde_json::to_string(&self.header).map_err(|_| fmt::Error)?;
        let entries = self
            .entries
            .iter()
            .map(|entry| serde_json::to_string(entry).map_err(|_| fmt::Error))
            .collect::<Result<Vec<String>, _>>()?;
        write!(f, "{}\n{}", header, entries.join("\n"))
    }
}

...

impl ToSql<Text, Sqlite> for AsciiCastData
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(...) -> serialize::Result {
        let json_str = format!("{}", self);
        ...
    }
}

impl<DB> FromSql<Text, DB> for AsciiCastData
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(...) -> deserialize::Result<Self> {
        let json_str = ...;
        AsciiCastData::parse(&json_str).map_err(Into::into)
    }
}

...
```

We then edit `src-tauri/src/commands/terminal.rs` to allow for capturing and replaying asciicasts using the logic above:

```rs
...
use crate::models::asciicasts::AsciiCastData;
...

pub fn command_args_to_string(command: &str, args: &[&str]) -> String {
    let escaped_args = args
        .iter()
        .map(|arg| {
            if arg.contains(' ') {
                format!("\"{}\"", arg)
            } else {
                arg.to_string()
            }
        })
        .collect::<Vec<String>>();
    format!("{} {}", command, escaped_args.join(" "))
}

pub trait Terminal {
    async fn run_command(&mut self, command: &str, args: &[&str])
        -> ZammResult<String>;
}

pub struct ActualTerminal {
    pub session_data: AsciiCastData,
}

impl ActualTerminal {
    #[allow(dead_code)]
    pub fn new() -> Self {
        Self {
            session_data: AsciiCastData::new(),
        }
    }
}

impl Terminal for ActualTerminal {
    async fn run_command(
        &mut self,
        command: &str,
        args: &[&str],
    ) -> ZammResult<String> {
        self.session_data.header.command = Some(command_args_to_string(command, args));

        let starting_time = chrono::Utc::now();
        self.session_data.header.timestamp = Some(starting_time);

        let output = cmd(command, args).stderr_to_stdout().read()?;
        let output_time = chrono::Utc::now();
        let duration = output_time - starting_time;
        self.session_data.entries.push(asciicast::Entry {
            time: duration.num_milliseconds() as f64 / 1000.0,
            event_type: asciicast::EventType::Output,
            event_data: output.clone(),
        });
        Ok(output)
    }
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_command_output() {
        let mut terminal = ActualTerminal::new();
        #[cfg(target_os = "windows")]
        let output = terminal
            .run_command("cmd", &[...])
            ...;
        #[cfg(not(target_os = "windows"))]
        let output = terminal
            .run_command("echo", &[...])
            ...;
        ...
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        let mut terminal = ActualTerminal::new();
        let output = terminal
            .run_command("python", &[...])
            ...;

        ...
    }
}
```

We expose `command_args_to_string` for tests in `src-tauri/src/commands/mod.rs` by making the submodule public:

```rs
pub mod terminal;
```

We then create a `src-tauri/src/test_helpers/terminal.rs` that either records or replays terminal sessions:

```rs
use crate::commands::errors::ZammResult;
use crate::commands::terminal::{command_args_to_string, ActualTerminal, Terminal};
use crate::models::asciicasts::AsciiCastData;
use asciicast::EventType;
use rvcr::VCRMode;

pub struct TestTerminal {
    mode: VCRMode,
    recording_file: String,
    cast: AsciiCastData,
}

impl TestTerminal {
    fn new(recording_file: &str) -> Self {
        let mode = match std::fs::metadata(recording_file) {
            Ok(_) => VCRMode::Replay,
            Err(_) => VCRMode::Record,
        };
        let cast = match mode {
            VCRMode::Record => AsciiCastData::new(),
            VCRMode::Replay => AsciiCastData::load(recording_file).unwrap(),
        };
        Self {
            mode,
            recording_file: recording_file.to_string(),
            cast,
        }
    }
}

impl Drop for TestTerminal {
    fn drop(&mut self) {
        if self.mode == VCRMode::Record {
            self.cast.save(&self.recording_file).unwrap();
        }
    }
}

impl Terminal for TestTerminal {
    async fn run_command(
        &mut self,
        command: &str,
        args: &[&str],
    ) -> ZammResult<String> {
        let actual_command = command_args_to_string(command, args);
        match self.mode {
            VCRMode::Record => {
                let mut actual_terminal = ActualTerminal::new();
                let actual_output = actual_terminal.run_command(command, args).await?;
                self.cast = actual_terminal.session_data.clone();
                Ok(actual_output)
            }
            VCRMode::Replay => {
                let expected_command = self.cast.header.command.as_ref().unwrap();
                assert_eq!(&actual_command, expected_command);

                assert_eq!(self.cast.entries.len(), 1);
                let entry = &self.cast.entries[0];
                assert_eq!(entry.event_type, EventType::Output);
                Ok(entry.event_data.clone())
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_terminal_replay() {
        let mut terminal = TestTerminal::new("api/sample-terminal-sessions/date.cast");
        let output = terminal
            .run_command("date", &["+%A %B %e, %Y %R %z"])
            .await
            .unwrap();
        assert_eq!(output, "Tuesday July  9, 2024 21:03 +0700");
    }
}

```

We expose this in `src-tauri/src/test_helpers/mod.rs`:

```rs
pub mod terminal;
```

Running the new `test_helpers::terminal::tests::test_terminal_replay` test, we end up with a `src-tauri/api/sample-terminal-sessions/date.cast` that looks like this:

```
{"version":2,"width":80,"height":24,"timestamp":1720533808,"command":"date \"+%A %B %e, %Y %R %z\""}
[0.004,"o","Tuesday July  9, 2024 21:03 +0700"]
```

Rerunning the test successfully produces the same output rather than a new time, so we have successfully replayed our recorded test.

We double check that we have correctly interpreted the spec by using the [Rust `asciinema`](https://github.com/LegNeato/asciinema-rs) release binary to check that our saved cast file can be correctly interpreted:

```bash
$ asciinema play src-tauri/api/sample-terminal-sessions/date.cast
Tuesday July  9, 2024 21:03 +0700
```

### Adding terminal to API tests

We edit `src-tauri/api/sample-call-schema.json` to allow for defining terminal interactions:

```json
{
  ...
  "$ref": "#/definitions/SampleCall",
  "definitions": {
    "SampleCall": {
      ...,
      "properties": {
        ...,
        "sideEffects": {
          ...
          "properties": {
            ...,
            "terminal": {
              "type": "object",
              "additionalProperties": false,
              "properties": {
                "recordingFile": {
                  "type": "string"
                }
              },
              "required": ["recordingFile"]
            },
            ...
          }
        }
      }
    }
  }
}
```

We run

```bash
$ make quicktype
```

and then we find that we have to remove the exception for the `src-svelte/src/lib/sample-call.ts$` pattern in `src-svelte/src/lib/sample-call.ts` so that the newly generated `src-svelte/src/lib/sample-call.ts` won't have a massive diff from the old one. It's unclear why we ever added that exception in the first place.

When we edit `src-tauri\src\test_helpers\api_testing.rs` to add the terminal side effect:

```rs
...
use crate::commands::terminal::Terminal;
...

#[derive(Default)]
pub struct SideEffectsHelpers {
    ...
    pub terminal: Option<Box<dyn Terminal>>,
}
```

we run into the compilation error

```
error[E0038]: the trait `commands::terminal::Terminal` cannot be made into an object
   --> src\test_helpers\api_testing.rs:219:30
    |
219 |     pub terminal: Option<Box<dyn Terminal>>,
    |                              ^^^^^^^^^^^^ `commands::terminal::Terminal` cannot be made into an object
    |
note: for a trait to be "object safe" it needs to allow building a vtable to allow the call to be resolvable dynamically; for more information visit <https://doc.rust-lang.org/reference/items/traits.html#object-safety>
   --> src\commands\terminal.rs:20:14
    |
19  | pub trait Terminal {
    |           -------- this trait cannot be made into an object...
20  |     async fn run_command(&mut self, command: &str, args: &... 
    |              ^^^^^^^^^^^ ...because method `run_command` is `async`
    = help: consider moving `run_command` to another trait
    = help: the following types implement the trait, consider defining an enum where each variant holds one of these types, implementing `commands::terminal::Terminal` for this new enum and using it instead:
              commands::terminal::ActualTerminal
              test_helpers::terminal::TestTerminal

For more information about this error, try `rustc --explain E0038`. 
```

It turns out this is a [known issue](https://users.rust-lang.org/t/resolving-not-object-safe-error-with-trait-having-async-methods/105175/3), and we make use of the [`async_trait`](https://docs.rs/async-trait/0.1.81/async_trait/index.html) crate:

```bash
$ cargo add async-trait
```

Then we edit `src-tauri/src/commands/terminal.rs` to make use of this:

```rs
...
use async_trait::async_trait;
...

#[async_trait]
pub trait Terminal {
    ...
}

...

#[async_trait]
impl Terminal for ActualTerminal {
    ...
}
```

and we edit `src-tauri/src/test_helpers/terminal.rs` to use this too, and to also make the `TestTerminal::new` function public:

```rs
...
use async_trait::async_trait;
...

impl TestTerminal {
    pub fn new(...) -> Self {
        ...
    }
}

#[async_trait]
impl Terminal for TestTerminal {
    ...
}
```

Sure enough, the compilation errors now go away, so we proceed with further editing `src-tauri/src/test_helpers/api_testing.rs`:

```rs
...
use crate::test_helpers::terminal::TestTerminal;
...

impl SampleCall {
    ...

    pub fn terminal_recording(&self) -> String {
        let recording_file = &self
            .side_effects
            .as_ref()
            .unwrap()
            .terminal
            .as_ref()
            .unwrap()
            .recording_file;

        format!("api/sample-terminal-sessions/{}", recording_file)
    }

    ...
}

...

pub trait SampleCallTestCase<T, U>
where
    ...
{
    ...

    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...
        if let Some(side_effects) = &sample.side_effects {
            ...

            // prepare terminal if necessary
            if side_effects.terminal.is_some() {
                let recording_path = PathBuf::from(sample.terminal_recording());
                let terminal =
                    Box::new(TestTerminal::new(recording_path.to_str().unwrap()));
                side_effects_helpers.terminal = Some(terminal);
            }

            ...
        }
        ...
    }
}
```

Next, we find that there's quite [a few](https://users.rust-lang.org/t/crate-for-splitting-a-string-like-bash/40062) options for parsing commandline strings, so we pick the most popular one and add

```bash
$ cargo add shlex
```

We can now edit `src-tauri/src/commands/terminal.rs` to remove `command_args_to_string` and to simplify the `run_command` arguments, since we'll only ever be getting raw strings from the LLM anyways:

```rs
...
use anyhow::anyhow;
...

#[async_trait]
pub trait Terminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String>;
}

...

#[async_trait]
impl Terminal for ActualTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        let cmd_and_args = shlex::split(command)
            .ok_or_else(|| anyhow!("Failed to split command '{}'", command))?;
        self.session_data.header.command = Some(command.to_string());

        ...

        let parsed_cmd = cmd_and_args
            .first()
            .ok_or_else(|| anyhow!("Failed to get command"))?;
        let parsed_args = &cmd_and_args[1..];
        let output = cmd(parsed_cmd, parsed_args).stderr_to_stdout().read()?;
        ...
    }
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_command_output() {
        ...
        #[cfg(target_os = "windows")]
        let output = terminal
            .run_command("cmd /C \"echo hello world\"")
            ...;
        #[cfg(not(target_os = "windows"))]
        let output = terminal.run_command("echo hello world")...;
        ...;
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        ...
        let output = terminal
            .run_command("python api/sample-terminal-sessions/interleaved.py")
            ...;
        ...
    }
}
```

We simplify our code in `src-tauri/src/test_helpers/terminal.rs` as well, removing the reference to `command_args_to_string`:

```rs
#[async_trait]
impl Terminal for TestTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        match self.mode {
            VCRMode::Record => {
                ...
                let actual_output = actual_terminal.run_command(command).await?;
                ...
            }
            VCRMode::Replay => {
                let expected_command = self.cast.header.command.as_ref().unwrap();
                assert_eq!(command, expected_command);
                ...
            }
        }
    }
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_terminal_replay() {
        ...
        let output = terminal
            .run_command("date \"+%A %B %e, %Y %R %z\"")
            ...;
        ...
    }
}
```

### Adding the command function

We first refactor the `crate::commands::terminal` module by creating `src-tauri/src/commands/terminal/mod.rs`:

```rs
pub mod models;
mod run;

#[allow(unused_imports)]
pub use models::{ActualTerminal, Terminal};
pub use run::run_command;

```

and move `src-tauri/src/commands/terminal.rs` to `src-tauri/src/commands/terminal/models.rs`. This does require an import change:

```rs
...
use crate::commands::errors::ZammResult;
...
```

Our first implementation in `src-tauri/src/commands/terminal/run.rs` looks like:

```rs
async fn run_command_helper(terminal: Box<dyn Terminal>, command: &str) -> ZammResult<String> {
    let output = terminal.run_command(command).await?;
    Ok(output)
}
```

but then we get the error

```
error[E0596]: cannot borrow `*terminal` as mutable, as `terminal` is not declared as mutable
 --> src\commands\terminal\run.rs:9:18
  |
9 |     let output = terminal.run_command(command).await?;
  |                  ^^^^^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
8 | async fn run_command_helper(mut terminal: Box<dyn Terminal>, command: &str) -> ZammResult<String> {
  |                             +++
```

We use the recommended fix, but then when we write test code, we run into a problem with converting `side_effects.terminal` into the right argument.

We realize that we need to make the side effects mutable because the test terminal needs to be mutable. As such, we edit `src-tauri/src/test_helpers/api_testing.rs`:

```rs
...

pub trait SampleCallTestCase<T, U>
where
    ...
{
    ...

    async fn make_request(
        ...,
        side_effects: &mut SideEffectsHelpers,
    ) -> U;

    ...

    async fn check_sample_call(...) -> SampleCallResult<T, U> {
        ...
        let result = self.make_request(&args, &mut side_effects_helpers).await;
        ...
    }

    ...
}

#[macro_export]
macro_rules! impl_direct_test_case {
    ...

            async fn make_request(
                ...,
                side_effects: &mut $crate::test_helpers::SideEffectsHelpers,
            ) -> $resp_type {
                ...
            }
    ...
}

#[macro_export]
macro_rules! impl_result_test_case {
    ...

            async fn make_request(
                ...
                side_effects: &mut $crate::test_helpers::SideEffectsHelpers,
            ) -> ZammResult<$resp_type> {
                ...
            }
    
    ...
}
```

We propagae these changes to the rest of the files, for example `src-tauri/src/commands/sounds.rs`:

```rs
...

#[cfg(test)]
mod tests {
    ...

    async fn make_request_helper(..., _: &mut SideEffectsHelpers) {
        ...
    }
    
    ...
}
```

Our final `src-tauri/src/commands/terminal/run.rs` looks like:

```rs
use specta::specta;

use crate::commands::errors::ZammResult;
use crate::commands::terminal::models::{ActualTerminal, Terminal};

async fn run_command_helper(
    terminal: &mut dyn Terminal,
    command: &str,
) -> ZammResult<String> {
    let output = terminal.run_command(command).await?;
    Ok(output)
}

#[allow(dead_code)]
#[tauri::command(async)]
#[specta]
pub async fn run_command(command: String) -> ZammResult<String> {
    let mut terminal = ActualTerminal::new();
    run_command_helper(&mut terminal, &command).await
}

#[cfg(test)]
mod tests {
    use super::*;

    use crate::test_helpers::SideEffectsHelpers;
    use crate::{check_sample, impl_result_test_case};

    #[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
    struct RunCommandRequest {
        command: String,
    }

    async fn make_request_helper(
        args: &RunCommandRequest,
        side_effects: &mut SideEffectsHelpers,
    ) -> ZammResult<String> {
        let terminal_mut = side_effects.terminal.as_mut().unwrap();
        run_command_helper(&mut **terminal_mut, &args.command).await
    }

    impl_result_test_case!(
        RunCommandTestCase,
        run_command,
        true,
        RunCommandRequest,
        String
    );

    check_sample!(
        RunCommandTestCase,
        test_date,
        "./api/sample-calls/run_command-date.yaml"
    );
}

```

where the test file at `src-tauri/api/sample-calls/run_command-date.yaml` looks like:

```yaml
request:
  - run_command
  - >
    {
      "command": "date \"+%A %B %e, %Y %R %z\""
    }
response:
  message: >
    "Tuesday July  9, 2024 21:03 +0700"
sideEffects:
  terminal:
    recordingFile: date.cast

```

Note that the response message is enclosed in quotes because the response should be a valid JSON string. This test passes.

We export this in `src-tauri/src/commands/mod.rs`:

```rs
#[allow(unused_imports)]
pub use terminal::run_command;
```

Unfortunately, adding the command to `src-tauri/src/main.rs`:

```rs
...
use commands::{
    ..., run_command, ...
};
...

fn main() {
    ...
    match ... {
        #[cfg(debug_assertions)]
        Some(Commands::ExportBindings {}) => {
            ts::export(
                collect_types![
                    ...,
                    run_command,
                ],
                ...
            )
            ...;
            ...
        }
        Some(Commands::Gui {}) | None => {
            ...
            tauri::Builder::default()
                ...
                .invoke_handler(tauri::generate_handler![
                    ...,
                    run_command,
                ])
                ...;
        }
    }
}
```

results in the error

```
error: future cannot be sent between threads safely                 
   --> src\commands\terminal\run.rs:15:1
    |
15  |   #[tauri::command(async)]
    |   ^^^^^^^^^^^^^^^^^^^^^^^^ future returned by `run_command` is not `Send`
    |
   ::: src\main.rs:85:33
    |
85  |                   .invoke_handler(tauri::generate_handler![   
    |  _________________________________-
86  | |                     get_api_keys,
87  | |                     set_api_key,
88  | |                     play_sound,
...   |
97  | |                     run_command,
98  | |                 ])
    | |_________________- in this macro invocation
    |
    = note: the full trait has been written to 'C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-tauri\target\debug\deps\zamm-3195bc6261177f2f.long-type-8021917870495259518.txt'
    = help: within `impl Future<Output = Result<..., ...>>`, the trait `std::marker::Send` is not implemented for `dyn commands::terminal::models::Terminal`
note: captured value is not `Send` because `&mut` references cannot be sent unless their referent is `Send`
   --> src\commands\terminal\run.rs:7:5
    |
7   |     terminal: &mut dyn Terminal,
    |     ^^^^^^^^ has type `&mut dyn commands::terminal::models::Terminal` which is not `Send`, because `dyn commands::terminal::models::Terminal` is not `Send`
note: required by a bound in `ResultFutureTag::future`
   --> C:\Users\Amos Ng\.cargo\registry\src\index.crates.io-6f17d22bba15001f\tauri-1.5.4\src\command.rs:293:42
    |
289 |     pub fn future<T, E, F>(self, value: F) -> impl Future<... 
    |            ------ required by a bound in this associated function
...
293 |       F: Future<Output = Result<T, E>> + Send,
    |                                          ^^^^ required by this bound in `ResultFutureTag::future`
    = note: this error originates in the macro `__cmd__run_command` which comes from the expansion of the macro `tauri::generate_handler` (in Nightly builds, run with -Z macro-backtrace for more info)    
```

It turns out all we have to do is edit `src-tauri/src/commands/terminal/models.rs` to declare that the trait `Terminal` must be implemented on `Send` and `Sync` structs:

```rs
#[async_trait]
pub trait Terminal: Send + Sync {
    ...
}
```

We now remove as many of the `#[allow(unused_imports)]` and `#[allow(dead_code)]` markers in our code as possible.

#### Storing the cast to the database

At first our tests are failing with

```
called `Result::unwrap()` on an `Err` value: DeserializationError(Serde { source: Json { source: Error("missing field `timestamp`", line: 1, column: 74) } })
```

We see in the **asciicast** repo that the file `src/header.rs` has the following line:

```rs
    #[test]
    fn test_deserializes_with_optional_data() {
        // TODO: Figure out why with chrono integration we need this extra
        // `timestamp`. Using `cargo test --no-default-features` works fine.
        let data = r#"{"version":2, "width":80, "height":40, "timestamp": null}"#;
        ...
    }
```

We fix that by putting in a non-null timestamp, since we're going to do that in the database anyways. We then run into the problem

```
assertion `left == right` failed
  left: AsciiCastData { header: Header { version: 2, width: 80, height: 24, timestamp: Some(2024-07-11T17:40:57Z), duration: Some(1.0), idle_time_limit: None, command: Some("echo hello"), title: None, env: None }, entries: [Entry { time: 0.0, event_type: Input, event_data: "echo hello" }, Entry { time: 1.0, event_type: Output, event_data: "hello" }] }
 right: AsciiCastData { header: Header { version: 2, width: 80, height: 24, timestamp: Some(2024-07-11T17:40:57.405142500Z), duration: Some(1.0), idle_time_limit: None, command: Some("echo hello"), title: None, env: None }, entries: [Entry { time: 0.0, event_type: Input, event_data: "echo hello" }, Entry { time: 1.0, event_type: Output, event_data: "hello" }] }
```

We see that there are [built-in methods](https://stackoverflow.com/a/56264863) to deal with this.

#### Changing the API to allow for interactive programs

We add a new test to `src-tauri/src/commands/terminal/models.rs`:

```rs
    #[cfg(not(target_os = "windows"))]
    #[tokio::test]
    async fn test_capture_output_without_blocking() {
        let mut terminal = ActualTerminal::new();
        let output = terminal
            .run_command("bash")
            .await
            .unwrap();

            assert_eq!(output, "bash-3.2$ ");
    }
```

As expected, this test hangs because the `bash` process does not exit.

Despite the existence of [an issue](https://github.com/oconnor663/duct.rs/issues/69) regarding live streaming command output, it appears that the `duct` crate does not really have great support for this despite the implementation of [`ReaderHandle`](https://docs.rs/duct/latest/duct/struct.ReaderHandle.html).

Some resources we find are:

- [This question](https://stackoverflow.com/q/34611742) that explains the difficulty of reading from OS pipes in a non-blocking manner
- [This answer](https://stackoverflow.com/a/73463169) appears to give an example for retrieving non-blocking reads from `stdin`. This is not quite our use case, as we want to do non-blocking reads from `stdout`. Nonetheless, the use of `tokio::select` is useful to note down.
- There is the `interactive_process` crate, but it appears not to support [general non-blocking reads](https://github.com/paulgb/interactive_process/issues/1) or [stderr reads](https://github.com/paulgb/interactive_process/issues/5)
- [This example code](https://andres.svbtle.com/convert-subprocess-stdout-stream-into-non-blocking-iterator-in-rust) that uses `std::process::Command` with `std::sync::mpsc`

We find out from `zamm/actions/use_terminal/terminal.py` of v0.0.5 of the original commandline version that we were using the `pexpect` package with its [`read_nonblocking`](https://pexpect.readthedocs.io/en/3.x/api/pexpect.html) function. As such, we try to use the Rust equivalent `rexpect` with its [`try_read`](https://docs.rs/rexpect/0.5.0/rexpect/reader/struct.NBReader.html#method.try_read) function. Incidentally, the implementation for that function also makes use of `std::sync::mpsc` channels and `try_recv`.

We replace `duct` with `rexpect` in `src-tauri/Cargo.toml`, and then edit `src-tauri/src/commands/terminal/models.rs` to replace `duct` there too:

```rs
...
use rexpect::spawn;
...

#[async_trait]
impl Terminal for ActualTerminal {
    async fn run_command(...) -> ZammResult<String> {
        ...
        let mut session = spawn(command, None)?;
        let output = session.exp_eof()?;
        ...
    }

    ...
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_command_output() {
        ...
        assert_eq!(output, "hello world\r\n");
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        ...
        assert_eq!(output, "stdout\r\nstderr\r\nstdout\r\n");
    }
}
```

For some reason, the results here now include the carriage return `\r`, even on Mac OS.

We then add a new error to `src-tauri/src/commands/errors.rs`:

```rs
#[derive(...)]
pub enum Error {
    ...,
    #[error(transparent)]
    Rexpect {
        #[from]
        source: rexpect::error::Error,
    },
    ...
}
```

Next, we try to see if we can use the non-blocking read instead:

```rs
pub struct ActualTerminal {
    pub session: Option<PtySession>,
    ...
}

impl ActualTerminal {
    pub fn new() -> Self {
        Self {
            session: None,
            ...
        }
    }
}

#[async_trait]
impl Terminal for ActualTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        ...
        let reader = &session.reader;
        let mut output = String::new();
        while let Some(chunk) = reader.try_read() {
            output.push_str(&chunk);
        }
        ...
        self.session = Some(session);
        Ok(output)
    }
    
    ...
}
```

We run into the compilation error

```
error[E0277]: `std::sync::mpsc::Receiver<Result<reader::PipedChar, reader::PipeError>>` cannot be shared between threads safely
   --> src/commands/terminal/models.rs:28:19
    |
28  | impl Terminal for ActualTerminal {
    |                   ^^^^^^^^^^^^^^ `std::sync::mpsc::Receiver<Result<reader::PipedChar, reader::PipeError>>` cannot be shared between threads safely
    |
    = help: within `commands::terminal::models::ActualTerminal`, the trait `Sync` is not implemented for `Receiver<Result<..., ...>>`
note: required because it appears within the type `NBReader
```

We eventually find that we can fix it with

```rs
...
use anyhow::anyhow;
...
use rexpect::session::PtySession;
...
use std::sync::{Arc, Mutex};
use std::thread::sleep;
use std::time::Duration;

...

pub struct ActualTerminal {
    pub session: Option<Arc<Mutex<PtySession>>>,
    ...
}

impl ActualTerminal {
    pub fn new() -> Self {
        Self {
            session: None,
            ...
        }
    }

    fn drain_read_buffer(&mut self) -> ZammResult<String> {
        let mut output = String::new();
        if let Some(session) = self.session.as_mut() {
            let reader = &mut session.lock()?.reader;
            while let Some(chunk) = reader.try_read() {
                output.push(chunk);
            }
        }
        Ok(output)
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        let mut output = String::new();
        loop {
            sleep(Duration::from_millis(100));
            let new_output = self.drain_read_buffer()?;
            if new_output.is_empty() {
                break;
            }
            output.push_str(&new_output);
        }

        let output_time = chrono::Utc::now();
        let duration = output_time
            - self
                .session_data
                .header
                .timestamp
                .ok_or(anyhow!("No timestamp"))?;
        self.session_data.entries.push(asciicast::Entry {
            time: duration.num_milliseconds() as f64 / 1000.0,
            event_type: asciicast::EventType::Output,
            event_data: output.clone(),
        });
        Ok(output)
    }
}

#[async_trait]
impl Terminal for ActualTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        if self.session.is_some() {
            return Err(anyhow!("Session already started").into());
        }

        ...

        self.session_data.header.timestamp = ...;

        let session = spawn(command, Some(1_000))?;
        self.session = Some(Arc::new(Mutex::new(session)));

        let result = self.read_updates()?;
        Ok(result)
    }

    ...
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_output_without_blocking() {
        let mut terminal = ActualTerminal::new();
        let output = terminal.run_command("bash").await.unwrap();

        assert!(output.ends_with("bash-3.2$ "), "Output: {}", output);
    }
}

```

The original example we set out to use now works! We realize that we can simplify this further by removing `drain_read_buffer` and instead using `rexpect`'s built-in timeout functionality to get the output thus far:

```rs
    fn read_updates(&mut self) -> ZammResult<String> {
        let session = self.session.as_mut().ok_or(anyhow!("No session"))?;
        let reader = &mut session.lock()?.reader;
        let output = match reader.read_until(&ReadUntil::EOF) {
            Ok((_, output)) => output,
            Err(e) => {
                match e {
                    rexpect::error::Error::Timeout { got, .. } => {
                        got
                    }
                    _ => return Err(e.into()),
                }
            }
        };

        ...
    }
```

We also change the session to start with only 100ms to get tests to run faster:

```rs
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        ...

        let session = spawn(command, Some(100))?;
        ...
    }
```

We then edit this further to try to get interactivity going:

```rs
...
use chrono::DateTime;
...

impl ActualTerminal {
    ...

    fn start_time(&self) -> ZammResult<DateTime<chrono::Utc>> {
        let result = self
            .session_data
            .header
            .timestamp
            .ok_or(anyhow!("No timestamp"))?;
        Ok(result)
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        let output = {
            let session_mutex = self.session.as_mut().ok_or(anyhow!("No session"))?;
            let mut session = session_mutex.lock()?;
            match session.exp_eof() {
                Ok(output) => output,
                Err(e) => {
                    match e {
                        rexpect::error::Error::Timeout { got, .. } => got
                            .replace("`\\n`\n", "\n")
                            .replace("`\\r`", "\r")
                            .replace("`^`", "\u{1b}"),
                        _ => return Err(e.into()),
                    }
                }
            }
        };

        let output_time = ...;
        let relative_time = output_time - self.start_time()?;
        self.session_data.entries.push(asciicast::Entry {
            time: relative_time.num_milliseconds() as f64 / 1000.0,
            ...
        });
        Ok(output)
    }

    #[cfg(test)]
    async fn send_input(&mut self, input: &str) -> ZammResult<String> {
        let session_mutex = self.session.as_mut().ok_or(anyhow!("No session"))?;
        {
            let mut session = session_mutex.lock()?;
            session.send(input)?;
            session.flush()?;
        }

        let relative_time = chrono::Utc::now() - self.start_time()?;
        self.session_data.entries.push(asciicast::Entry {
            time: relative_time.num_milliseconds() as f64 / 1000.0,
            event_type: asciicast::EventType::Input,
            event_data: input.to_string(),
        });

        self.read_updates()
    }
}

...

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_interaction() {
        let mut terminal = ActualTerminal::new();
        terminal.run_command("bash").await.unwrap();

        let output = terminal
            .send_input("python api/sample-terminal-sessions/interleaved.py\n")
            .await
            .unwrap();
        assert_eq!(output, "stdout\r\nstderr\r\nstdout\r\n");
    }
}
```

Note that we:

- refactor out `start_time` for reusability
- simplify the read further with `session.exp_eof` instead of using `session.reader`
- do all the `got.replace(...)` to restore the original output
- hide `send_input` behind a `#[cfg(test)]` for now because it's unused in regular code

Unfortunately, this fails with

```
---- commands::terminal::models::tests::test_capture_interaction stdout ----
thread 'commands::terminal::models::tests::test_capture_interaction' panicked at src/commands/terminal/models.rs:157:9:
assertion `left == right` failed
  left: "\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "
 right: "stdout\r\nstderr\r\nstdout\r\n"
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

It turns out that the buffer does not advance when there's a timeout. As such, we revert to the original implementation:

```rs
impl ActualTerminal {
    ...

    fn drain_read_buffer(&mut self) -> ZammResult<String> {
        ...
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        let mut output = String::new();
        loop {
            ...
        }

        ...
    }
```

and now the test passes when we change the assert to

```rs
assert_eq!(output, "stdout\r\nstderr\r\nstdout\r\nbash-3.2$ ");
```

Because we sleep ourselves, we can even change the timeout to

```rs
let session = spawn(command, Some(0))?;
```

We realize that we need to only add in new Asciicast entries when the output is non-empty. As such, we add logic and a new test for that:

```rs
impl ActualTerminal {
    ...

    pub fn read_updates(&mut self) -> ZammResult<String> {
        ...

        if !output.is_empty() {
            let output_time = ...;
            ...
            self.session_data.entries.push(asciicast::Entry {
                ...
            });
        }
        Ok(output)
    }

    ...
}

...

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_command_output() {
        ...
        assert_eq!(terminal.get_cast().entries.len(), 1);
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        ...
        assert_eq!(terminal.get_cast().entries.len(), 1);
    }

    #[tokio::test]
    async fn test_capture_output_without_blocking() {
        ...
        assert_eq!(terminal.get_cast().entries.len(), 1);
    }

    #[tokio::test]
    async fn test_no_entry_on_empty_capture() {
        let mut terminal = ActualTerminal::new();
        terminal.run_command("bash").await.unwrap();
        terminal.read_updates().unwrap();
        assert_eq!(terminal.get_cast().entries.len(), 1);
    }

    #[tokio::test]
    async fn test_capture_interaction() {
        ...
        assert_eq!(terminal.get_cast().entries.len(), 3);
    }
}

```

Unfortunately, when running all Cargo tests, we now find that this test fails:

```
---- commands::terminal::models::tests::test_capture_interleaved_output stdout ----
thread 'commands::terminal::models::tests::test_capture_interleaved_output' panicked at src/commands/terminal/models.rs:144:9:
assertion `left == right` failed
  left: ""
 right: "stdout\r\nstderr\r\nstdout\r\n"
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

It appears to only fail when running all tests -- not when running it as part of the terminal-specific tests. Sometimes this other test also fails non-deterministically:

```
---- commands::terminal::models::tests::test_capture_interaction stdout ----
thread 'commands::terminal::models::tests::test_capture_interaction' panicked at src/commands/terminal/models.rs:187:9:
assertion `left == right` failed
  left: ""
 right: "stdout\r\nstderr\r\nstdout\r\nbash-3.2$ "
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

We edit the function again, this time combining both approaches:

```rs
    pub fn read_updates(&mut self) -> ZammResult<String> {
        let output = {
            let session_mutex = self.session.as_mut().ok_or(anyhow!("No session"))?;
            let output_until_eof = {
                let mut session = session_mutex.lock()?;
                session.exp_eof().ok()
            };
            match output_until_eof {
                Some(full_output) => full_output,
                None => {
                    let mut interim_output = String::new();
                    loop {
                        sleep(Duration::from_millis(100));
                        let new_output = self.drain_read_buffer()?;
                        if new_output.is_empty() {
                            break;
                        }
                        interim_output.push_str(&new_output);
                    }
                    interim_output
                },
            }
        };

        ...
    }
```

Now the tests pass the vast majority of the time. We also raise the default timeout to 100ms.

##### Replaying interactivity

We edit `src-tauri/src/commands/terminal/models.rs` to move `read_updates` from the implementation to the trait:

```rs
pub trait Terminal: Send + Sync {
    ...
    fn read_updates(&mut self) -> ZammResult<String>;
    ...
}

#[async_trait]
impl Terminal for ActualTerminal {
    ...

    fn read_updates(&mut self) -> ZammResult<String> {
        ...
    }

    ...
}
```

We add `either`:

```bash
$ cargo add either
```

and refactor `src-tauri/src/test_helpers/terminal.rs` to use this instead of keeping track of the mode:

```rs
...
use either::Either::{self, Left, Right};
use std::thread::sleep;
use std::time::Duration;

pub struct TestTerminal {
    recording_file: String,
    terminal: Either<AsciiCastData, ActualTerminal>,
    entry_index: usize,
}

impl TestTerminal {
    pub fn new(recording_file: &str) -> Self {
        let terminal = match std::fs::metadata(recording_file) {
            Ok(_) => Either::Left(AsciiCastData::load(recording_file).unwrap()),
            Err(_) => Either::Right(ActualTerminal::new()),
        };
        Self {
            recording_file: recording_file.to_string(),
            terminal,
            entry_index: 0,
        }
    }

    fn next_entry(&mut self) -> &asciicast::Entry {
        match &self.terminal {
            Left(cast) => {
                let entry = &cast.entries[self.entry_index];
                self.entry_index += 1;
                entry
            }
            Right(_) => panic!("Expected recording"),
        }
    }
}

impl Drop for TestTerminal {
    fn drop(&mut self) {
        if let Right(terminal) = &self.terminal {
            terminal.get_cast().save(&self.recording_file).unwrap();
        }
    }
}

#[async_trait]
impl Terminal for TestTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        match &mut self.terminal {
            Left(cast) => {
                let expected_command = cast.header.command.as_ref().unwrap();
                assert_eq!(command, expected_command);

                let entry = self.next_entry();
                assert_eq!(entry.event_type, EventType::Output);
                Ok(entry.event_data.clone())
            }
            Right(actual_terminal) => actual_terminal.run_command(command).await,
        }
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        match &mut self.terminal {
            Left(_) => {
                let entry = self.next_entry();
                assert_eq!(entry.event_type, EventType::Output);
                Ok(entry.event_data.clone())
            }
            Right(actual_terminal) => actual_terminal.read_updates(),
        }
    }

    fn get_cast(&self) -> &AsciiCastData {
        match &self.terminal {
            Left(cast) => cast,
            Right(actual_terminal) => actual_terminal.get_cast(),
        }
    }
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_terminal_replay() {
        ...
        assert_eq!(output, "Friday September 20, 2024 18:23 +0700\r\n");
    }

    #[tokio::test]
    async fn test_terminal_pause() {
        let mut terminal = TestTerminal::new("api/sample-terminal-sessions/pause.cast");
        terminal
            .run_command("python api/sample-terminal-sessions/pause.py")
            .await
            .unwrap();

        sleep(Duration::from_millis(1_000));
        let output = terminal.read_updates().unwrap();
        assert_eq!(output, "Second\r\n");
    }
}

```

We rerun and update `src-tauri/api/sample-terminal-sessions/date.cast`, `src-tauri/api/sample-calls/run_command-date.yaml`, `src-tauri/api/sample-database-writes/command-run-date/dump.sql`, and `src-tauri/api/sample-database-writes/command-run-date/dump.yaml` to reflect the new changes -- namely, that `\r\n` now appears at the end.

We create a `src-tauri/api/sample-terminal-sessions/pause.py` that pauses for a second:

```python
#!/usr/bin/env python3

import sys
import time

print("First")
sys.stdout.flush()

time.sleep(0.5)
print("Second")
sys.stdout.flush()

```

The ensuing `src-tauri/api/sample-terminal-sessions/pause.cast` is recorded as:

```asciicast
{"version":2,"width":80,"height":24,"timestamp":1726831353,"command":"python api/sample-terminal-sessions/pause.py"}
[0.315,"o","First\r\n"]
[1.319,"o","Second\r\n"]
```

Now we do the same for input interactivity. `src-tauri/src/commands/terminal/models.rs` is once again edited to move `send_input` into the `Terminal` trait, with the `#[cfg(test)]` marker now removed:

```rs
pub trait Terminal: Send + Sync {
    ...
    async fn send_input(&mut self, input: &str) -> ZammResult<String>;
    ...
}

```

`src-tauri/src/test_helpers/terminal.rs` is edited to implement this new method, along with a new test:

```rs
#[async_trait]
impl Terminal for TestTerminal {
    ...

    async fn send_input(&mut self, input: &str) -> ZammResult<String> {
        match &mut self.terminal {
            Left(_) => {
                let input_entry = self.next_entry();
                assert_eq!(input_entry.event_type, EventType::Input);
                assert_eq!(input_entry.event_data, input);

                let output_entry = self.next_entry();
                assert_eq!(output_entry.event_type, EventType::Output);
                Ok(output_entry.event_data.clone())
            }
            Right(actual_terminal) => actual_terminal.send_input(input).await,
        }
    }

    ...
}

...

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_interactivity() {
        let mut terminal = TestTerminal::new("api/sample-terminal-sessions/bash.cast");
        terminal.run_command("bash").await.unwrap();
        let output = terminal
            .send_input("python api/sample-terminal-sessions/interleaved.py\n")
            .await
            .unwrap();
        assert_eq!(output, "stdout\r\nstderr\r\nstdout\r\nbash-3.2$ ");
    }
}
```

The recorded `src-tauri/api/sample-terminal-sessions/bash.cast` now looks like this:

```asciicast
{"version":2,"width":80,"height":24,"timestamp":1726835972,"command":"bash"}
[0.313,"o","\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "]
[0.313,"i","python api/sample-terminal-sessions/interleaved.py\n"]
[0.622,"o","stdout\r\nstderr\r\nstdout\r\nbash-3.2$ "]
```

##### Windows compatibility

The `rexpect` crate doesn't compile on Windows. While we can conditionally include it as a dependency based on platform, we decide instead to use the [`portable-pty`](https://docs.rs/portable-pty/0.8.1/portable_pty/index.html) crate, as found in [this answer](https://www.reddit.com/r/rust/comments/ip8wsk/comment/g4iw7u9/).

We do

```bash
$ cargo add portable-pty
```

and remove `rexpect` from `src-tauri/Cargo.toml`. We also remove that error from `src-tauri/src/commands/errors.rs`. Then, we edit `src-tauri/src/commands/terminal/models.rs`:

```rs
...
use portable_pty::{
    native_pty_system, Child, CommandBuilder, MasterPty, PtySize, SlavePty,
};
use std::io::{Read, Write};
...

struct PtySession {
    #[allow(dead_code)]
    master: Box<dyn MasterPty + Send>,
    #[allow(dead_code)]
    slave: Box<dyn SlavePty + Send>,
    #[allow(dead_code)]
    child: Box<dyn Child + Send + Sync>,
    reader: Box<dyn Read + Send>,
    writer: Box<dyn Write + Send>,
}

impl PtySession {
    fn new(command: &str) -> ZammResult<Self> {
        let cmd_and_args = shlex::split(command)
            .ok_or_else(|| anyhow!("Failed to split command '{}'", command))?;
        let parsed_cmd = cmd_and_args
            .first()
            .ok_or_else(|| anyhow!("Failed to get command"))?;
        let parsed_args = &cmd_and_args[1..];
        let mut cmd_builder = CommandBuilder::new(parsed_cmd);
        cmd_builder.args(parsed_args);
        let current_dir = std::env::current_dir()?;
        cmd_builder.cwd(current_dir);

        let session = native_pty_system().openpty(PtySize {
            rows: 24,
            cols: 80,
            pixel_width: 0,
            pixel_height: 0,
        })?;
        let child = session.slave.spawn_command(cmd_builder)?;
        let reader = session.master.try_clone_reader()?;
        let writer = session.master.take_writer()?;
        Ok(Self {
            master: session.master,
            slave: session.slave,
            child,
            reader,
            writer,
        })
    }
}

pub struct ActualTerminal {
    session: Option<Arc<Mutex<PtySession>>>,
    ...
}

#[async_trait]
impl Terminal for ActualTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        ...
        let session = PtySession::new(command)?;
        ...
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        let output = {
            let session_mutex = ...;
            let mut session = session_mutex.lock()?;
            let mut partial_output = String::new();
            let buf = &mut [0; 1024];
            let mut bytes_read = session.reader.read(buf)?;
            while bytes_read > 0 {
                partial_output.push_str(&String::from_utf8_lossy(&buf[..bytes_read]));
                sleep(Duration::from_millis(100));
                bytes_read = session.reader.read(buf)?;
            }
            partial_output
        };

        if !output.is_empty() {
            ...
        }
        ...
    }

    async fn send_input(&mut self, input: &str) -> ZammResult<String> {
        ...
        {
            let mut session = ...;
            session.writer.write_all(input.as_bytes())?;
            session.writer.flush()?;
        }

        ...
    }

    ...
}
```

where `drain_read_buffer` is no longer needed. Now the `test_capture_command_output` and `test_capture_interleaved_output` tests pass, but `test_capture_output_without_blocking` and so on still fail by blocking.

We try to see if we can patch the dependency. We do

```bash
$ git submodule add https://github.com/wez/wezterm.git
Cloning into '/Users/amos/Documents/zamm/forks/wezterm'...
remote: Enumerating objects: 73366, done.
remote: Counting objects: 100% (5404/5404), done.
remote: Compressing objects: 100% (511/511), done.
remote: Total 73366 (delta 5088), reused 5030 (delta 4889), pack-reused 67962 (from 1)
Receiving objects: 100% (73366/73366), 298.05 MiB | 2.10 MiB/s, done.
Resolving deltas: 100% (51881/51881), done.
```

 We add the following to `src-tauri/Cargo.toml`:

```toml
[patch.crates-io]
...
portable-pty = { path = "../forks/wezterm/pty" }

```

We now get the error

```bash
$ cargo build
    Updating crates.io index
error: failed to select a version for `libc`.
    ... required by package `nix v0.28.0`
    ... which satisfies dependency `nix = "^0.28"` of package `portable-pty v0.8.1 (/Users/amos/Documents/zamm/forks/wezterm/pty)`
    ... which satisfies dependency `portable-pty = "^0.8.1"` (locked to 0.8.1) of package `zamm v0.1.7 (/Users/amos/Documents/zamm/src-tauri)`
versions that meet the requirements `^0.2.153` are: 0.2.158, 0.2.157, 0.2.156, 0.2.155, 0.2.153

all possible versions conflict with previously selected packages.

  previously selected package `libc v0.2.152`
    ... which satisfies dependency `libc = "^0.2"` (locked to 0.2.152) of package `portable-pty v0.8.1 (/Users/amos/Documents/zamm/forks/wezterm/pty)`
    ... which satisfies dependency `portable-pty = "^0.8.1"` (locked to 0.8.1) of package `zamm v0.1.7 (/Users/amos/Documents/zamm/src-tauri)`

failed to select a version for `libc` which could resolve this conflict
```

We try editing `Cargo.toml` again:

```toml
[dependencies]
...
libc = "0.2.153"
```

Now it compiles. It appears there have been [previous complaints](https://github.com/rust-lang/cargo/issues/10189) about the resolver not being good enough.

We try editing `forks/wezterm/pty/src/unix.rs` to [set a non-blocking file descriptor](https://docs.rs/filedescriptor/0.8.1/filedescriptor/struct.FileDescriptor.html#method.set_non_blocking):

```rs
fn openpty(size: PtySize) -> anyhow::Result<(UnixMasterPty, UnixSlavePty)> {
    ...

    let master = UnixMasterPty {
        fd: PtyFd(unsafe { 
            let mut master_fd = FileDescriptor::from_raw_fd(master);
            master_fd.set_non_blocking(true)?;
            master_fd
        }),
        ...
    };
    ...
}
```

Unfortunately, our tests now just fail with

```
---- commands::terminal::models::tests::test_capture_output_without_blocking stdout ----
thread 'commands::terminal::models::tests::test_capture_output_without_blocking' panicked at src/commands/terminal/models.rs:190:57:
called `Result::unwrap()` on an `Err` value: Io { source: Os { code: 35, kind: WouldBlock, message: "Resource temporarily unavailable" } }
```

We reset. It appears that there's [no way](https://github.com/wez/wezterm/discussions/3739) around the fact that pty reads are blocking; we'll have to create a separate thread. There's also [no way](https://users.rust-lang.org/t/tokio-timeout-not-timeouting/85895/7) to enforce a timeout on CPU reads.

As such, we edit `src-tauri/src/commands/terminal/models.rs`, removing `reader` and replacing it with `input_receiver`, and now getting input echo. We use [this function](https://doc.rust-lang.org/std/sync/mpsc/struct.Receiver.html#method.try_recv) along with [this example](https://doc.rust-lang.org/rust-by-example/std_misc/channels.html).

```rs
...
use std::sync::mpsc;
use std::sync::mpsc::{Receiver, Sender};
...
use std::thread::{..., spawn};
...

struct PtySession {
    ...
    input_receiver: Receiver<char>,
}

impl PtySession {
    fn new(command: &str) -> ZammResult<Self> {
        ...

        let (tx, rx): (Sender<char>, Receiver<char>) = mpsc::channel();
        let mut reader = session.master.try_clone_reader()?;
        spawn(move || {
            let buf = &mut [0; 1];
            loop {
                let bytes_read = reader.read(buf).unwrap();
                if bytes_read == 0 {
                    break;
                }
                tx.send(buf[0] as char).unwrap();
            }
        });

        ...

        Ok(Self {
            ...,
            input_receiver: rx,
        })
    }
}

...

impl ActualTerminal {
    ...

    fn read_once(&mut self) -> ZammResult<String> {
        let session_mutex = self.session.as_mut().ok_or(anyhow!("No session"))?;
        let session = session_mutex.lock()?;
        let mut partial_output = String::new();
        while let Ok(c) = session.input_receiver.try_recv() {
            partial_output.push(c);
        }
        Ok(partial_output)
    }
}

#[async_trait]
impl Terminal for ActualTerminal {
    ...

    fn read_updates(&mut self) -> ZammResult<String> {
        let output = {
            let mut partial_output = String::new();
            loop {
                sleep(Duration::from_millis(100));
                let partial = self.read_once()?;
                if partial.is_empty() {
                    break;
                }
                partial_output.push_str(&partial);
            }
            partial_output
        };

        ...
    }

    ...
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_interaction() {
        ...

        let output = ...;
        assert_eq!(output, "python api/sample-terminal-sessions/interleaved.py\r\nstdout\r\nstderr\r\nstdout\r\nbash-3.2$ ");
        ...
    }
}

```

`src-tauri/api/sample-terminal-sessions/bash.cast` gets updated again:

```asciicast
{"version":2,"width":80,"height":24,"timestamp":1727195245,"command":"bash"}
[0.208,"o","\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "]
[0.208,"i","python api/sample-terminal-sessions/interleaved.py\n"]
[0.412,"o","python api/sample-terminal-sessions/interleaved.py\r\nstdout\r\nstderr\r\nstdout\r\nbash-3.2$ "]
```

and with that, the playback test at `src-tauri/src/test_helpers/terminal.rs` needs to be updated again:

```rs
    #[tokio::test]
    async fn test_interactivity() {
        ...
        assert_eq!(
            output,
            "python api/sample-terminal-sessions/interleaved.py\r\nstdout\r\nstderr\r\nstdout\r\nbash-3.2$ "
        );
    }
```

Now when we run on Windows, we finally do get failing but compiling tests. We edit `src-tauri/src/commands/terminal/models.rs` as such:

```rs
mod tests {
    ...

    #[cfg(target_os = "windows")]
    const SHELL_COMMAND: &str = "cmd";
    #[cfg(not(target_os = "windows"))]
    const SHELL_COMMAND: &str = "bash";

    #[tokio::test]
    async fn test_capture_command_output() {
        let (command, expected_output) = if cfg!(target_os = "windows") {
            ("cmd /C \"echo hello world\"", "\u{1b}[?25l\u{1b}[2J\u{1b}[m\u{1b}[Hhello world\r\n\u{1b}]0;C:\\WINDOWS\\system32\\cmd.EXE\u{7}\u{1b}[?25h")
        } else {
            ("echo hello world", "hello world\r\n")
        };

        let mut terminal = ActualTerminal::new();
        let output = terminal.run_command(command).await.unwrap();
        assert_eq!(output, expected_output);
        ...
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        ...

        // No trailing newline on Windows
        #[cfg(target_os = "windows")]
        assert!(
            output.contains("stdout\r\nstderr\r\nstdout"),
            "Output: {:?}",
            output
        );
        #[cfg(not(target_os = "windows"))]
        assert_eq!(output, "stdout\r\nstderr\r\nstdout\r\n");

        ...
    }

    #[tokio::test]
    async fn test_capture_output_without_blocking() {
        let mut terminal = ActualTerminal::new();
        let output = terminal.run_command(SHELL_COMMAND).await.unwrap();

        // Windows output contains a whole lot of control characters, so we don't test
        // directly with `starts_with` or `ends_with` here
        #[cfg(target_os = "windows")]
        assert!(
            output.contains("(c) Microsoft Corporation. All rights reserved.")
                && output.contains("src-tauri>"),
            "Output: {:?}",
            output
        );
        #[cfg(not(target_os = "windows"))]
        assert!(
            output.ends_with("$ ") || output.ends_with("# "),
            "Output: {}",
            output
        );

        ...
    }

    #[tokio::test]
    async fn test_no_entry_on_empty_capture() {
        ...
        terminal.run_command(SHELL_COMMAND).await.unwrap();
        ...
    }

    #[tokio::test]
    async fn test_capture_interaction() {
        let input = if cfg!(target_os = "windows") {
            "python api/sample-terminal-sessions/interleaved.py\r\n"
        } else {
            "python api/sample-terminal-sessions/interleaved.py\n"
        };

        let mut terminal = ActualTerminal::new();
        terminal.run_command(SHELL_COMMAND).await.unwrap();

        let output = terminal.send_input(input).await.unwrap();
        #[cfg(target_os = "windows")]
        assert!(
            output.contains("stdout\r\nstderr\r\nstdout"),
            "Output: {:?}",
            output
        );
        #[cfg(not(target_os = "windows"))]
        assert!(
            output.contains("stdout\r\nstderr\r\nstdout\r\n")
                && (output.ends_with("$ ") || output.ends_with("# ")),
            "Output: {:?}",
            output
        );

        ...
    }
}
```

Now all tests on Windows pass. It fails on the Mac again because we accidentally left out the `\r` when we edited `test_capture_command_output`, so we fix that.

##### Linux compatibility

Finally, we edit this test to get an assert that will pass on both Linux and Mac:

```rs
    #[tokio::test]
    async fn test_capture_interaction() {
        ...

        #[cfg(not(target_os = "windows"))]
        assert!(
            output.contains("stdout\r\nstderr\r\nstdout\r\n")
                && (output.ends_with("$ ") || output.ends_with("# ")),
            "Output: {:?}",
            output
        );

        ...
    }
```

##### Exit codes

We edit `src-tauri/src/commands/terminal/models.rs` to record exit codes and exit time:

```rs
struct PtySession {
    ...
    exit_code: Option<u32>,
}

impl PtySession {
    fn new(command: &str) -> ZammResult<Self> {
        ...
        Ok(Self {
            ...,
            exit_code: None,
        })
    }
}

...

impl ActualTerminal {
    ...

    #[allow(dead_code)]
    fn exit_code(&self) -> Option<u32> {
        let session_mutex = self.session.as_ref().unwrap();
        let mut session = session_mutex.lock().unwrap();
        match session.exit_code {
            Some(code) => Some(code),
            None => {
                let status = session
                    .child
                    .try_wait()
                    .unwrap_or(None)
                    .map(|status| status.exit_code());
                if let Some(code) = status {
                    session.exit_code = Some(code);
                    // todo: record exit code and total runtime
                }
                status
            }
        }
    }
}

...

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_command_output() {
        ...
        assert_eq!(terminal.exit_code(), Some(0));
    }

    #[tokio::test]
    async fn test_capture_interleaved_output() {
        ...
        assert_eq!(terminal.exit_code(), Some(0));
    }

    #[tokio::test]
    async fn test_capture_output_without_blocking() {
        ...
        assert_eq!(terminal.exit_code(), None);
    }

    ...

    #[tokio::test]
    async fn test_capture_interaction() {
        ...
        assert_eq!(terminal.exit_code(), None);
    }
}
```

As it turns out, the `asciicast::EventType` enum does not include the "marker" type that is specified in the asciicast format, and the asciicast format does not include exit codes. We'll either have to fork asciicast to add a new type to the enum, or we'll have to extend the Asciicast Header type to include exit codes. We avoid doing this for now.

#### Storing terminal sessions

Now we store live terminal sessions. We edit `src-tauri/src/main.rs` to store this new data:

```rs
...
use std::collections::HashMap;
...
use commands::terminal::Terminal;
...
use models::llm_calls::EntityId;
...

pub struct ZammApiKeys(...);
pub struct ZammTerminalSessions(Mutex<HashMap<EntityId, Box<dyn Terminal>>>);

fn main() {
    ...
        Some(Commands::Gui {}) | None => {
            ...
            let api_keys = ...;
            let terminal_sessions = HashMap::new();

            tauri::Builder::default()
                ...
                .manage(ZammTerminalSessions(Mutex::new(terminal_sessions)))
                ...;
}
```

We edit `src-tauri/src/commands/terminal/run.rs` accordingly to use this data:

```rs
...
use crate::{..., ZammTerminalSessions};
...

async fn run_command_helper(
    zamm_db: ...,
    zamm_sessions: &ZammTerminalSessions,
    session_id: &EntityId,
    command: &str,
) -> ZammResult<RunCommandResponse> {
    ...
    let mut sessions = zamm_sessions.0.lock().await;
    let terminal = sessions
        .get_mut(session_id)
        .ok_or_else(|| anyhow!("No session found"))?;
    ...

    if let Some(conn) = db.as_mut() {
        ...
        diesel::insert_into(asciicasts::table)
            .values(NewAsciiCast {
                id: session_id,
                ...,
                cast: &cast,
            })
            .execute(conn)?;
    }

    Ok(RunCommandResponse {
        id: session_id.clone(),
        ...
    })
}

#[tauri::command(async)]
#[specta]
pub async fn run_command(
    database: ...,
    sessions: State<'_, ZammTerminalSessions>,
    command: ...,
) -> ZammResult<RunCommandResponse> {
    let terminal = ActualTerminal::new();
    let new_session_id = EntityId::new();
    sessions
        .0
        .lock()
        .await
        .insert(new_session_id.clone(), Box::new(terminal));

    run_command_helper(&database, &sessions, &new_session_id, &command).await
}

#[cfg(test)]
mod tests {
    ...

    impl SampleCallTestCase<RunCommandRequest, ZammResult<RunCommandResponse>>
        for RunCommandTestCase
    {
        ...

        async fn make_request(
            ...
        ) -> ZammResult<RunCommandResponse> {
            let terminal_helper = side_effects.terminal.as_ref().unwrap();
            run_command_helper(
                side_effects.db.as_ref().unwrap(),
                &terminal_helper.sessions,
                &terminal_helper.mock_session_id,
                &args.command,
            )
            .await
        }

        fn output_replacements(
            &self,
            sample: &SampleCall,
            result: &ZammResult<RunCommandResponse>,
        ) -> HashMap<String, String> {
            ...
            let expected_os = if sample
                .side_effects
                .as_ref()
                .unwrap()
                .terminal
                .as_ref()
                .unwrap()
                .recording_file
                .ends_with("bash.cast")
            {
                "Mac"
            } else {
                "Linux"
            };
            HashMap::from([
                ...,
                #[cfg(target_os = "windows")]
                ("Windows".to_string(), expected_os.to_string()),
                #[cfg(target_os = "macos")]
                ("Mac".to_string(), expected_os.to_string()),
                #[cfg(target_os = "linux")]
                ("Linux".to_string(), expected_os.to_string()),
            ])
        }

        ...
    }

    ...

    check_sample!(
        RunCommandTestCase,
        test_start_bash,
        "./api/sample-calls/run_command-bash.yaml"
    );
}
```

where `src-tauri/api/sample-calls/run_command-bash.yaml` is a new API recording we've created that looks like this:

```yaml
request:
  - run_command
  - >
    {
      "command": "bash"
    }
response:
  message: >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118",
      "timestamp": "2024-09-24T16:27:25",
      "output": "\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "
    }
sideEffects:
  database:
    endStateDump: command-run-bash
  terminal:
    recordingFile: bash.cast
```

Note that because this is run on the Mac, we change the test to expect a different operating system based on the recording filename.

We edit `src-tauri/src/models/llm_calls/entity_id.rs` to implement the `new` function:

```rs
impl Default for EntityId {
    fn default() -> Self {
        Self::new()
    }
}
impl EntityId {
    pub fn new() -> Self {
        EntityId {
            uuid: Uuid::new_v4(),
        }
    }
}
```

where the `Default` is automatically implemented by Clippy.

We also have to edit `src-tauri/src/test_helpers/api_testing.rs`:

```rs
pub struct TerminalHelper {
    pub sessions: ZammTerminalSessions,
    pub mock_session_id: EntityId,
}

#[derive(Default)]
pub struct SideEffectsHelpers {
    ...
    pub terminal: Option<TerminalHelper>,
}

    async fn check_sample_call(...) -> SampleCallResult<T, U> {
            // prepare terminal if necessary
            if side_effects.terminal.is_some() {
                ...
                let new_session_id = EntityId::new();
                let sessions = ZammTerminalSessions(Mutex::new(HashMap::from([(
                    new_session_id.clone(),
                    terminal as Box<dyn Terminal>,
                )])));
                side_effects_helpers.terminal = Some(TerminalHelper {
                    sessions,
                    mock_session_id: new_session_id,
                });
            }

            ...
    }
```

We see that `src-tauri/api/sample-database-writes/command-run-bash/dump.sql` looks like this:

```sql
INSERT INTO asciicasts VALUES('3717ed48-ab52-4654-9f33-de5797af5118','2024-09-24 16:27:25','bash','Mac',replace('{"version":2,"width":80,"height":24,"timestamp":1727195245,"command":"bash"}\012[0.208,"o","\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "]','\012',char(10)));
```

We see that the use of the `replace` function is [expected](https://stackoverflow.com/q/44989176) from changes to SQLite. The corresponding `src-tauri/api/sample-database-writes/command-run-bash/dump.yaml` is:

```yaml
terminal_sessions:
- id: 3717ed48-ab52-4654-9f33-de5797af5118
  timestamp: 2024-09-24T16:27:25
  command: bash
  os: Mac
  cast:
    header:
      version: 2
      width: 80
      height: 24
      timestamp: 1727195245
      command: bash
    entries:
    - - 0.208
      - 'o'
      - "\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "
```

Initially, the entries returned the *full* AsciiCast, not just the cast so far. As such, this requires editing `src-tauri/src/commands/terminal/models.rs` to return a new struct instead of a reference to one:

```rs
pub trait Terminal: Send + Sync {
    ...
    fn get_cast(&self) -> AsciiCastData;
}

...

impl Terminal for ActualTerminal {
    ...

    fn get_cast(&self) -> AsciiCastData {
        self.session_data.clone()
    }
}
```

We also need to edit `src-tauri/src/test_helpers/terminal.rs` to do the same:

```rs
    fn get_cast(&self) -> AsciiCastData {
        match &self.terminal {
            Left(cast) => AsciiCastData {
                header: cast.header.clone(),
                entries: cast.entries[..self.entry_index].to_vec(),
            },
            ...
        }
    }
```

#### Continuing from stored terminal sessions

We update `src-tauri/api/sample-call-schema.json` to allow for the current state of the terminal recording to be specified as well:

```json
            "terminal": {
              ...,
              "properties": {
                ...,
                "startingIndex": {
                  "type": "integer"
                }
              },
              ...
            }
```

`src-tauri/src/sample_call.rs` gets updated as well when we run `make quicktype`. We have to edit `src-tauri/src/test_helpers/api_testing.rs` as well to make use of this new functionality:

```rs
            // prepare terminal if necessary
            if let Some(terminal_info) = &side_effects.terminal {
                let recording_path = ...;
                let mut terminal =
                    ...;
                if let Some(recording_index) = terminal_info.starting_index {
                    terminal.set_entry_index(recording_index.try_into().unwrap());
                }

                ...
            }
```

and to add this functionality to `src-tauri/src/test_helpers/terminal.rs`:

```rs
impl TestTerminal {
    ...

    pub fn set_entry_index(&mut self, index: usize) {
        self.entry_index = index;
    }

    ...
}
```

During implementation of `src-tauri/src/commands/terminal/send_input.rs`, we run into

```
error[E0581]: return type references an anonymous lifetime, which is not constrained by the fn input types
  --> src/commands/terminal/send_input.rs:19:1
   |
19 | / pub async fn send_input(
20 | |     database: State<'_, ZammDatabase>,
21 | |     sessions: State<'_, ZammTerminalSessions>,
22 | |     session_id: String,
23 | |     input: String,
24 | | ) -> ZammResult<String> {
   | |_______________________^
   |
   = note: lifetimes appearing in an associated or opaque type are not considered constrained
   = note: consider introducing a named lifetime parameter
```

It turns out this is because we haven't yet imported `use tauri::State;` in the new file. We end up with a file that looks like this:

```rs
use crate::commands::errors::ZammResult;
use crate::models::llm_calls::EntityId;
use crate::schema::asciicasts;

use crate::{ZammDatabase, ZammTerminalSessions};
use anyhow::anyhow;
use diesel::prelude::*;
use specta::specta;
use tauri::State;
use uuid::Uuid;

async fn send_command_input_helper(
    zamm_db: &ZammDatabase,
    zamm_sessions: &ZammTerminalSessions,
    session_id: &Uuid,
    input: &str,
) -> ZammResult<String> {
    let db = &mut zamm_db.0.lock().await;
    let mut sessions = zamm_sessions.0.lock().await;
    let session_entity_id = EntityId { uuid: *session_id };
    let terminal = sessions
        .get_mut(&session_entity_id)
        .ok_or_else(|| anyhow!("No session found"))?;
    let result = terminal.send_input(input).await?;

    if let Some(conn) = db.as_mut() {
        let result = diesel::update(asciicasts::table)
            .filter(asciicasts::id.eq(session_entity_id))
            .set(asciicasts::cast.eq(terminal.get_cast()))
            .execute(conn)?;
        if result == 0 {
            return Err(anyhow!("Couldn't update session in database").into());
        }
    }

    Ok(result)
}

#[tauri::command(async)]
#[specta]
pub async fn send_command_input(
    database: State<'_, ZammDatabase>,
    sessions: State<'_, ZammTerminalSessions>,
    session_id: Uuid,
    input: String,
) -> ZammResult<String> {
    send_command_input_helper(&database, &sessions, &session_id, &input).await
}

#[cfg(test)]
mod tests {
    use super::*;

    use crate::test_helpers::SideEffectsHelpers;
    use crate::{check_sample, impl_result_test_case};
    use serde::{Deserialize, Serialize};

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct SendInputRequest {
        session_id: Uuid,
        input: String,
    }

    async fn make_request_helper(
        args: &SendInputRequest,
        side_effects: &mut SideEffectsHelpers,
    ) -> ZammResult<String> {
        let terminal_helper = side_effects.terminal.as_mut().unwrap();
        let mock_terminal = terminal_helper
            .sessions
            .0
            .lock()
            .await
            .remove(&terminal_helper.mock_session_id)
            .unwrap();
        terminal_helper.sessions.0.lock().await.insert(
            EntityId {
                uuid: args.session_id,
            },
            mock_terminal,
        );
        send_command_input_helper(
            side_effects.db.as_mut().unwrap(),
            &terminal_helper.sessions,
            &args.session_id,
            &args.input,
        )
        .await
    }

    impl_result_test_case!(
        SendInputTestCase,
        send_command_input,
        true,
        SendInputRequest,
        String
    );

    check_sample!(
        SendInputTestCase,
        test_bash_interleaved,
        "./api/sample-calls/send_command_input-bash-interleaved.yaml"
    );
}

```

Note that we have to swap the key of the hashmap in the test, because otherwise the right database row won't be updated if we're only updating the wrong session ID in the hashmap.

We export this command in `src-tauri/src/commands/terminal/mod.rs`:

```rs
...
mod send_input;

...
pub use send_input::send_command_input;
```

and again in `src-tauri/src/commands/mod.rs`:

```rs
pub use terminal::{..., send_command_input};
```

and consume it in `src-tauri/src/main.rs`:

```rs
...
use commands::{
    ...
    send_command_input,
    ...
};

fn main() {
    ...
            ts::export(
                collect_types![
                    ...,
                    send_command_input,
                ],
                ...
            );
    ...
            tauri::Builder::default()
                ...
                .invoke_handler(tauri::generate_handler![
                    ...
                    send_command_input,
                    ...
                ])
                ...;
}
```

The request at `src-tauri/api/sample-calls/send_command_input-bash-interleaved.yaml` looks like this:

```yaml
request:
  - send_command_input
  - >
    {
      "session_id": "3717ed48-ab52-4654-9f33-de5797af5118",
      "input": "python api/sample-terminal-sessions/interleaved.py\n"
    }
response:
  message: >
    "python api/sample-terminal-sessions/interleaved.py\r\nstdout\r\nstderr\r\nstdout\r\nbash-3.2$ "
sideEffects:
  database:
    startStateDump: command-run-bash
    endStateDump: command-run-bash-interleaved
  terminal:
    recordingFile: bash.cast
    startingIndex: 1
```

The resulting dump at `src-tauri/api/sample-database-writes/command-run-bash-interleaved/dump.sql` contains the new input/output lines:

```sql
INSERT INTO asciicasts VALUES('3717ed48-ab52-4654-9f33-de5797af5118','2024-09-24 16:27:25','bash','Mac',replace('{"version":2,"width":80,"height":24,"timestamp":1727195245,"command":"bash"}\012[0.208,"o","\r\nThe default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ "]\012[0.208,"i","python api/sample-terminal-sessions/interleaved.py\n"]\012[0.412,"o","python api/sample-terminal-sessions/interleaved.py\r\nstdout\r\nstderr\r\nstdout\r\nbash-3.2$ "]','\012',char(10)));
```

Ditto for the YAML dump at `src-tauri/api/sample-database-writes/command-run-bash-interleaved/dump.yaml`.

#### Flaky tests

We notice that we get this failing test once on the Mac:

```
---- commands::terminal::models::tests::test_capture_interaction stdout ----
thread 'commands::terminal::models::tests::test_capture_interaction' panicked at src/commands/terminal/models.rs:301:9:
Output: "python api/sample-terminal-sessions/interleaved.py\r\n"
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread '<unnamed>' panicked at src/commands/terminal/models.rs:64:41:
called `Result::unwrap()` on an `Err` value: SendError { .. }
```

A rerun failed to produce the same error again.

However, sometimes we get

```
---- commands::terminal::models::tests::test_capture_interaction stdout ----
thread 'commands::terminal::models::tests::test_capture_interaction' panicked at src/commands/terminal/models.rs:308:9:
assertion `left == right` failed
  left: 2
 right: 3
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread '<unnamed>' panicked at src/commands/terminal/models.rs:64:41:
called `Result::unwrap()` on an `Err` value: SendError { .. }
```

on CI. This happens rarely, but still frequently enough in CI that we should address it because it may point to issues around concurrency. We try to edit `src-tauri/src/commands/terminal/models.rs` to introduce another mutex on the other field of `ActualTerminal`:

```rs
pub struct ActualTerminal {
    session: Option<Arc<Mutex<PtySession>>>,
    session_data: Arc<Mutex<AsciiCastData>>,
}
```

Our first implementation:

```rs
#[async_trait]
impl Terminal for ActualTerminal {
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        if self.session.is_some() {
            return Err(anyhow!("Session already started").into());
        }

        let mut session_data = self.session_data.lock()?;
        session_data.header.command = Some(command.to_string());

        let starting_time = chrono::Utc::now();
        session_data.header.timestamp = Some(starting_time);

        let session = PtySession::new(command)?;
        self.session = Some(Arc::new(Mutex::new(session)));

        let result = self.read_updates()?;
        Ok(result)
    }

    ...
}
```

runs into the issue

```
error[E0502]: cannot borrow `*__self` as mutable because it is also borrowed as immutable
   --> src/commands/terminal/models.rs:151:22
    |
135 | #[async_trait]
    |              - immutable borrow might be used here, when `session_data` is dropped and runs the `Drop` code for type `std::sync::MutexGuard`
...
142 |         let mut session_data = self.session_data.lock()?;
    |                                ----------------- immutable borrow occurs here
...
151 |         let result = self.read_updates()?;
    |                      ^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

It turns out we just need to wrap that section:


```rs
    async fn run_command(&mut self, command: &str) -> ZammResult<String> {
        ...

        {
            let mut session_data = self.session_data.lock()?;
            session_data.header.command = Some(command.to_string());

            let starting_time = chrono::Utc::now();
            session_data.header.timestamp = Some(starting_time);
        }

        ...

        let result = self.read_updates()?;
        ...
    }
```

so that the lock on `self.session_data` is dropped by the time it is needed again in `self.read_updates`. This now compiles, but it deadlocks on unit tests. We try putting everything in one single inner struct that we then lock, but then we run into the error

```
error: future cannot be sent between threads safely
   --> src/commands/terminal/models.rs:228:71
    |
228 |       async fn send_input(&mut self, input: &str) -> ZammResult<String> {
    |  _______________________________________________________________________^
229 | |         let mut inner = self.inner.lock()?;
230 | |         inner.send_input(input).await
231 | |     }
    | |_____^ future created by async block is not `Send`
    |
    = help: within `{async block@src/commands/terminal/models.rs:228:71: 231:6}`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, ActualTerminalInner>`
note: future is not `Send` as this value is used across an await
   --> src/commands/terminal/models.rs:230:33
    |
229 |         let mut inner = self.inner.lock()?;
    |             --------- has type `std::sync::MutexGuard<'_, ActualTerminalInner>` which is not `Send`
230 |         inner.send_input(input).await
    |                                 ^^^^^ await occurs here, with `mut inner` maybe used later
    = note: required for the cast from `Pin<Box<{async block@src/commands/terminal/models.rs:228:71: 231:6}>>` to `Pin<Box<dyn futures::Future<Output = Result<std::string::String, errors::Error>> + std::marker::Send>>`
```

From [this thread](https://www.reddit.com/r/rust/comments/pnzple/comment/hct59dj/), it appears that we should try removing the async nature of the function. We realize that there's no longer any particular reason to keep these functions async, so we remove the async signatures altogether and remove the `await` from the test functions. We also make the result of the exit code function `ZammResult` because it also needs to acquire a lock. We edit every function in inner to remove the locking functionality, and we edit every function in outer to use locking. We edit `src-tauri/src/commands/terminal/models.rs`:

```rs
pub trait Terminal: Send + Sync {
    fn run_command(&mut self, command: &str) -> ZammResult<String>;
    fn read_updates(&mut self) -> ZammResult<String>;
    fn send_input(&mut self, input: &str) -> ZammResult<String>;
    fn get_cast(&self) -> ZammResult<AsciiCastData>;
}

...

pub struct ActualTerminalInner {
    session: Option<PtySession>,
    session_data: AsciiCastData,
}

impl ActualTerminalInner {
    pub fn new() -> Self {
        Self {
            session: None,
            session_data: AsciiCastData::new(),
        }
    }

    fn start_time(&self) -> ZammResult<DateTime<chrono::Utc>> {
        let result = self
            .session_data
            .header
            .timestamp
            .ok_or(anyhow!("No timestamp"))?;
        Ok(result)
    }

    fn read_once(&mut self) -> ZammResult<String> {
        match self.session.as_mut() {
            None => Err(anyhow!("No session started").into()),
            Some(session) => {
                let mut partial_output = String::new();
                while let Ok(c) = session.input_receiver.try_recv() {
                    partial_output.push(c);
                }
                Ok(partial_output)
            }
        }
    }

    fn run_command(&mut self, command: &str) -> ZammResult<String> {
        if self.session.is_some() {
            return Err(anyhow!("Session already started").into());
        }

        self.session_data.header.command = Some(command.to_string());
        self.session_data.header.timestamp = Some(chrono::Utc::now());

        let session = PtySession::new(command)?;
        self.session = Some(session);

        let result = self.read_updates()?;
        Ok(result)
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        let output = {
            let mut partial_output = String::new();
            loop {
                sleep(Duration::from_millis(100));
                let partial = self.read_once()?;
                if partial.is_empty() {
                    break;
                }
                partial_output.push_str(&partial);
            }
            partial_output
        };

        if !output.is_empty() {
            let output_time = chrono::Utc::now();
            let relative_time = output_time - self.start_time()?;
            self.session_data.entries.push(asciicast::Entry {
                time: relative_time.num_milliseconds() as f64 / 1000.0,
                event_type: asciicast::EventType::Output,
                event_data: output.clone(),
            });
        }
        Ok(output)
    }

    fn send_input(&mut self, input: &str) -> ZammResult<String> {
        match self.session.as_mut() {
            None => Err(anyhow!("No session started").into()),
            Some(session) => {
                session.writer.write_all(input.as_bytes())?;
                session.writer.flush()?;

                let relative_time = chrono::Utc::now() - self.start_time()?;
                self.session_data.entries.push(asciicast::Entry {
                    time: relative_time.num_milliseconds() as f64 / 1000.0,
                    event_type: asciicast::EventType::Input,
                    event_data: input.to_string(),
                });

                self.read_updates()
            }
        }
    }

    fn get_cast(&self) -> &AsciiCastData {
        &self.session_data
    }

    fn exit_code(&mut self) -> Option<u32> {
        match self.session.as_mut() {
            None => None,
            Some(session) => {
                if let Some(code) = session.exit_code {
                    Some(code)
                } else {
                    let status = session
                        .child
                        .try_wait()
                        .unwrap_or(None)
                        .map(|status| status.exit_code());
                    if let Some(code) = status {
                        session.exit_code = Some(code);
                        // todo: record exit code and total runtime
                    }
                    status
                }
            }
        }
    }
}

pub struct ActualTerminal {
    inner: Arc<Mutex<ActualTerminalInner>>,
}

impl ActualTerminal {
    pub fn new() -> Self {
        Self {
            inner: Arc::new(Mutex::new(ActualTerminalInner::new())),
        }
    }

    #[allow(dead_code)]
    fn exit_code(&self) -> ZammResult<Option<u32>> {
        let mut inner = self.inner.lock()?;
        Ok(inner.exit_code())
    }
}

impl Terminal for ActualTerminal {
    fn run_command(&mut self, command: &str) -> ZammResult<String> {
        let mut inner = self.inner.lock()?;
        inner.run_command(command)
    }

    fn read_updates(&mut self) -> ZammResult<String> {
        let mut inner = self.inner.lock()?;
        inner.read_updates()
    }

    fn send_input(&mut self, input: &str) -> ZammResult<String> {
        let mut inner = self.inner.lock()?;
        inner.send_input(input)
    }

    fn get_cast(&self) -> ZammResult<AsciiCastData> {
        let inner = self.inner.lock()?;
        Ok(inner.get_cast().clone())
    }
}

#[cfg(test)]
mod tests {
    ...

    #[tokio::test]
    async fn test_capture_command_output() {
        ...
        let output = terminal.run_command(command).unwrap();
        ...
        assert_eq!(terminal.exit_code().unwrap(), Some(0));
    }

    ...
}
```

We take this change to also refactor the relative time calculations into a single function:

```rs
impl ActualTerminalInner {
    ...

    fn relative_time(&self) -> ZammResult<f64> {
        let time_diff = chrono::Utc::now() - self.start_time()?;
        Ok(time_diff.num_milliseconds() as f64 / 1000.0)
    }

    ...

    fn read_updates(&mut self) -> ZammResult<String> {
        ...

        if !output.is_empty() {
            self.session_data.entries.push(asciicast::Entry {
                time: self.relative_time()?,
                ...
            });
        }
        ...
    }

    fn send_input(&mut self, input: &str) -> ZammResult<String> {
        match self.session.as_mut() {
            None => ...,
            Some(session) => {
                ...

                self.session_data.entries.push(asciicast::Entry {
                    time: self.relative_time()?,
                    ...
                });

                ...
            }
        }
    }

    ...
}
```

Unfortunately, while our code is cleaner now, it still doesn't fix the problem on CI. It is possible that the problem on CI is simply that the read times out before the process has a chance to respond. As such, we edit the file again:

```rs
#[cfg(test)]
const TERMINAL_READ_TIMEOUT: Duration = Duration::from_millis(500);
#[cfg(not(test))]
const TERMINAL_READ_TIMEOUT: Duration = Duration::from_millis(100);

...

    fn read_updates(&mut self) -> ZammResult<String> {
        let output = {
            ...
            loop {
                sleep(TERMINAL_READ_TIMEOUT);
                ...
            }
            ...
        };

        ...
    }
```

#### Frontend

We update the bindings again with `cargo run -- export-bindings`.

We refactor `src-svelte/src/routes/chat/Form.svelte` into `src-svelte/src/lib/controls/SendInputForm.svelte`, in the process renaming `submitChat` to `submitInput`:

```svelte
<script lang="ts">
  ...
  export let sendInput: (message: string) => void;
  export let accessibilityLabel: string;
  export let isBusy = false;
  export let currentMessage = "";
  export let placeholder = "Type your message here...";
  ...

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Enter" && ...) {
      ...
      submitInput();
    }
  }

  function submitInput() {
    ...
    if (message && !isBusy) {
      sendInput(currentMessage);
      ...
    }
  }
</script>

...

<form
  ...
  on:submit|preventDefault={submitInput}
>
  <label for="message" ...>{accessibilityLabel}</label>
  <textarea
    ...
    {placeholder}
    ...
  />
  <Button ... disabled={isBusy} ...
    >...</Button
  >
</form>
```

We edit `src-svelte/src/routes/chat/Chat.svelte` to use this newly refactored component:

```svelte
...

<script lang="ts">
  ...
  import SendInputForm from "$lib/controls/SendInputForm.svelte";
  ...
</script>

...
    <SendInputForm
      accessibilityLabel="Chat with the AI:"
      sendInput={sendChatMessage}
      isBusy={expectingResponse}
      ...
    />
...
```

We can now use this in `src-svelte/src/routes/terminal/TerminalSession.svelte`, where we use the same resize logic as the Chat component:

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import SendInputForm from "$lib/controls/SendInputForm.svelte";
  import { runCommand, sendCommandInput } from "$lib/bindings";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";
  import EmptyPlaceholder from "$lib/EmptyPlaceholder.svelte";
  import Scrollable from "$lib/Scrollable.svelte";

  export let sessionId: string | undefined = undefined;
  export let command: string | undefined = undefined;
  export let output = "";
  let expectingResponse = false;
  let growable: Scrollable | undefined;
  $: awaitingSession = sessionId === undefined;
  $: accessibilityLabel = awaitingSession
    ? "Enter command to run"
    : "Enter input for command";
  $: placeholder = awaitingSession
    ? "Enter command to run (e.g. /bin/bash)"
    : "Enter input for command";

  function resizeTerminalView() {
    growable?.resizeScrollable();
    setTimeout(() => {
      growable?.scrollToBottom();
    }, 100);
  }

  async function sendCommand(newInput: string) {
    try {
      if (sessionId === undefined) {
        // start new command
        let result = await runCommand(newInput);
        command = newInput;
        sessionId = result.id;
        output += result.output.trimStart();
      } else {
        let result = await sendCommandInput(sessionId, newInput + "\n");
        output += result;
      }
      resizeTerminalView();
    } catch (error) {
      snackbarError(error as string);
    }
  }
</script>

<InfoBox title="Terminal Session" fullHeight>
  <div class="terminal-container composite-reveal full-height">
    {#if command}
      <p>Current command: <span class="command">{command}</span></p>
    {:else}
      <EmptyPlaceholder>
        No running process.<br />Get started by entering a command below.
      </EmptyPlaceholder>
    {/if}

    <Scrollable bind:this={growable}>
      <pre>{output}</pre>
    </Scrollable>

    <SendInputForm
      {accessibilityLabel}
      {placeholder}
      sendInput={sendCommand}
      isBusy={expectingResponse}
      onTextInputResize={resizeTerminalView}
    />
  </div>
</InfoBox>

<style>
  .terminal-container {
    height: 100%;
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  p {
    margin: 0;
    padding: 0.5rem 1rem;
    background: var(--color-background);
    border-radius: var(--corner-roundness);
  }

  span.command {
    font-weight: bold;
  }
</style>

```

We create new stories at `src-svelte/src/routes/terminal/TerminalSession.stories.ts`:

```ts
import TerminalSession from "./TerminalSession.svelte";
import type { StoryFn, StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";
import SvelteStoresDecorator from "$lib/__mocks__/stores";
import MockFullPageLayout from "$lib/__mocks__/MockFullPageLayout.svelte";

export default {
  component: TerminalSession,
  title: "Screens/Terminal/Session",
  argTypes: {},
  decorators: [
    SvelteStoresDecorator,
    TauriInvokeDecorator,
    (story: StoryFn) => {
      return {
        Component: MockFullPageLayout,
        slot: story,
      };
    },
  ],
};

const Template = ({ ...args }) => ({
  Component: TerminalSession,
  props: args,
});

export const New: StoryObj = Template.bind({}) as any;
New.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/run_command-bash.yaml",
    "/api/sample-calls/send_command_input-bash-interleaved.yaml",
  ],
};

export const InProgress: StoryObj = Template.bind({}) as any;
InProgress.args = {
  sessionId: "3717ed48-ab52-4654-9f33-de5797af5118",
  command: "bash",
  output:
    // eslint-disable-next-line max-len
    "The default interactive shell is now zsh.\r\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\r\nFor more details, please visit https://support.apple.com/kb/HT208050.\r\nbash-3.2$ ",
};
InProgress.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/send_command_input-bash-interleaved.yaml",
  ],
};

```

When interacting with these stories, we get the error

```
No matching call found for ["send_command_input",{"sessionId":"3717ed48-ab52-4654-9f33-de5797af5118","input":"python api/sample-terminal-sessions/interleaved.py\n"}].
Candidates are ["send_command_input",{"input":"python api/sample-terminal-sessions/interleaved.py\n","sessionId":"3717ed48-ab52-4654-9f33-de5797af5118"}]
```

We see that [this](https://stackoverflow.com/a/16168003) is the way to have `JSON.stringify` be consistent in the order of keys. As such, we modify `src-svelte/src/lib/sample-call-testing.ts`:

```ts
...

function stringify(obj: any): string {
  if (typeof obj === "object") {
    return JSON.stringify(obj, Object.keys(obj).sort());
  }
  return JSON.stringify(obj);
}

export class TauriInvokePlayback {
  ...

  mockCall(
    ...
  ): Promise<Record<string, string>> {
    const jsonArgs = stringify(args);
    const matchingCallIndex = this.unmatchedCalls.findIndex(
      (call) => stringify(call.request) === jsonArgs,
    );
    if (matchingCallIndex === -1) {
      const candidates = this.unmatchedCalls
        .map((call) => stringify(call.request))
        .join("\n");
        ...
    }
    ...
  }
}
```

We add the new screenshots to `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "terminal", "session"],
    variants: ["new", "in-progress"],
    screenshotEntireBody: true,
  },
];
```

Apart from manual testing with Storybook stories, we also create `src-svelte/src/routes/terminal/TerminalSession.test.ts` to properly test the callbacks:

```ts
import { expect, test, vi, type Mock } from "vitest";
import "@testing-library/jest-dom";
import { render, screen, waitFor } from "@testing-library/svelte";
import TerminalSession from "./TerminalSession.svelte";
import userEvent from "@testing-library/user-event";
import { TauriInvokePlayback } from "$lib/sample-call-testing";

describe("Terminal session", () => {
  let tauriInvokeMock: Mock;
  let playback: TauriInvokePlayback;

  beforeEach(() => {
    tauriInvokeMock = vi.fn();
    vi.stubGlobal("__TAURI_INVOKE__", tauriInvokeMock);
    playback = new TauriInvokePlayback();
    tauriInvokeMock.mockImplementation(
      (...args: (string | Record<string, string>)[]) =>
        playback.mockCall(...args),
    );

    window.IntersectionObserver = vi.fn(() => {
      return {
        observe: vi.fn(),
        unobserve: vi.fn(),
        disconnect: vi.fn(),
      };
    }) as unknown as typeof IntersectionObserver;
  });

  afterEach(() => {
    vi.unstubAllGlobals();
  });

  test("Terminal session", async () => {
    render(TerminalSession, {});
    const commandInput = screen.getByLabelText("Enter command to run");
    const sendButton = screen.getByRole("button", { name: "Send" });

    expect(tauriInvokeMock).not.toHaveBeenCalled();
    playback.addSamples("../src-tauri/api/sample-calls/run_command-bash.yaml");

    expect(commandInput).toHaveValue("");
    await userEvent.type(commandInput, "bash");
    await userEvent.click(sendButton);
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    await waitFor(() => {
      expect(
        screen.getByText(
          new RegExp("The default interactive shell is now zsh"),
        ),
      ).toBeInTheDocument();
    });

    playback.addSamples(
      "../src-tauri/api/sample-calls/send_command_input-bash-interleaved.yaml",
    );
    expect(commandInput).toHaveValue("");
    await userEvent.type(
      commandInput,
      "python api/sample-terminal-sessions/interleaved.py",
    );
    await userEvent.click(sendButton);
    expect(tauriInvokeMock).toHaveReturnedTimes(2);
    await waitFor(() => {
      expect(screen.getByText(new RegExp("stderr"))).toBeInTheDocument();
    });
  });
});

```

Next, we refactor out the API calls table into its own component at `src-svelte/src/routes/api-calls/ApiCallsTable.svelte`:

```svelte
<script lang="ts">
  import { getApiCalls, type LightweightLlmCall } from "$lib/bindings";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";
  import Scrollable from "$lib/Scrollable.svelte";
  import ApiCallReference from "$lib/ApiCallReference.svelte";
  import { onMount } from "svelte";
  import EmptyPlaceholder from "$lib/EmptyPlaceholder.svelte";

  const PAGE_SIZE = 50;
  const MIN_MESSAGE_WIDTH = "5rem";
  const MIN_TIME_WIDTH = "12.5rem";

  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
  let llmCalls: LightweightLlmCall[] = [];
  let llmCallsPromise: Promise<void> | undefined = undefined;
  let allCallsLoaded = false;
  let messageWidth = MIN_MESSAGE_WIDTH;
  let timeWidth = MIN_TIME_WIDTH;
  let headerMessageWidth = MIN_MESSAGE_WIDTH;

  const formatter = new Intl.DateTimeFormat(dateTimeLocale, {
    year: "numeric",
    month: "numeric",
    day: "numeric",
    hour: "numeric",
    minute: "numeric",
    hour12: true,
    timeZone,
  });

  export function formatTimestamp(timestamp: string): string {
    const timestampUTC = timestamp + "Z";
    const date = new Date(timestampUTC);
    return formatter.format(date);
  }

  function getWidths(selector: string) {
    const elements = document.querySelectorAll(selector);
    const results = Array.from(elements)
      .map((el) => el.getBoundingClientRect().width)
      .filter((width) => width > 0);
    return results;
  }

  function resizeMessageWidth() {
    messageWidth = MIN_MESSAGE_WIDTH;
    // time width doesn't need a reset because it never decreases

    setTimeout(() => {
      const textWidths = getWidths(".api-calls-page .text-container");
      const timeWidths = getWidths(".api-calls-page .time");
      const minTextWidth = Math.floor(Math.min(...textWidths));
      messageWidth = `${minTextWidth}px`;
      const maxTimeWidth = Math.ceil(Math.max(...timeWidths));
      timeWidth = `${maxTimeWidth}px`;

      headerMessageWidth = messageWidth;
    }, 10);
  }

  function loadApiCalls() {
    if (llmCallsPromise) {
      return;
    }

    if (allCallsLoaded) {
      return;
    }

    llmCallsPromise = getApiCalls(llmCalls.length)
      .then((newCalls) => {
        llmCalls = [...llmCalls, ...newCalls];
        allCallsLoaded = newCalls.length < PAGE_SIZE;
        llmCallsPromise = undefined;

        requestAnimationFrame(resizeMessageWidth);
      })
      .catch((error) => {
        snackbarError(error);
      });
  }

  function asReference(call: LightweightLlmCall) {
    return {
      id: call.id,
      snippet: call.response_message.text.trim(),
    };
  }

  onMount(() => {
    resizeMessageWidth();
    window.addEventListener("resize", resizeMessageWidth);

    return () => {
      window.removeEventListener("resize", resizeMessageWidth);
    };
  });

  $: minimumWidths =
    `--message-width: ${messageWidth}; ` +
    `--header-message-width: ${headerMessageWidth}; ` +
    `--time-width: ${timeWidth}`;
</script>

<div class="container api-calls-page full-height" style={minimumWidths}>
  <div class="message header">
    <div class="text-container">
      <div class="text">Message</div>
    </div>
    <div class="time">Time</div>
  </div>
  <div class="scrollable-container full-height">
    <Scrollable on:bottomReached={loadApiCalls}>
      {#if llmCalls.length > 0}
        {#each llmCalls as call (call.id)}
          <a href={`/api-calls/${call.id}`}>
            <div class="message instance">
              <div class="text-container">
                <div class="text">
                  <ApiCallReference
                    selfContained
                    nolink
                    apiCall={asReference(call)}
                  />
                </div>
              </div>
              <div class="time">{formatTimestamp(call.timestamp)}</div>
            </div>
          </a>
        {/each}
      {:else}
        <div class="message placeholder">
          <div class="text-container">
            <EmptyPlaceholder>
              Looks like you haven't made any calls to an LLM yet.<br />Get
              started via <a href="/chat">chat</a> or by making one
              <a href="/api-calls/new/">from scratch</a>.
            </EmptyPlaceholder>
          </div>
          <div class="time"></div>
        </div>
      {/if}
    </Scrollable>
  </div>
</div>

<style>
  .container {
    gap: 0.25rem;
  }

  .scrollable-container {
    --side-padding: 0.8rem;
    margin: 0 calc(-1 * var(--side-padding));
    width: calc(100% + 2 * var(--side-padding));
    box-sizing: border-box;
  }

  .message {
    display: flex;
    color: black;
  }

  .message.placeholder :global(p) {
    margin-top: 0.5rem;
  }

  .message.header {
    margin-bottom: 0.5rem;
  }

  .message.header .text-container,
  .message.header .time {
    text-align: center;
    font-weight: bold;
  }

  .message .text-container {
    flex: 1;
  }

  .message.instance {
    padding: 0.2rem var(--side-padding);
    border-radius: var(--corner-roundness);
    transition: background 0.5s;
    height: 1.62rem;
    box-sizing: border-box;
  }

  .message.instance:hover {
    background: var(--color-hover);
  }

  .message .text-container .text {
    max-width: var(--message-width);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .message.header .text-container .text {
    max-width: var(--header-message-width);
  }

  .message .time {
    min-width: var(--time-width);
    box-sizing: border-box;
    text-align: right;
  }
</style>

```

and the main database page at `src-svelte/src/routes/api-calls/ApiCalls.svelte` just appears as:

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import IconAdd from "~icons/mingcute/add-fill";
  import ApiCallsTable from "./ApiCallsTable.svelte";

  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
</script>

<InfoBox title="LLM API Calls" fullHeight>
  <div class="container full-height">
    <a class="new-api-call" href="/api-calls/new/" title="New API call">
      <IconAdd />
    </a>
    <ApiCallsTable {dateTimeLocale} {timeZone} />
  </div>
</InfoBox>

<style>
  .container {
    gap: 0.25rem;
  }

  a.new-api-call {
    position: absolute;
    top: 1rem;
    right: 1rem;
  }

  a.new-api-call :global(svg) {
    transform: scale(1.2);
    color: var(--color-faded);
  }
</style>

```

Next, we refactor out the select element from the new API editor page, into a reusable component at `src-svelte/src/lib/controls/Select.svelte`:

```svelte
<script lang="ts">
  import getComponentId from "$lib/label-id";

  export let name: string;
  export let value: string;
  export let label: string;
  let id = getComponentId("select");
</script>

<div class="setting">
  <label for={id}>{label}</label>
  <div class="select-wrapper">
    <select {name} {id} bind:value>
      <slot />
    </select>
  </div>
</div>

<style>
  .setting {
    display: flex;
    flex-direction: row;
    gap: 0.5rem;
  }

  select {
    -webkit-appearance: none;
    -ms-appearance: none;
    -moz-appearance: none;
    appearance: none;
    border: none;
    padding-right: 1rem;
    box-sizing: border-box;
    direction: rtl;
    width: 100%;
  }

  select :global(option) {
    direction: ltr;
  }

  select,
  .select-wrapper {
    background-color: transparent;
    font-family: var(--font-body);
    font-size: 1rem;
  }

  .select-wrapper {
    flex: 1;
    border-bottom: 1px dotted var(--color-faded);
    position: relative;
    display: inline-block;
  }

  .select-wrapper::after {
    content: "▼";
    display: inline-block;
    position: absolute;
    right: 0.25rem;
    top: 0.35rem;
    color: var(--color-faded);
    font-size: 0.5rem;
    pointer-events: none;
  }
</style>

```

and then we edit `src-svelte/src/routes/api-calls/new/ApiCallEditor.svelte` to use this new component, where we update `selectModels` immediately on component load to avoid problems with the `value` being updated but not displaying as such in the HTML render:

```svelte
<script lang="ts">
  ...
  import Select from "$lib/controls/Select.svelte";
  ...

  let selectModels = $provider === "OpenAI" ? OPENAI_MODELS : OLLAMA_MODELS;

</script>

...

  <div class="model-settings">
    <Select name="provider" label="Provider: " bind:value={$provider}>
      <option value="OpenAI">OpenAI</option>
      <option value="Ollama">Ollama</option>
    </Select>
    <Select name="model" label="Model: " bind:value={$llm}>
      {#each selectModels as model}
        <option value={model.apiName}>{model.humanName}</option>
      {/each}
    </Select>
    ...
  </div>
...
```

Now we edit `src-svelte/src/routes/api-calls/ApiCalls.svelte` to use this select, in the process renaming the CSS class `.new-api-call` to `.new-button`:

```svelte
<script lang="ts" context="module">
  import { writable } from "svelte/store";

  type DataTypeEnum = "llm-calls" | "terminal";
  export const dataType = writable<DataTypeEnum>("llm-calls");
</script>

<script lang="ts">
  ...
  import Select from "$lib/controls/Select.svelte";
  import EmptyPlaceholder from "$lib/EmptyPlaceholder.svelte";

  $: infoBoxTitle =
    $dataType === "llm-calls" ? "LLM API Calls" : "Terminal Sessions";
  $: newHref = $dataType === "llm-calls" ? "/api-calls/new/" : "/terminal/new/";
  $: newTitle =
    $dataType === "llm-calls" ? "New API call" : "New Terminal Session";
</script>

<InfoBox title={infoBoxTitle} ...>
  <div class="container full-height">
    <a class="new-button" href={newHref} title={newTitle}>
      <IconAdd />
    </a>
    <Select name="data-type" label="Showing " bind:value={$dataType}>
      <option value="llm-calls">LLM Calls</option>
      <option value="terminal">Terminal</option>
    </Select>
    {#if $dataType === "llm-calls"}
      <ApiCallsTable {dateTimeLocale} {timeZone} />
    {:else}
      <EmptyPlaceholder>
        Terminal sessions cannot be viewed yet.<br />You may
        <a href="/terminal/new/">start</a> a new one.
      </EmptyPlaceholder>
    {/if}
  </div>
</InfoBox>

<style>
  ...

  .container :global(.select-wrapper) {
    flex: 0;
  }

  .container :global(select) {
    width: fit-content;
  }

  a.new-button {
    ...
  }

  a.new-button :global(svg) {
    ...
  }
</style>
```

Note that we have to update the CSS for both the select wrapper and the actual select element to make the select element fit the width of its internal content.

We create the corresponding file at `src-svelte/src/routes/terminal/new/+page.svelte` to avoid a 404:

```svelte
<script lang="ts">
  import TerminalSession from "../TerminalSession.svelte";
</script>

<TerminalSession />

```

We realize upon manual testing that we've never updated, so we edit `src-svelte/src/routes/terminal/TerminalSession.svelte`:

```ts
  async function sendCommand(newInput: string) {
    try {
      expectingResponse = true;
      ...
    } catch (error) {
      ...
    } finally {
      expectingResponse = false;
    }
  }
```

Upon manual testing, we also find out that we cannot navigate away from the terminal session page once we go there. This is because the new path is not shown in the sidebar.

##### Refactoring paths to fix sidebar

We move our files around to fit the new schema and fix the sidebar:

- The folder `src-svelte/src/routes/api-calls/[slug]/` is moved to `src-svelte/src/routes/database/api-calls/[slug]/`
- `src-svelte/src/routes/api-calls/new/` is moved to `src-svelte/src/routes/database/api-calls/new/`
- `src-svelte/src/routes/terminal/` is moved to `src-svelte/src/routes/database/terminal-sessions/`

We rename `src-svelte/src/routes/api-calls/+page.svelte`

We move `src-svelte/src/routes/database/+page.svelte`

Stories such as `src-svelte/src/routes/database/DatabaseView.full-page.stories.ts` are edited:

```ts
import ApiCallsComponent from "./DatabaseView.svelte";
...

export default {
  component: DatabaseView,
  title: "Screens/Database/List",
  ...
};

...
```

The corresponding tests in files such as `src-svelte/src/routes/database/DatabaseView.playwright.test.ts` are edited:

```ts
  const getScrollElement = async () => {
    const url = `http://localhost:6006/?path=/story/screens-database-list--full`;
    ...
  };
```

`src-svelte/src/routes/storybook.test.ts` is updated as well:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "database", "llm-call", "new"],
    ...
  },
  {
    path: ["screens", "database", "llm-call", "import"],
    ...
  },
  {
    path: ["screens", "database", "llm-call"],
    ...
  },
  {
    path: ["screens", "database", "llm-call", "actions"],
    ...
  },
  {
    path: ["screens", "database", "list"],
    ...
  },
  {
    path: ["screens", "database", "terminal-session"],
    ...
  },
  ...
];
```

We move the screenshot folders to correspond:

- `src-svelte/screenshots/baseline/screens/llm-call/list/` to `src-svelte/screenshots/baseline/screens/database/list/`
- `src-svelte/screenshots/baseline/screens/llm-call/import/` to `src-svelte/screenshots/baseline/screens/database/llm-call/import/`
- `src-svelte/screenshots/baseline/screens/llm-call/individual/` to `src-svelte/screenshots/baseline/screens/database/llm-call/individual/`
- `src-svelte/screenshots/baseline/screens/llm-call/new/` to `src-svelte/screenshots/baseline/screens/database/llm-call/new/`
- `src-svelte/screenshots/baseline/screens/terminal/session/` to `src-svelte/screenshots/baseline/screens/database/terminal-session/`

It's initially unclear whether Jest image snapshot will pass on CI if images are missing, especially given that [this issue](https://github.com/americanexpress/jest-image-snapshot/issues/281) initially implied that it would. But from tests, we see that it does fail, and we update the issue with our findings.

However, when run locally with `CI` set to true, this gets mixed in with all the actual screenshot failures. As such, we do this to allow our tests to fail fast:

```ts
...
import { existsSync } from "fs";
...

      test(
        `${testName} should render the same`,
        async ({ expect }) => {
          if (process.env.CI === "true") {
            const goldFilePath = `${SCREENSHOTS_BASE_DIR}/baseline/${storybookPath}/${variantConfig.name}.png`;
            expect(existsSync(goldFilePath), `No baseline found for ${goldFilePath}`).toBeTruthy();
          }
          ...
        },
      );
```

and then we find that we have to move everything in `src-svelte/screenshots/baseline/screens/database/llm-call/individual/` to the `src-svelte/screenshots/baseline/screens/database/llm-call/` folder.

We still have one test failing due to dynamic image comparison. Thus, we do:

```ts
if (process.env.CI === "true" && !variantConfig.assertDynamic) {
  ...
}
```

Now all the quick checks pass.

We edit links in files such as `src-svelte/src/lib/ApiCallReferenceLink.svelte`:

```svelte
  <a href="/database/api-calls/{apiCall.id}">{apiCall.snippet}</a>
```

and imports in files such as `src-svelte/src/lib/__mocks__/stores.ts`:

```ts
import {
  canonicalRef,
  getDefaultApiCall,
  prompt,
} from "../../routes/database/api-calls/new/ApiCallEditor.svelte";
```

The Sidebar itself at `src-svelte/src/routes/SidebarUI.svelte` is edited:

```ts
  const routes: App.Route[] = [
    ...
    {
      name: "Database",
      path: "/database",
      icon: IconDatabase,
    },
    ...
  ];
```

The various tests at `src-svelte/src/routes/SidebarUI.test.ts` have to be edited accordingly too, for example:

```ts
 test("highlights right icon for sub-paths", () => {
    render(SidebarUI, {
      currentRoute: "/database/api-calls/1234",
      ...
    });
    ...
    const apiCallsLink = screen.getByTitle("Database");
    ...
  });
```

When building the whole project, we get the error

```
SvelteKitError: Not found: /api-calls/new/
    at resolve2 (file:///Users/amos/Documents/zamm/src-svelte/.svelte-kit/output/server/index.js:2853:18)
    at resolve (file:///Users/amos/Documents/zamm/src-svelte/.svelte-kit/output/server/index.js:2685:34)
    at #options.hooks.handle (file:///Users/amos/Documents/zamm/src-svelte/.svelte-kit/output/server/index.js:2928:71)
    at respond (file:///Users/amos/Documents/zamm/src-svelte/.svelte-kit/output/server/index.js:2683:43) {
  status: 404,
  text: 'Not Found'
}

node:internal/event_target:1054
  process.nextTick(() => { throw err; });
                           ^
Error: 404 /api-calls/new/
To suppress or handle this error, implement `handleHttpError` in https://kit.svelte.dev/docs/configuration#prerender
    at file:///Users/amos/Documents/zamm/node_modules/@sveltejs/kit/src/core/config/options.js:202:13
```

This turns out to be because we haven't updated `src-svelte/svelte.config.js`:

```js
const config = {
  ...

  kit: {
    ...,
    prerender: {
      crawl: false,
      entries: [
        "*",
        "/database/api-calls/new/",
        "/database/api-calls/[slug]",
        "/database/terminal-sessions/new/",
      ],
    },
  },
};
```

The one failing test is at `src-svelte/src/routes/SidebarUI.test.ts`. It is unknown why it was passing before and failing now, but it passes if we run just the failing test of "plays whoosh sound with right speed and volume." We therefore edit the tests to reset the mock API invocations before each test, merging the `beforeAll` into the `beforeEach`, and only adding the mock API call in the tests where it's actually expected to be called:

```ts
describe("Sidebar interactions", () => {
  ...
  const whooshSample = "../src-tauri/api/sample-calls/play_sound-whoosh.yaml";

  beforeEach(() => {
    tauriInvokeMock = vi.fn();
    vi.stubGlobal("__TAURI_INVOKE__", tauriInvokeMock);
    playback = new TauriInvokePlayback();
    tauriInvokeMock.mockImplementation(
      ...
    );

    render(SidebarUI, {
      ...
    });
    ...
  });

  test("can change page path", async () => {
    playback.addSamples(whooshSample);
    ...
  });

  test("plays whoosh sound with right speed and volume", async () => {
    playback.addSamples(whooshSample);
    ...
  });
});
```

Finally, we notice that on the actual app, the title element does not update when we update the selected combo box value. We create `src-svelte/src/routes/database/DatabaseView.test.ts` to try to reproduce this error in a test environment:

```ts
import { expect, test, vi } from "vitest";
import "@testing-library/jest-dom";
import DatabaseView, { resetDataType } from "./DatabaseView.svelte";
import { render, screen, waitFor } from "@testing-library/svelte";
import userEvent from "@testing-library/user-event";

describe("Database View", () => {
  beforeEach(() => {
    window.IntersectionObserver = vi.fn(() => {
      return {
        observe: vi.fn(),
        unobserve: vi.fn(),
        disconnect: vi.fn(),
      };
    }) as unknown as typeof IntersectionObserver;
  });

  afterEach(() => {
    resetDataType();
  });

  test("renders LLM calls by default", async () => {
    render(DatabaseView, {
      dateTimeLocale: "en-GB",
      timeZone: "Asia/Phnom_Penh",
    });

    const title = screen.getByRole("heading");
    expect(title).toHaveTextContent("LLM API Calls");
    userEvent.selectOptions(
      screen.getByRole("combobox", { name: "Showing" }),
      "Terminal Sessions",
    );
    await waitFor(() => {
      expect(title).toHaveTextContent("Terminal Sessions");
    });
  });
});
```

wherein we also edit `src-svelte/src/routes/database/DatabaseView.svelte` to make it amenable to testing, as well as fixing the label for the dropdown:

```svelte
<script lang="ts" context="module">
  ...

  export function resetDataType() {
    dataType.set("llm-calls");
  }
</script>

...
    <Select name="data-type" ...>
      ...
      <option value="terminal">Terminal Sessions</option>
    </Select>
...
```

Unfortunately, it turns out this test does not fail. Upon further inspection, even in Storybook, the bug is not reproducible except in the "Full Page" view. The bug goes away if we get rid of the `in:revealTitle|global={timing.title}` transition effect for the `h2` element in `InfoBox`, but right now we don't care to create a minimally reproducible version of this bug.

We refactor `src-svelte/src/routes/database/DatabaseView.playwright.test.ts` to add an extra test, which does finally end up failing. In the process, we refactor out the `getFrame` part of `getScrollElement`, and makes the functions take in the actual Storybook URL:

```ts
describe("Database View", () => {
  ...

  const getFrame = async (url: string) => {
    ...
    return maybeFrame;
  };

  const getScrollElement = async (url: string) => {
    const frame = await getFrame(url);
    ...
  };
  
  ...

  test(
    "loads more messages when scrolled to end",
    async () => {
      const { apiCallsScrollElement } = await getScrollElement(
        `http://localhost:6006/?path=/story/screens-database-list--full`,
      );
      ...
    },
    ...
  );

  test(
    "updates title when changing dropdown",
    async () => {
      const frame = await getFrame(
        `http://localhost:6006/?path=/story/screens-database-list--full-page`,
      );
      await expect(frame.locator("h2")).toHaveText("LLM API Calls");

      await frame.locator("select").selectOption("Terminal Sessions");
      await expect(frame.locator("h2")).toHaveText("Terminal Sessions");
    },
    { retry: 2, timeout: PLAYWRIGHT_TEST_TIMEOUT },
  );
});
```

We edit `src-svelte/src/lib/InfoBox.svelte` to apply the actual fix:

```ts
  ...

  function forceUpdateTitleText(newTitle: string) {
    if (titleElement) {
      titleElement.textContent = newTitle;
    }
  }

  ...
  $: forceUpdateTitleText(title);
```

##### End-to-end testing

We edit `webdriver/test/specs/e2e.test.js` to add e2e screenshots of the new pages, as well as the API import page that wasn't tested before. This gives us confidence that everything still works as expected despite the major reshuffling of pages and paths.

It turns out we need to add some extra functions to interact with the select and the input (the input text function we already documented before in the e2e test setup, and the select function also needs [the `change` event to fire](https://stackoverflow.com/a/28324400)). We also change all the `await findAndClick('a[title="API Calls"]');` to `await findAndClick('a[title="Database"]');` to reflect the updated name of the database sidebar icon link. We also need to click twice on the database link to reset what the sidebar link points to from previous tests, and then still navigate away and back to the same page to reset the animation effects:

```js
async function findAndSelect(selector, index, timeout) {
  const select = await $(selector);
  await select.waitForClickable({
    timeout: timeout ?? DEFAULT_TIMEOUT,
  });
  await browser.execute(`arguments[0].selectedIndex = ${index}`, select);
  await browser.execute(
    `arguments[0].dispatchEvent(new Event('change'))`,
    select,
  );
}

async function findAndInput(selector, value) {
  const field = await $(selector);
  await browser.execute(`arguments[0].value="${value}"`, field);
  await browser.execute(
    'arguments[0].dispatchEvent(new Event("input", { bubbles: true }))',
    field,
  );
}

describe("App", function () {
  ...

  it("should allow navigation to the API calls page", async function () {
      this.retries(2);
      await findAndClick('a[title="Database"]');
      ...
      await findAndClick('a[title="Database"]');
      ...
  }

  ...

  it("should allow navigation to the API call import page", async function () {
    this.retries(2);
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Database"]');
    await findAndClick("a=from scratch");
    await findAndClick("a=import one");
    await findAndClick('a[title="Dashboard"]');
    await findAndClick('a[title="Database"]');
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("import-api-call", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  it("should allow navigation to the terminal sessions page", async function () {
    this.retries(2);
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Database"]');
    await findAndSelect('select[name="data-type"]', 1);
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("terminal-sessions", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  it("should allow navigation to the new terminal session page", async function () {
    this.retries(2);
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Database"]');
    await findAndSelect('select[name="data-type"]', 1);
    await findAndClick("a=start");
    await findAndClick('a[title="Dashboard"]');
    await findAndClick('a[title="Database"]');
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("new-terminal-session", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  it("should successfully interact with the terminal", async function () {
    this.retries(2);
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Database"]');
    await findAndSelect('select[name="data-type"]', 1);
    await findAndClick("a=start");
    await findAndClick('a[title="Dashboard"]');
    await findAndClick('a[title="Database"]');

    await findAndInput('textarea[name="message"]', "/bin/bash");
    await findAndClick('button[type="submit"]');
    await findAndInput('textarea[name="message"]', "pwd");
    await findAndClick('button[type="submit"]');

    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("running-terminal-session", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  it("should allow navigation to the settings page", async function () {
    ...
    // click twice to reset the saved navigation to the "New API Call" page
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Dashboard"]');
    await findAndClick('a[title="Database"]');
    ...
  });

  ...
});
```

On CI, however, we find that the machine name always changes, so that the prompt looks more like this:

```bash
runner@fv-az700-454:~/work/zamm/zamm/webdriver$ pwd
/home/runner/work/zamm/zamm/webdriver
```

and the `fv-az700-454` part always changes.

So we try to follow [this answer](https://stackoverflow.com/a/8688138) and use a different bash prompt. We edit `webdriver/test/specs/e2e.test.js`, but find that we have to change the input function to escape quotes properly:

```js
async function findAndInput(selector, value) {
  ...
  const escapedValue = value.replace(/\\/g, "\\\\").replace(/"/g, '\\"');
  await browser.execute(`arguments[0].value="${escapedValue}"`, field);
  ...
}
```

Unfortunately it still fails because it cannot find the "file". Perhaps that command works only in a shell session, not when invoking a local program directly. As such, we have the test simply load a local rc file:

```js
  it("should successfully interact with the terminal", async function () {
    ...
    await findAndInput(
      'textarea[name="message"]',
      "bash --rcfile ./zamm.bashrc",
    );
    ...
  });
```

where `webdriver/zamm.bashrc` looks like

```bashrc
export PS1="zamm-e2e-testing> "
```

Finally, one more failing test doesn't result in a screenshot at all:

```
[wry 0.24.7 linux #0-0] 1) App should be able to view single LLM call
[wry 0.24.7 linux #0-0] element (".api-calls-page a:nth-child(2)") still not clickable after 5000ms
[wry 0.24.7 linux #0-0] Error: element (".api-calls-page a:nth-child(2)") still not clickable after 5000ms
[wry 0.24.7 linux #0-0]     at async findAndClick (e2e.test.js:12:3)
[wry 0.24.7 linux #0-0]     at async Context.<anonymous> (e2e.test.js:201:5)
```

This is because we'll have navigated away from the database view page. We edit `webdriver/test/specs/e2e.test.js` again:

```js
  it("should be able to view single LLM call", async function () {
    this.retries(2);
    // make sure we're back on the LLM APIs database view
    await findAndClick('a[title="Database"]');
    await findAndClick('a[title="Database"]');
    await findAndSelect('select[name="data-type"]', 0);
    ...
  });
```

##### Spacing

We do a little bit of final editing for the spacing. We edit `src-svelte/src/routes/database/DatabaseView.svelte` to add just a bit of spacing to the select element:

```css
  .container :global(.select-wrapper) {
    ...
    margin-bottom: 0.25rem;
  }
```

We remove the bottom margin styling for `.message.header` in `src-svelte/src/routes/database/api-calls/ApiCallsTable.svelte` because there's already half a rem of spacing added by the flex gap for the table.

##### Windows compatibility

We realize that inputs to the `cmd` process on Windows are not being evaluated. We reproduce this in `src-tauri/src/test_helpers/terminal.rs`, and find out that `cmd` expects `\r\n` for inputs on Windows before they get evaluated:

```rs
    #[tokio::test]
    async fn test_windows_interactivity() {
        let mut terminal =
            TestTerminal::new("api/sample-terminal-sessions/windows.cast");
        terminal.run_command("cmd").await.unwrap();
        terminal.send_input("dir\r\n").await.unwrap();
        let output = terminal.send_input("echo %cd%\r\n").await.unwrap();
        assert!(output.contains("src-tauri"));
    }
```

The recording at `src-tauri/api/sample-terminal-sessions/windows.cast` shows us the correct output:

```asciicast
{"version":2,"width":80,"height":24,"timestamp":1729144933,"command":"cmd"}
[0.208,"o","\u001b[?25l\u001b[2J\u001b[m\u001b[HMicrosoft Windows [Version 10.0.22631.4169]\r\n(c) Microsoft Corporation. All rights reserved.\u001b[4;1HC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>\u001b]0;C:\\WINDOWS\\system32\\cmd.EXE\u0007\u001b[?25h"]
[0.208,"i","dir\r\n"]
[0.41,"o","\u001b[?25ldir\r\n Volume in drive C is Windows\r\n Volume Serial Number is 30A7-E02E\u001b[8;1H Directory of C:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri\u001b[10;1H25/09/2024  05:54 pm    <DIR>          .\r\n20/09/2024  07:02 pm    <DIR>          ..\r\n14/02/2024  08:37 pm                73 .gitignore\r\n14/02/2024  08:37 pm                52 .rustfmt.toml\r\n17/10/2024  12:47 pm    <DIR>          api\r\n14/02/2024  08:37 pm    <DIR>          binaries\r\n14/02/2024  08:37 pm                90 build.rs\r\n25/09/2024  05:54 pm           142,536 Cargo.lock\r\n17/10/2024  12:58 pm             2,102 Cargo.toml\r\n20/09/2024  07:02 pm               830 clippy.py\r\n14/02/2024  08:37 pm               244 diesel.toml\r\n20/09/2024  07:02 pm    <DIR>          icons\r\n14/05/2024  03:37 pm               495 Makefile\r\n20/09/2024  07:02 pm    <DIR>          migrations\r\n14/02/2024  08:37 pm    <DIR>          sounds\r\u001b]0;C:\\WINDOWS\\system32\\cmd.EXE - dir\u0007\u001b[?25h\n17/10/2024  12:47 pm    <DIR>          src\r\n25/06/2024  08:14 pm    <DIR>          target\r\n20/09/2024  07:02 pm             1,645 tauri.conf.json\r\n               9 File(s)        148,067 bytes\r\n               9 Dir(s)  190,782,652,416 bytes free\r\n\u001b]0;C:\\WINDOWS\\system32\\cmd.EXE\u0007\nC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>"]
[0.41,"i","echo %cd%\r\n"]
[0.611,"o","echo %cd%\r\nC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri\r\u001b]0;C:\\WINDOWS\\system32\\cmd.EXE - echo  C:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri\u0007\n\u001b]0;C:\\WINDOWS\\system32\\cmd.EXE\u0007\nC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>"]
```

We keep seeing lines such as `\u001b]0;C:\\WINDOWS\\system32\\cmd.EXE\u0007` because [`cmd` sets the title](https://stackoverflow.com/a/75400944) of the window to the command being executed, and that is apparently the escape code for setting pty window titles.

We fix this by editing `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte`:

```ts
  ...
  import { systemInfo } from "$lib/system-info";
  ...

  async function sendCommand(newInput: string) {
    ...
      else {
        ...
        const inputNewline = $systemInfo?.os === "Windows" ? "\r\n" : "\n";
        let result = await sendCommandInput(sessionId, newInput + inputNewline);
        ...
      }
    ...
  }
```

##### Failing Svelte tests

###### Svelte component exported function

Apart from the usual necessary Storybook screenshot updates, we also get

```
⎯⎯⎯⎯⎯ Uncaught Exception ⎯⎯⎯⎯⎯
TypeError: scrollable.getDimensions is not a function
 ❯ src/lib/Scrollable.svelte:32:48
     30|       const newHeight = Math.floor(container.getBoundingClientRect().h…
     31|       scrollableHeight = `${newHeight}px`;
     32|       dispatchResizeEvent("resize", scrollable.getDimensions());
       |                                                ^
     33|     });
     34|   }
 ❯ invokeTheCallbackFunction ../node_modules/jsdom/lib/jsdom/living/generated/Function.js:19:26
 ❯ runAnimationFrameCallbacks ../node_modules/jsdom/lib/jsdom/browser/Window.js:636:13
 ❯ Timeout.<anonymous> ../node_modules/jsdom/lib/jsdom/browser/Window.js:614:11
 ❯ listOnTimeout node:internal/timers:573:17
 ❯ processTimers node:internal/timers:514:7

This error originated in "src/routes/database/terminal-sessions/TerminalSession.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
```

This appears to be the same problem as the `growable.scrollToBottom` problem in [API calls display](/zamm-notes/api-calls-display.md). There, we solved it by simply including a guard for it, because it only ever happens in CI. We do the same here by editing `src-svelte/src/lib/Scrollable.svelte`:

```ts
  export function resizeScrollable() {
    ...

    requestAnimationFrame(() => {
      ...
      if (scrollable?.getDimensions) {
        // CI environment guard
        dispatchResizeEvent("resize", scrollable.getDimensions());
      } else {
        console.warn("scrollable.getDimensions() is not a function");
      }
    });
  }
```

###### Info box update

There's also

```
 FAIL  src/routes/database/DatabaseView.playwright.test.ts > Database View > updates title when changing dropdown
Error: Timed out 5000ms waiting for expect(locator).toHaveText(expected)

Locator: locator('h2')
Expected string: "LLM API Calls"
Received string: "LLM API Cal"
Call log:
  - locator._expect with timeout 5000ms
  - waiting for locator('h2')
  -   locator resolved to <h2 id="infobox-205706" class="s-BIEZu5ODCI1i">LLM API Cal</h2>
  -   unexpected value "LLM API Cal"
```

which surely arises from issues with the end animation not finishing yet. On our own Linux test box, we get instead

```
 FAIL  src/routes/database/DatabaseView.playwright.test.ts > Database View > updates title when changing dropdown
Error: Timed out 5000ms waiting for expect(locator).toHaveText(expected)

Locator: locator('h2')
Expected string: "Terminal Sessions"
Received string: "LLM API Calls"
Call log:
  - locator._expect with timeout 5000ms
  - waiting for locator('h2')
  -   locator resolved to <h2 id="infobox-728103" class="s-BIEZu5ODCI1i">LLM API Calls</h2>
  -   unexpected value "LLM API Calls"
  -   locator resolved to <h2 id="infobox-728103" class="s-BIEZu5ODCI1i">LLM API Calls</h2>
```

On the Mac, we can't reproduce this manually in Safari, but we can in Firefox. It turns out that this is because toggling the dropdown too quickly causes the `h2` element to update before the transition is complete. This doesn't exactly explain why the presence of the transition effect affects the lack of title update in regular circumstances when the transition has long finished, but since the infrastructure for that fix ties in neatly with this one, we edit it a bit further. We edit `src-svelte/src/lib/InfoBox.svelte`:

```ts
  class TypewriterEffect extends SubAnimation<void> {
    constructor(...) {
      const originalText = anim.node.textContent ?? "";
      super({
        ...,
        tick: (tLocalFraction: number) => {
          let currentText = anim.node.getAttribute("data-text") ?? originalText;
          let length = currentText.length + 1;
          const i = ...;
          anim.node.textContent = i === 0 ? "" : currentText.slice(0, i - 1);
        },
      });
    }
  }

  ...

  function forceUpdateTitleText(newTitle: string) {
    if (titleElement) {
      titleElement.textContent = ...;
      titleElement.setAttribute("data-text", newTitle);
    }
  }
```

The `LLM API CALL` failure on CI is likely because the timeout is too low. We edit `src-svelte/src/routes/database/DatabaseView.playwright.test.ts`:

```ts
  test(
    "updates title when changing dropdown",
    async () => {
      ...
      await expect(frame.locator("h2")).toHaveText("LLM API Calls", {
        timeout: PLAYWRIGHT_TIMEOUT,
      });
      ...
      await expect(frame.locator("h2")).toHaveText("Terminal Sessions", {
        timeout: PLAYWRIGHT_TIMEOUT,
      });
    },
    ...
  );
```

### Stripping out ANSI escape codes

We want to try stripping out the ANSI escape codes from the Windows terminal output. We find out that there's a crate that is specifically designed for this, so we do `cargo add` to get the latest version and automatically update `src-tauri/Cargo.toml`:

```toml
strip-ansi-escapes = "0.2.0"
```

We create `src-tauri/api/sample-calls/run_command-cmd.yaml`:

```yaml
request:
  - run_command
  - >
    {
      "command": "cmd"
    }
response:
  message: >
    {
      "id": "319cc7fd-58cc-4320-ab46-2f0ba11c5402",
      "timestamp": "2024-10-17T06:02:13",
      "output": "Microsoft Windows [Version 10.0.22631.4169]\n(c) Microsoft Corporation. All rights reserved.C:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>"
    }
sideEffects:
  database:
    endStateDump: command-run-cmd
  terminal:
    recordingFile: windows.cast

```

We edit `src-tauri/src/commands/terminal/run.rs` to actuall run a test with this file, and to get the test to pass on other machines (but we limit it to Windows anyways in anticipation of escape codes possibly being different across different platforms):

```rs
...

fn clean_output(output: &str) -> String {
    strip_ansi_escapes::strip_str(output)
}

...

async fn run_command_helper(
  ...
) -> ZammResult<RunCommandResponse> {
  ...

  let raw_output = terminal.run_command(command)?;
  ...

  let output = clean_output(&raw_output);

  ...
}

...

#[cfg(test)]
mod tests {
  ...

  impl SampleCallTestCase<RunCommandRequest, ZammResult<RunCommandResponse>>
        for RunCommandTestCase
  {
      ...

      fn output_replacements(
            ...
        ) -> HashMap<String, String> {
          ...
          let asciicast_filename = &sample
                .side_effects
                .as_ref()
                .unwrap()
                .terminal
                .as_ref()
                .unwrap()
                .recording_file;
            let expected_os = if asciicast_filename.ends_with("bash.cast") {
                "Mac"
            } else if asciicast_filename.ends_with("windows.cast") {
                "Windows"
            } else {
                "Linux"
            };
            ...
        }
    }

    ...

    #[cfg(target_os = "windows")]
    check_sample!(
        RunCommandTestCase,
        test_start_cmd,
        "./api/sample-calls/run_command-cmd.yaml"
    );
}
```

After we run the test, the `src-tauri/api/sample-database-writes/command-run-cmd/dump.yaml` (and the corresponding dumped SQL) looks like:

```yaml
terminal_sessions:
- id: 319cc7fd-58cc-4320-ab46-2f0ba11c5402
  timestamp: 2024-10-17T06:02:13
  command: cmd
  os: Windows
  cast:
    header:
      version: 2
      width: 80
      height: 24
      timestamp: 1729144933
      command: cmd
    entries:
    - - 0.208
      - 'o'
      - "\e[?25l\e[2J\e[m\e[HMicrosoft Windows [Version 10.0.22631.4169]\r\n(c) Microsoft Corporation. All rights reserved.\e[4;1HC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>\e]0;C:\\WINDOWS\\system32\\cmd.EXE\a\e[?25h"

```

Note that the dump correctly marks the original running OS as Windows, even if the tests will ignore that fact.

Unfortunately, when we go to actually test this, we find that there's no newlines between the copyright notification and the text. We see

```
Microsoft Windows [Version 10.0.22631.4169]
(c) Microsoft Corporation. All rights reserved.C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-tauri>
```

instead of what we actually see when we open up a new prompt:

```
Microsoft Windows [Version 10.0.22631.4169]
(c) Microsoft Corporation. All rights reserved.

C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-tauri>
```

We add a story to `src-svelte/src/routes/database/terminal-sessions/TerminalSession.stories.ts` that makes it easy to replicate this problem:

```ts
export const NewOnWindows: StoryObj = Template.bind({}) as any;
NewOnWindows.parameters = {
  sampleCallFiles: ["/api/sample-calls/run_command-cmd.yaml"],
};

```

Also, our other Rust tests now fail as well because the `strip_ansi_escapes::strip_str` function also strips out `\r`'s. This is probably desired behavior, however, so we also modify the tests. We update the terminal state in `src-svelte/src/routes/database/terminal-sessions/TerminalSession.stories.ts` to match, and remove the Windows-only test guard for now.

While testing things in Storybook, we realize that we can now enter *any* command and the Svelte terminal component in Storybook will still play it. After some debugging, we realize that the problem is in our `stringify` function, because the array is being detected as an "object" by Javascript, but then our stringified dictionary has no keys because the array has no keys. we come across [this answer](https://stackoverflow.com/a/51285298). We edit `src-svelte/src/lib/sample-call-testing.ts` to fix this, and to also make stringification less redundant by only stringifying once:

```ts
...
import { ..., isArray } from "lodash";

...

function stringify(obj: any): string {
  if (isArray(obj)) {
    const items = obj.map((item) => stringify(item));
    return `[${items.join(",")}]`;
  }
  if (obj.constructor === Object) {
    ...
  }
  ...
}

export class TauriInvokePlayback {
  ...

  mockCall(
    ...
  ): Promise<Record<string, string>> {
    ...
    const stringifiedUnmatchedCalls = this.unmatchedCalls.map((call) =>
      stringify(call.request),
    );
    const matchingCallIndex = stringifiedUnmatchedCalls.findIndex(
      (call) => call === jsonArgs,
    );
    if (matchingCallIndex === -1) {
      const candidates = stringifiedUnmatchedCalls.join("\n");
      ...
    }
    ...
  }
}
```

It appears that our test code has gotten complicated enough to warrant testing of their own, because if we can't be sure that our tests are testing what we think they are, then the whole chain of confidence is broken. We create `src-svelte/src/lib/sample-call-testing.test.ts`, noting that we [have to wrap](https://stackoverflow.com/a/76720638) the failing function call inside a function in order for Vitest to correctly catch the expected error:

```ts
import { TauriInvokePlayback } from "./sample-call-testing";

describe("TauriInvokePlayback", () => {
  it("should mock matched calls", async () => {
    const playback = new TauriInvokePlayback();
    playback.addCalls({
      request: ["command", { inputArg: "input" }],
      response: { outputKey: "output" },
      succeeded: true,
    });
    const result = await playback.mockCall("command", { inputArg: "input" });
    expect(result).toEqual({ outputKey: "output" });
  });

  it("should throw an error for unmatched calls", async () => {
    const playback = new TauriInvokePlayback();
    playback.addCalls({
      request: ["command", { inputArg: "input" }],
      response: { outputKey: "output" },
      succeeded: true,
    });
    expect(() => playback.mockCall("command", { inputArg: "wrong" })).toThrow(
      'No matching call found for ["command",{"inputArg":"wrong"}].\n' +
        'Candidates are ["command",{"inputArg":"input"}]',
    );
  });
});
```

Next, we decide to move the bulk of the I/O logic to the backend, leaving the frontend only concerned with handling user input and displaying completely cleaned output. We edit `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte` to stop trimming the start of output strings:

```ts
  async function sendCommand(newInput: string) {
    try {
      ...
      if (sessionId === undefined) {
        ...
        output += result.output;
      } ...
    }
    ...
  }
```

add that functionality to `src-tauri/src/commands/terminal/run.rs`:

```rs
fn clean_output(output: &str) -> String {
    strip_ansi_escapes::strip_str(output)
        .trim_start()
        .to_string()
}
```

and update `src-tauri/api/sample-calls/run_command-bash.yaml` to remove the leading newline in the expected output.

Next, we do the same for handling trailing newlines on command input. We edit `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte` to remove that OS-specific logic and directly send the user's own input:

```ts
  async function sendCommand(newInput: string) {
    try {
      ...
      if (sessionId === undefined) {
        ...
      } else {
        let result = await sendCommandInput(sessionId, newInput);
        ...
      }
      ...
    }
    ...
  }
```

We edit `src-tauri/src/commands/terminal/send_input.rs` instead to handle the trailing newline, and to mark the bash text as a non-Windows test because otherwise, on Windows the terminal input sent will be different than what is recorded on the asciicast:

```rs
async fn send_command_input_helper(
    ...
) -> ZammResult<String> {
    ...
    #[cfg(target_os = "windows")]
    let input_with_newline = format!("{}\r\n", input);
    #[cfg(not(target_os = "windows"))]
    let input_with_newline = format!("{}\n", input);
    let result = terminal.send_input(&input_with_newline)?;
    ...
}

...

#[cfg(test)]
mod tests {
    ...

    #[cfg(not(target_os = "windows"))]
    check_sample!(
        SendInputTestCase,
        test_bash_interleaved,
        "./api/sample-calls/send_command_input-bash-interleaved.yaml"
    );
}
```

We update `src-tauri/api/sample-calls/send_command_input-bash-interleaved.yaml` to remove the trailing newline from the expected input.

We go to [v0.0.5 of ZAMM](https://github.com/zamm-dev/zamm/blob/v0.0.5/zamm/utils.py#L62-L122) (commit `e7e589f`) to find out how we've done this before. We revisit the link to [this resource](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html) to see that

> Set Position: `\u001b[{n};{m}H` moves cursor to row `n` column `m`

Hence, the sequence `\u001b[4;1H` in our Windows terminal output means "move cursor to row 4, column 1" (where these rows and columns are indexed starting at 1). Because there's only been two rows so far, moving to row 4 is equivalent to outputting `\n\n`.

As such, we remove the `strip-ansi-escapes` package from `src-tauri/Cargo.toml` and edit `src-tauri/src/commands/terminal/run.rs` to define state-machine-like logic. We use [this answer](https://stackoverflow.com/a/23810557) to define the constants, and [this](https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797)​ and [this resource](https://notes.burke.libbey.me/ansi-escape-codes/) on ANSI escape codes:

```rs
...

#[derive(PartialEq)]
enum EscapeSequence {
    None,
    Start,
    InEscape,
    InOperatingSystemEscape,
    LineStart,
}

static ESCAPE_COMMANDS: &[char] = &['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'b', 'm'];

...

fn clean_output(output: &str) -> String {
    let mut cleaned_lines = Vec::<String>::new();
    output.split('\n').for_each(|line| {
        let mut escape: EscapeSequence = EscapeSequence::None;
        let mut cleaned_line = String::new();
        let mut current_escape_arg = String::new();
        let mut escape_args = Vec::<String>::new();
        let mut escape_command = ' ';
        line.chars().for_each(|c| {
            if c == '\u{001B}' {
                escape = EscapeSequence::Start;
            } else if escape == EscapeSequence::Start {
                if c == '[' || c == '(' {
                    escape = EscapeSequence::InEscape;
                } else if c == ']' {
                    escape = EscapeSequence::InOperatingSystemEscape;
                } else {
                    escape = EscapeSequence::None;
                    cleaned_line.push(c);
                }
            } else if escape == EscapeSequence::InEscape
                || escape == EscapeSequence::InOperatingSystemEscape
            {
                if c == '?' {
                    // it's just a private sequence marker, do nothing
                } else if escape == EscapeSequence::InEscape
                    && ESCAPE_COMMANDS.contains(&c)
                {
                    escape_command = c;
                    escape_args.push(current_escape_arg.clone());
                    current_escape_arg.clear();
                    escape = EscapeSequence::None;
                } else if c == ';' {
                    escape_args.push(current_escape_arg.clone());
                    current_escape_arg.clear();
                } else if c == '\u{0007}' {
                    if let Some(last_char) = current_escape_arg.pop() {
                        escape_command = last_char;
                    }
                    escape_args.push(current_escape_arg.clone());
                    current_escape_arg.clear();
                    escape = EscapeSequence::None;
                } else {
                    current_escape_arg.push(c);
                }

                if escape == EscapeSequence::None {
                    escape_args.push(current_escape_arg.clone());
                    current_escape_arg.clear();

                    // now we actually handle the escape sequence
                    if escape_command == 'H' {
                        if let Some(first_arg) = escape_args.first() {
                            if let Ok(row) = first_arg.parse::<usize>() {
                                if row > cleaned_lines.len() {
                                    cleaned_lines.push(cleaned_line.clone());
                                    cleaned_line.clear();
                                    // -1 because the new cleaned_line will be added at
                                    // the end as the next line
                                    cleaned_lines.resize(row - 1, "".to_string());
                                }
                            }
                        }
                    }
                }
            } else if c == '\r' {
                escape = EscapeSequence::LineStart;
            } else if escape == EscapeSequence::LineStart {
                escape = EscapeSequence::None;
                cleaned_line.clear();
                cleaned_line.push(c);
            } else {
                cleaned_line.push(c);
            }
        });
        cleaned_lines.push(cleaned_line);
    });
    cleaned_lines.join("\n").trim_start().to_string()
}

...
```

And now, `src-tauri/api/sample-calls/run_command-cmd.yaml` can like this:

```yaml
...
response:
  message: >
    {
      ...
      "output": "Microsoft Windows [Version 10.0.22631.4169]\n(c) Microsoft Corporation. All rights reserved.\n\nC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>"
    }
```

and we verify in Storybook that the terminal session now finally correctly displays the output.

#### Refactoring

We do a little bit of refactoring, and move the parsing code to `src-tauri/src/commands/terminal/parse.rs`:

```rs
#[derive(PartialEq)]
enum EscapeSequence {
    ...
}

static ESCAPE_COMMANDS: &[char] = &[...];

pub fn clean_output(output: &str) -> String {
    ...
}
```

We then edit `src-tauri/src/commands/terminal/mod.rs`:

```rs
mod parse;
```

and import it in `src-tauri/src/commands/terminal/run.rs`:

```rs
use crate::commands::terminal::parse::clean_output;
```

We realize that it's rather unnecessary to split the strings up by newline as a completely different operation from the regular flow, so we edit `src-tauri/src/commands/terminal/parse.rs`:

```rs
pub fn clean_output(output: &str) -> String {
    let mut cleaned_lines = Vec::<String>::new();
    let mut escape: EscapeSequence = EscapeSequence::None;
    let mut cleaned_line = String::new();
    let mut current_escape_arg = String::new();
    let mut escape_args = Vec::<String>::new();
    let mut escape_command = ' ';

    output.chars().for_each(|c| {
        if c == '\u{001B}' {
            ...
        } else if escape == EscapeSequence::Start {
            ...
        } else if escape == EscapeSequence::InEscape
            || escape == EscapeSequence::InOperatingSystemEscape
        {
            ...

            if escape == EscapeSequence::None {
                escape_args.push(current_escape_arg.clone());
                current_escape_arg.clear();

                // now we actually handle the escape sequence
                if escape_command == 'H' {
                    ...
                }

                escape_args.clear();
                escape_command = ' ';
            }
        } else if c == '\r' {
            ...
        } else if escape == EscapeSequence::LineStart {
            if c == '\n' {
                escape = EscapeSequence::None;
                cleaned_lines.push(cleaned_line.clone());
                cleaned_line.clear();
            } else {
                escape = EscapeSequence::None;
                cleaned_line.clear();
                cleaned_line.push(c);
            }
        } else if c == '\n' {
            escape = EscapeSequence::None;
            cleaned_lines.push(cleaned_line.clone());
            cleaned_line.clear();
        } else {
            cleaned_line.push(c);
        }
    });
    cleaned_lines.push(cleaned_line);

    cleaned_lines.join("\n").trim_start().to_string()
}

```

We edit `src-tauri/src/commands/terminal/parse.rs` to further refactor the variables into their own struct, and to make the `EscapeSequence` public so that it's congruent with the struct fields being public (although we could make it package private instead), and then prepend `parser.` in front of every local variable:

```rs
pub enum EscapeSequence {
    ...
}

...

pub struct OutputParser {
    pub cleaned_lines: Vec<String>,
    pub cleaned_line: String,
    pub escape: EscapeSequence,
    pub current_escape_arg: String,
    pub escape_args: Vec<String>,
    pub escape_command: char,
}

impl Default for OutputParser {
    fn default() -> Self {
        OutputParser {
            cleaned_lines: Vec::<String>::new(),
            cleaned_line: String::new(),
            escape: EscapeSequence::None,
            current_escape_arg: String::new(),
            escape_args: Vec::<String>::new(),
            escape_command: ' ',
        }
    }
}

pub fn clean_output(output: &str) -> String {
    let mut parser = OutputParser::default();

    output.chars().for_each(|c| {
        if c == '\u{001B}' {
            parser.escape = EscapeSequence::Start;
        }
        ...
    });
    ...
}
```

Next, we continue our refactor by defining a function to handle commands, and remove the `escape_command` variable because it's only ever needed right before invoking that function:

```rs
impl OutputParser {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn handle_escape_command(&mut self, escape_command: char) {
        if escape_command == 'H' {
            if let Some(first_arg) = self.escape_args.first() {
                if let Ok(row) = first_arg.parse::<usize>() {
                    if row > self.cleaned_lines.len() {
                        self.cleaned_lines.push(self.cleaned_line.clone());
                        self.cleaned_line.clear();
                        // -1 because the new cleaned_line will be added at
                        // the end as the next line
                        self.cleaned_lines.resize(row - 1, "".to_string());
                    }
                }
            }
        }
    }
}

pub fn clean_output(output: &str) -> String {
    let mut parser = OutputParser::new();

    output.chars().for_each(|c| {
        ...
        else if parser.escape == EscapeSequence::InEscape
            || parser.escape == EscapeSequence::InOperatingSystemEscape
        {
            let mut escape_command: Option<char> = None;
            ...
            else if parser.escape == EscapeSequence::InEscape
                && ESCAPE_COMMANDS.contains(&c)
            {
                escape_command = Some(c);
                parser.escape = EscapeSequence::None;
            }
            ...
            else if c == '\u{0007}' {
                escape_command = parser.current_escape_arg.pop();
                parser.escape_args.push(parser.current_escape_arg.clone());
                parser.escape = EscapeSequence::None;
            }
            ...

            if parser.escape == EscapeSequence::None {
                parser.escape_args.push(parser.current_escape_arg.clone());
                parser.current_escape_arg.clear();

                if let Some(escape_command) = escape_command {
                    parser.handle_escape_command(escape_command);
                }

                parser.escape_args.clear();
            }
        }
        ...
    });
    ...
}
```

We decide that we want to do a little more refactoring by renaming variables:

```rs
pub struct OutputParser {
    pub state: EscapeSequence,
    pub cleaned_lines: Vec<String>,
    pub current_line: String,
    pub escape_args: Vec<String>,
    pub current_arg: String,
}
```

Next, we remove the `ESCAPE_COMMANDS` array and simply do a check for

```rs
              else if parser.state == EscapeSequence::InEscape
                && c.is_ascii_alphabetic()
```

Finally, we do a good bit more chunking:

```rs
impl OutputParser {
    ...

    pub fn handle_escape_sequence_start(&mut self, c: char) {
        if c == '[' || c == '(' {
            self.state = EscapeSequence::InEscape;
        } else if c == ']' {
            self.state = EscapeSequence::InOperatingSystemEscape;
        } else {
            self.state = EscapeSequence::None;
            self.current_line.push(c);
        }
    }

    pub fn handle_escape_sequence_input(&mut self, c: char) {
        let mut escape_command: Option<char> = None;
        if c == '?' {
            // it's just a private sequence marker, do nothing
        } else if self.state == EscapeSequence::InEscape && c.is_ascii_alphabetic() {
            escape_command = Some(c);
            self.state = EscapeSequence::None;
        } else if c == ';' {
            self.escape_args.push(self.current_arg.clone());
            self.current_arg.clear();
        } else if c == '\u{0007}' {
            escape_command = self.current_arg.pop();
            self.escape_args.push(self.current_arg.clone());
            self.state = EscapeSequence::None;
        } else {
            self.current_arg.push(c);
        }

        // if we changed the state here
        if self.state == EscapeSequence::None {
            self.escape_args.push(self.current_arg.clone());
            self.current_arg.clear();

            if let Some(escape_command) = escape_command {
                self.handle_escape_command(escape_command);
            }

            self.escape_args.clear();
        }
    }

    ...

    pub fn handle_newline(&mut self) {
        self.state = EscapeSequence::None;
        self.cleaned_lines.push(self.current_line.clone());
        self.current_line.clear();
    }
}

...

pub fn clean_output(output: &str) -> String {
    let mut parser = OutputParser::new();

    output.chars().for_each(|c| {
        if c == '\u{001B}' {
            ...
        } else if parser.state == EscapeSequence::Start {
            parser.handle_escape_sequence_start(c);
        } else if parser.state == EscapeSequence::InEscape
            || parser.state == EscapeSequence::InOperatingSystemEscape
        {
            parser.handle_escape_sequence_input(c);
        } else if c == '\r' {
            ...
        } else if parser.state == EscapeSequence::LineStart {
            if c == '\n' {
                parser.handle_newline();
            } else {
                ...
            }
        } else if c == '\n' {
            parser.handle_newline();
        } else {
            ...
        }
    });

    ...
}
```

#### Keeping track of total output lines

We create `src-tauri/api/sample-calls/send_command_input-cmd-dir.yaml` as such:

```yaml
request:
  - send_command_input
  - >
    {
      "session_id": "319cc7fd-58cc-4320-ab46-2f0ba11c5402",
      "input": "dir"
    }
response:
  message: >
    "dir\n Volume in drive C is Windows\n Volume Serial Number is ..."
sideEffects:
  database:
    startStateDump: command-run-cmd
    endStateDump: command-run-cmd-dir
  terminal:
    recordingFile: windows.cast
    startingIndex: 1

```

and edit `src-tauri/src/commands/terminal/send_input.rs` to test this:

```rs
...
use crate::commands::terminal::parse::clean_output;
...

async fn send_command_input_helper(
    ...
) -> ZammResult<String> {
    ...
    let raw_output = terminal.send_input(&input_with_newline)?;
    let output = clean_output(&raw_output);
    
    ...

    Ok(output)
}

...

#[cfg(test)]
mod tests {
    ...

    #[cfg(target_os = "windows")]
    check_sample!(
        SendInputTestCase,
        test_cmd_dir,
        "./api/sample-calls/send_command_input-cmd-dir.yaml"
    );
}
```

Once the test passes, the `src-tauri/api/sample-database-writes/command-run-cmd-dir/dump.yaml` file looks like this:

```yaml
terminal_sessions:
- id: 319cc7fd-58cc-4320-ab46-2f0ba11c5402
  timestamp: 2024-10-17T06:02:13
  command: cmd
  os: Windows
  cast:
    header:
      version: 2
      width: 80
      height: 24
      timestamp: 1729144933
      command: cmd
    entries:
    - - 0.208
      - 'o'
      - "\e[?25l\e[2J\e[m\e[HMicrosoft Windows [Version 10.0.22631.4169]\r\n(c) Microsoft Corporation. All rights reserved.\e[4;1HC:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\src-tauri>\e]0;C:\\WINDOWS\\system32\\cmd.EXE\a\e[?25h"
    - - 0.208
      - 'i'
      - "dir\r\n"
    - - 0.41
      - 'o'
      - "\e[?25ldir\r\n Volume in drive C is Windows\r\n Volume Serial Number is 30A7-E02E\e[8;1H Directory of ..."
```

We edit the story as well at `src-svelte/src/routes/database/terminal-sessions/TerminalSession.stories.ts`:

```ts
NewOnWindows.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/run_command-cmd.yaml",
    "/api/sample-calls/send_command_input-cmd-dir.yaml",
  ],
};
```

We see that the output looks like this:

```
Microsoft Windows [Version 10.0.22631.4169]
(c) Microsoft Corporation. All rights reserved.

C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-tauri>dir
 Volume in drive C is Windows
 Volume Serial Number is 30A7-E02E




 Directory of C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-tauri

25/09/2024  05:54 pm    <DIR>          .
20/09/2024  07:02 pm    <DIR>          ..
...
```

It turns out this is because the `dir` command outputs an `H` terminal escape code that moves the cursor to the 8th row, but because the parser doesn't realize that there's already been 4 rows of output, it adds way more newlines than expected.

Due to us running the output through the parser cleaner, we have to also update `src-tauri/api/sample-calls/send_command_input-bash-interleaved.yaml` for tests on Unix-like systems:

```yaml
...
response:
  message: >
    "python api/sample-terminal-sessions/interleaved.py\nstdout\nstderr\nstdout\nbash-3.2$ "
...
```

We initially try to add a field to `EscapeSequence` to specify the initial number of rows, and have its `new` function take that in as an argument (ditto for the `clean_output` function), but we realize that we have no way of getting this information to the parser unless we store it somewhere in whichever structs implement `Terminal`. We'd have to add new interface functions to set and retrieve this value, and implement this for both implementors of `Terminal`. We decide to go with a simpler solution for now: replace all 3+ consecutive newlines with just 2 newlines.

We add `regex` and `lazy_static` packages to `src-tauri/Cargo.toml`, the first one to enable regex functionality and the second one so that we can initialize our regex just once at startup time:

```toml
regex = "1.11.1"
lazy_static = "1.5.0"
```

We edit `src-tauri/src/commands/terminal/parse.rs`:

```rs
use lazy_static::lazy_static;
use regex::Regex;

lazy_static! {
    static ref THREE_OR_MORE_NEWLINES: Regex = Regex::new(r"\n{3,}").unwrap();
}

...

pub fn clean_output(output: &str) -> String {
    ...
    THREE_OR_MORE_NEWLINES
        .replace_all(&parser.cleaned_lines.join("\n"), "\n\n")
        .trim_start()
        .to_string()
}
```

Now we can finally update `src-tauri/api/sample-calls/send_command_input-cmd-dir.yaml` to have `Volume Serial Number is 30A7-E02E\n\n Directory` instead of `Volume Serial Number is 30A7-E02E\n\n\n\n\n Directory`. Even though this is a hack, this is still conducive to LLM output because we wouldn't want to be sending too many newlines to the LLM anyways.

## Listing terminal sessions

We realize that we need to reuse the page size variable, so we refactor it out of `src-tauri/src/commands/llms/get_api_calls.rs`:

```rs
use crate::commands::PAGE_SIZE;
```

into `src-tauri/src/commands/mod.rs` instead:

```rs
// size of one page of results in database list view
const PAGE_SIZE: i64 = 50;
```

We also need to export `EntityId` from the database models module in `src-tauri/src/models/mod.rs` to make it usable for our new command:

```rs
pub use llm_calls::EntityId;
```

Then we create `src-tauri/src/commands/terminal/get_sessions.rs` as such:

```rs
use crate::commands::errors::ZammResult;
use crate::commands::PAGE_SIZE;
use crate::models::asciicasts::AsciiCast;
use crate::models::EntityId;
use crate::schema::asciicasts;
use crate::ZammDatabase;
use anyhow::anyhow;
use asciicast::EventType;
use chrono::naive::NaiveDateTime;
use diesel::prelude::*;
use diesel::RunQueryDsl;
use serde::{Deserialize, Serialize};
use specta::specta;
use tauri::State;

#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct TerminalSessionReference {
    pub id: EntityId,
    pub timestamp: NaiveDateTime,
    pub command: String,
    pub last_io: Option<String>,
}

impl From<AsciiCast> for TerminalSessionReference {
    fn from(value: AsciiCast) -> Self {
        let mut last_io = value
            .cast
            .entries
            .iter()
            .filter(|e| e.event_type == EventType::Input)
            .last()
            .map(|e| e.event_data.clone());
        if last_io.is_none() {
            last_io = value
                .cast
                .entries
                .iter()
                .filter(|e| e.event_type == EventType::Output)
                .last()
                .map(|e| e.event_data.clone());
        }
        TerminalSessionReference {
            id: value.id,
            timestamp: value.timestamp,
            command: value.command,
            last_io,
        }
    }
}

async fn get_terminal_sessions_helper(
    zamm_db: &ZammDatabase,
    offset: i32,
) -> ZammResult<Vec<TerminalSessionReference>> {
    let mut db = zamm_db.0.lock().await;
    let conn = db.as_mut().ok_or(anyhow!("Failed to lock database"))?;
    let result: Vec<AsciiCast> = asciicasts::table
        .order(asciicasts::timestamp.desc())
        .offset(offset as i64)
        .limit(PAGE_SIZE)
        .load::<AsciiCast>(conn)?;
    let calls: Vec<TerminalSessionReference> =
        result.into_iter().map(|row| row.into()).collect();
    Ok(calls)
}

#[tauri::command(async)]
#[specta]
pub async fn get_terminal_sessions(
    database: State<'_, ZammDatabase>,
    offset: i32,
) -> ZammResult<Vec<TerminalSessionReference>> {
    get_terminal_sessions_helper(&database, offset).await
}

```

We edit `src-tauri/src/commands/terminal/mod.rs` to export the command:

```rs
...
mod get_sessions;
...

pub use get_sessions::get_terminal_sessions;

```

We edit `src-tauri/src/commands/mod.rs` again to re-export this new function:

```rs
pub use terminal::{get_terminal_sessions, ...};
```

And finally we register this command in `src-tauri/src/main.rs`:

```rs
...
use commands::{
    ..., get_terminal_sessions, ...
};
...

fn main() {
    ...

    match &cli.command {
        #[cfg(debug_assertions)]
        Some(Commands::ExportBindings {}) => {
            let builder = Builder::<tauri::Wry>::new().commands(collect_commands![
                ...,
                get_terminal_sessions,
            ]);
            ...
        }
        Some(Commands::Gui {}) | None => {
          ...

          tauri::Builder::default()
              ...
              .invoke_handler(tauri::generate_handler![
                    ...,
                    get_terminal_sessions,
                ])
              ...;
        }
    }
}
```

Next, we add tests. We edit `src-tauri/src/commands/terminal/get_sessions.rs` to add these tests, and also to trim whitespace from the `last_io` string:

```rs
impl From<AsciiCast> for TerminalSessionReference {
    fn from(value: AsciiCast) -> Self {
        let mut last_io = value
            ...
            .map(|e| e.event_data.trim().to_string());
        if last_io.is_none() {
            last_io = value
                ...
                .map(|e| e.event_data.trim().to_string());
        }
        ...
    }
}

...


#[cfg(test)]
mod tests {
    use super::*;
    use crate::test_helpers::SideEffectsHelpers;
    use crate::{check_sample, impl_result_test_case};
    use serde::{Deserialize, Serialize};

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct GetTerminalSessionsRequest {
        offset: i32,
    }

    async fn make_request_helper(
        args: &GetTerminalSessionsRequest,
        side_effects: &mut SideEffectsHelpers,
    ) -> ZammResult<Vec<TerminalSessionReference>> {
        get_terminal_sessions_helper(side_effects.db.as_ref().unwrap(), args.offset)
            .await
    }

    impl_result_test_case!(
        GetTerminalSessionsTestCase,
        get_terminal_sessions,
        true,
        GetTerminalSessionsRequest,
        Vec<TerminalSessionReference>
    );

    check_sample!(
        GetTerminalSessionsTestCase,
        test_empty_list,
        "./api/sample-calls/get_terminal_sessions-empty.yaml"
    );

    check_sample!(
        GetTerminalSessionsTestCase,
        test_small_list,
        "./api/sample-calls/get_terminal_sessions-small.yaml"
    );
}

```

Our empty test at `src-tauri/api/sample-calls/get_terminal_sessions-empty.yaml` looks like this:

```yaml
request:
  - get_terminal_sessions
  - >
    {
      "offset": 0
    }
response:
  message: >
    []
sideEffects:
  database:
    startStateDump: empty
    endStateDump: empty

```

Our populated one (with only one entry!) at `src-tauri/api/sample-calls/get_terminal_sessions-small.yaml` looks like this:

```yaml
request:
  - get_terminal_sessions
  - >
    {
      "offset": 0
    }
response:
  message: >
    [
      {
        "id": "3717ed48-ab52-4654-9f33-de5797af5118",
        "timestamp": "2024-09-24T16:27:25",
        "command": "bash",
        "last_io": "python api/sample-terminal-sessions/interleaved.py"
      }
    ]
sideEffects:
  database:
    startStateDump: command-run-bash-interleaved
    endStateDump: command-run-bash-interleaved
```

Now we move on to the frontend. We run `export-bindings` to regenerate `src-svelte/src/lib/bindings.ts`. Now that we have upgraded to Tauri v2, we no longer have to worry about manually fixing the `EntityId` to `string` typing.

We refactor the table logic from `src-svelte/src/routes/database/api-calls/ApiCallsTable.svelte` into `src-svelte/src/lib/Table.svelte`, renaming LLM call-related variable names into "blurb" and "item". Note that we're using [generics](https://svelte.dev/docs/svelte/typescript#Generic-$props) to make this component reusable for both API call and terminal session types. We also rename the `api-calls-page` CSS class to `database-page` instead. The slot is now for the default empty placeholder text.

```svelte
<script lang="ts" generics="Item extends { id: string, timestamp: string }">
  ...

  const MIN_BLURB_WIDTH = "5rem";
  const MIN_TIME_WIDTH = "12.5rem";

  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
  export let blurbLabel: string;
  export let itemUrl: (item: Item) => string;
  export let getItems: (offset: number) => Promise<Item[]>;
  export let renderItem: ConstructorOfATypedSvelteComponent;
  let items: Item[] = [];
  let newItemsPromise: Promise<void> | undefined = undefined;
  let allItemsLoaded = false;
  let blurbWidth = MIN_BLURB_WIDTH;
  let timeWidth = MIN_TIME_WIDTH;
  let headerBlurbWidth = MIN_BLURB_WIDTH;

  const formatter = new Intl.DateTimeFormat(dateTimeLocale, {
    ...
  });

  export function formatTimestamp(timestamp: string): string {
    ...
  }

  function getWidths(selector: string) {
    ...
  }

  function resizeBlurbWidth() {
    blurbWidth = MIN_BLURB_WIDTH;
    // time width doesn't need a reset because it never decreases

    setTimeout(() => {
      const textWidths = getWidths(".database-page .text-container");
      const timeWidths = getWidths(".database-page .time");
      const minTextWidth = ...;
      blurbWidth = `${minTextWidth}px`;
      const maxTimeWidth = ...;
      timeWidth = `${maxTimeWidth}px`;

      headerBlurbWidth = blurbWidth;
    }, 10);
  }

  function loadNewItems() {
    if (newItemsPromise) {
      return;
    }

    if (allItemsLoaded) {
      return;
    }

    newItemsPromise = getItems(items.length)
      .then((newItems) => {
        items = [...items, ...newItems];
        allItemsLoaded = newItems.length < PAGE_SIZE;
        newItemsPromise = undefined;

        requestAnimationFrame(resizeBlurbWidth);
      })
      .catch((error) => {
        snackbarError(error);
      });
  }

  onMount(() => {
    resizeBlurbWidth();
    window.addEventListener("resize", resizeBlurbWidth);

    return () => {
      window.removeEventListener("resize", resizeBlurbWidth);
    };
  });

  $: minimumWidths =
    `--blurb-width: ${blurbWidth}; ` +
    `--header-blurb-width: ${headerBlurbWidth}; ` +
    `--time-width: ${timeWidth}`;
</script>

<div class="container database-page full-height" style={minimumWidths}>
  <div class="blurb header">
    <div class="text-container">
      <div class="text">{blurbLabel}</div>
    </div>
    <div class="time">Time</div>
  </div>
  <div class="scrollable-container full-height">
    <Scrollable on:bottomReached={loadNewItems}>
      {#if items.length > 0}
        {#each items as item (item.id)}
          <a href={itemUrl(item)}>
            <div class="blurb instance">
              <div class="text-container">
                <div class="text">
                  <svelte:component this={renderItem} {item} />
                </div>
              </div>
              <div class="time">{formatTimestamp(item.timestamp)}</div>
            </div>
          </a>
        {/each}
      {:else}
        <div class="blurb placeholder">
          <div class="text-container">
            <EmptyPlaceholder>
              <slot />
            </EmptyPlaceholder>
          </div>
          <div class="time"></div>
        </div>
      {/if}
    </Scrollable>
  </div>
</div>

<style>
  ...

  .blurb {
    display: flex;
    color: black;
  }

  .blurb.placeholder :global(p) {
    margin-top: 0.5rem;
  }

  .blurb.header .text-container,
  .blurb.header .time {
    text-align: center;
    font-weight: bold;
  }

  .blurb .text-container {
    flex: 1;
  }

  .blurb.instance {
    padding: 0.2rem var(--side-padding);
    border-radius: var(--corner-roundness);
    transition: background 0.5s;
    height: 1.62rem;
    box-sizing: border-box;
  }

  .blurb.instance:hover {
    background: var(--color-hover);
  }

  .blurb .text-container .text {
    max-width: var(--blurb-width);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .blurb.header .text-container .text {
    max-width: var(--header-blurb-width);
  }

  .blurb .time {
    min-width: var(--time-width);
    box-sizing: border-box;
    text-align: right;
  }
</style>

```

Because the CSS class name has changed, we must also edit `webdriver/test/specs/e2e.test.js` to fit:

```js
    it("should be able to view single LLM call", async function () {
      ...
      await findAndClick(".database-page a:nth-child(2)");
      ...
      await findAndClick(".database-page a:nth-child(2)");
      ...
    });
```

We create `src-svelte/src/routes/database/api-calls/ApiCallBlurb.svelte` to serve as the component that renders the API call blurb in the row:

```svelte
<script lang="ts">
  import ApiCallReference from "$lib/ApiCallReference.svelte";
  import { type LightweightLlmCall } from "$lib/bindings";

  export let item: LightweightLlmCall;
  $: reference = {
    id: item.id,
    snippet: item.response_message.text.trim(),
  };
</script>

<ApiCallReference selfContained nolink apiCall={reference} />

```

All that remains of `src-svelte/src/routes/database/api-calls/ApiCallsTable.svelte` is this customization of the generic `Table`:

```svelte
<script lang="ts">
  import Table from "$lib/Table.svelte";
  import { commands, type LightweightLlmCall } from "$lib/bindings";
  import { unwrap } from "$lib/tauri";
  import ApiCallBlurb from "./ApiCallBlurb.svelte";

  const getApiCalls = (offset: number) => unwrap(commands.getApiCalls(offset));
  const apiCallUrl = (apiCall: LightweightLlmCall) =>
    `/database/api-calls/${apiCall.id}/`;

  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
</script>

<Table
  blurbLabel="Message"
  getItems={getApiCalls}
  itemUrl={apiCallUrl}
  renderItem={ApiCallBlurb}
  {dateTimeLocale}
  {timeZone}
>
  Looks like you haven't made any calls to an LLM yet.<br />Get started via
  <a href="/chat">chat</a>
  or by making one <a href="/database/api-calls/new/">from scratch</a>.
</Table>

```

Similarly, we create a `src-svelte/src/routes/database/terminal-sessions/TerminalSessionBlurb.svelte`:

```svelte
<script lang="ts">
  import { type TerminalSessionReference } from "$lib/bindings";

  export let item: TerminalSessionReference;
</script>

<div class="ellipsis-container">
  <span class="blurb">
    {item.command}
    {#if item.last_io}
      <span class="last-io">{item.last_io}</span>
    {/if}
  </span>
</div>

<style>
  .ellipsis-container {
    display: flex;
    width: 100%;
  }

  .blurb {
    min-width: 0;
    flex: 1;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    line-height: 1.22rem;
  }

  .last-io {
    margin-left: 0.5rem;
    color: var(--color-faded);
    font-style: italic;
  }
</style>

```

and the corresponding `src-svelte/src/routes/database/terminal-sessions/TerminalSessionsTable.svelte`:

```svelte
<script lang="ts">
  import Table from "$lib/Table.svelte";
  import { commands, type TerminalSessionReference } from "$lib/bindings";
  import { unwrap } from "$lib/tauri";
  import TerminalSessionBlurb from "./TerminalSessionBlurb.svelte";

  const getTerminalSessions = (offset: number) =>
    unwrap(commands.getTerminalSessions(offset));
  const terminalSessionUrl = (apiCall: TerminalSessionReference) =>
    `/database/terminal-sessions/${apiCall.id}/`;

  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
</script>

<Table
  blurbLabel="Command"
  getItems={getTerminalSessions}
  itemUrl={terminalSessionUrl}
  renderItem={TerminalSessionBlurb}
  {dateTimeLocale}
  {timeZone}
>
  Looks like you haven't <a href="/database/terminal-sessions/new/">started</a> any
  terminal sessions yet.
</Table>

```

We edit `src-svelte/src/routes/database/DatabaseView.svelte` to use both tables:

```svelte
<script lang="ts">
  ...
  import TerminalSessionsTable from "./terminal-sessions/TerminalSessionsTable.svelte";
  ...
</script>

<InfoBox ...>
  <div class="container full-height">
    ...
    {#if $dataType === "llm-calls"}
      <ApiCallsTable {dateTimeLocale} {timeZone} />
    {:else}
      <TerminalSessionsTable {dateTimeLocale} {timeZone} />
    {/if}
  </div>
</InfoBox>
```

We test this in `src-svelte/src/routes/database/DatabaseView.test.ts`. Because we want to trigger the API load on component mount, we mock the IntersectionObserver to automatically invoke an intersection. We also split the existing two into two separate ones to test the tables in both screens separately:

```ts
import { ..., type Mock } from "vitest";
...
import {
  TauriInvokePlayback,
  stubGlobalInvoke,
} from "$lib/sample-call-testing";

class MockIntersectionObserver {
  constructor(callback: IntersectionObserverCallback) {
    setTimeout(() => {
      const entry: IntersectionObserverEntry = {
        isIntersecting: true,
        boundingClientRect: new DOMRectReadOnly(),
        intersectionRatio: 1,
        intersectionRect: new DOMRectReadOnly(),
        rootBounds: new DOMRectReadOnly(),
        target: document.createElement("div"),
        time: 0,
      };
      callback([entry], this as unknown as IntersectionObserver);
    }, 10);
  }

  observe = vi.fn();
  unobserve = vi.fn();
  disconnect = vi.fn();
}

describe("Database View", () => {
  let tauriInvokeMock: Mock;
  let playback: TauriInvokePlayback;

  beforeEach(() => {
    tauriInvokeMock = vi.fn();
    stubGlobalInvoke(tauriInvokeMock);
    playback = new TauriInvokePlayback();
    tauriInvokeMock.mockImplementation(
      (...args: (string | Record<string, string>)[]) =>
        playback.mockCall(...args),
    );

    window.IntersectionObserver =
      MockIntersectionObserver as unknown as typeof IntersectionObserver;
    playback.addSamples(
      "../src-tauri/api/sample-calls/get_api_calls-small.yaml",
    );
  });

  ...

  test("renders LLM calls by default", async () => {
    render(DatabaseView, {
      dateTimeLocale: "en-GB",
      timeZone: "Asia/Phnom_Penh",
    });

    expect(screen.getByRole("heading")).toHaveTextContent("LLM API Calls");
    await waitFor(() => expect(tauriInvokeMock).toHaveReturnedTimes(1));
    await waitFor(() => {
      const linkToApiCall = screen.getByRole("link", {
        name: /yes, it works/i,
      });
      expect(linkToApiCall).toBeInTheDocument();
      expect(linkToApiCall).toHaveAttribute(
        "href",
        "/database/api-calls/d5ad1e49-f57f-4481-84fb-4d70ba8a7a74/",
      );
    });
  });

  test("can switch to rendering terminal sessions list", async () => {
    render(DatabaseView, {
      dateTimeLocale: "en-GB",
      timeZone: "Asia/Phnom_Penh",
    });
    await waitFor(() => expect(tauriInvokeMock).toHaveReturnedTimes(1));

    tauriInvokeMock.mockClear();
    playback.addSamples(
      "../src-tauri/api/sample-calls/get_terminal_sessions-small.yaml",
    );
    userEvent.selectOptions(
      screen.getByRole("combobox", { name: "Showing" }),
      "Terminal Sessions",
    );
    await waitFor(() => {
      expect(screen.getByRole("heading")).toHaveTextContent(
        "Terminal Sessions",
      );
    });
    await waitFor(() => expect(tauriInvokeMock).toHaveReturnedTimes(1));
    await waitFor(() => {
      const linkToTerminalSession = screen.getByRole("link", {
        name: /python api/i,
      });
      expect(linkToTerminalSession).toBeInTheDocument();
      expect(linkToTerminalSession).toHaveAttribute(
        "href",
        "/database/terminal-sessions/3717ed48-ab52-4654-9f33-de5797af5118/",
      );
    });
  });
});

```

We add the latest API call to `src-svelte/src/routes/database/DatabaseView.stories.ts` for manual testing:

```ts
Full.parameters = {
  ...,
  sampleCallFiles: [
    ...,
    "/api/sample-calls/get_terminal_sessions-small.yaml",
  ],
};
```

We do a little housecleaning and edit `src-svelte/src/routes/database/api-calls/ApiCallsTable.svelte` to use `$$restProps` instead:

```svelte
<Table
  blurbLabel="Message"
  getItems={getApiCalls}
  itemUrl={apiCallUrl}
  renderItem={ApiCallBlurb}
  {...$$restProps}
>
```

We do the same for `src-svelte/src/routes/database/terminal-sessions/TerminalSessionsTable.svelte`:

```svelte
<Table
  blurbLabel="Command"
  getItems={getTerminalSessions}
  itemUrl={terminalSessionUrl}
  renderItem={TerminalSessionBlurb}
  {...$$restProps}
>
```

Next, we edit `src-svelte/src/lib/Table.svelte` to set a bigger header blurb width before the resize, so as to reduce flicker.

```ts
  let headerBlurbWidth = "15rem"; // 3 * MIN_BLURB_WIDTH
```

Later on, we realize from a `git bisect` that `src-svelte/src/routes/database/DatabaseView.playwright.test.ts` is now failing after this commit. It turns out this is just because we changed the CSS classes, and so we edit the function to look for `.blurb.instance` instead of `.message.instance`:

```ts
  const expectLastMessage = async (
    ...
  ) => {
    ...
    const lastMessageContainer = apiCallsScrollElement.locator(
      "... .blurb.instance ...",
    );
    ...
  };
```

### Creating the slug page

Now, we have to create the slug page so that the links from the terminal sessions list page actually work.

First, however, we'll need to create the backend API call that the slug will use. We create `src-tauri/src/commands/terminal/get_session.rs`:

```rs
use super::parse::clean_output;
use crate::commands::errors::ZammResult;
use crate::models::asciicasts::AsciiCast;
use crate::models::os::OS;
use crate::models::EntityId;
use crate::schema::asciicasts;
use crate::ZammDatabase;
use anyhow::anyhow;
use chrono::NaiveDateTime;
use diesel::prelude::*;
use diesel::RunQueryDsl;
use serde::{Deserialize, Serialize};
use specta::specta;
use tauri::State;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct RecoveredTerminalSession {
    id: EntityId,
    timestamp: NaiveDateTime,
    command: String,
    os: Option<OS>,
    output: String,
}

async fn get_terminal_session_helper(
    zamm_db: &ZammDatabase,
    id: &str,
) -> ZammResult<RecoveredTerminalSession> {
    let parsed_uuid = EntityId {
        uuid: Uuid::parse_str(id)?,
    };
    let mut db = zamm_db.0.lock().await;
    let conn = db.as_mut().ok_or(anyhow!("Failed to lock database"))?;

    let result: AsciiCast = asciicasts::table
        .filter(asciicasts::id.eq(parsed_uuid))
        .first::<AsciiCast>(conn)?;
    let concantenated_output = result
        .cast
        .entries
        .iter()
        .flat_map(|e| {
            if e.event_type == asciicast::EventType::Output {
                Some(clean_output(&e.event_data))
            } else {
                None
            }
        })
        .collect::<Vec<String>>()
        .join("\n");
    let recovered_session = RecoveredTerminalSession {
        id: result.id,
        timestamp: result.timestamp,
        command: result.command.clone(),
        os: result.os,
        output: concantenated_output,
    };
    Ok(recovered_session)
}

#[tauri::command(async)]
#[specta]
pub async fn get_terminal_session(
    database: State<'_, ZammDatabase>,
    id: &str,
) -> ZammResult<RecoveredTerminalSession> {
    get_terminal_session_helper(&database, id).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::test_helpers::SideEffectsHelpers;
    use crate::{check_sample, impl_result_test_case};
    use serde::{Deserialize, Serialize};

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct GetTerminalSessionRequest {
        id: String,
    }

    async fn make_request_helper(
        args: &GetTerminalSessionRequest,
        side_effects: &mut SideEffectsHelpers,
    ) -> ZammResult<RecoveredTerminalSession> {
        get_terminal_session_helper(side_effects.db.as_ref().unwrap(), &args.id).await
    }

    impl_result_test_case!(
        GetTerminalSessionTestCase,
        get_terminal_session,
        true,
        GetTerminalSessionRequest,
        RecoveredTerminalSession
    );

    check_sample!(
        GetTerminalSessionTestCase,
        test_start_bash,
        "./api/sample-calls/get_terminal_session-bash.yaml"
    );

    check_sample!(
        GetTerminalSessionTestCase,
        test_bash_interleaved,
        "./api/sample-calls/get_terminal_session-bash-interleaved.yaml"
    );
}

```

At first, we try to return an `AsciiCast` directly and edit `src-tauri/src/models/asciicasts.rs` to derive `specta::Type` for `AsciiCastData`:

```rs
use asciicast::{Entry, Header};

#[derive(..., specta::Type)]
pub struct AsciiCastData {
    pub header: Header,
    pub entries: Vec<Entry>,
}
```

But then of course we get

```
the trait bound `Header: specta::Type` is not satisfied
```

But if we try to manually implement it

```rs
use specta::datatype::{DataType, StructType};

impl specta::Type for AsciiCastData {
    fn inline(type_map: &mut specta::TypeMap, generics: specta::Generics) -> DataType {
        DataType::Struct(
            StructType {
                name: "AsciiCastData".to_string(),
                sid: None,
                fields: vec![
                    ...
                ],
                generics: vec![],
            }
        )
    }
}
```

then we get an error because those fields are `pub[crate]`. We create [this issue](https://github.com/specta-rs/specta/issues/286) to ask for a way to derive `specta::Type` for our own custom structs, and apparently the official answer is to make a workaround struct. We decide not to do this in the end, but it is good to know for future purposes. Instead, for now we return the string as the frontend would want to display it.

We create `src-tauri/api/sample-calls/get_terminal_session-bash.yaml` to return a fresh session that we can then invoke I/O on:

```yaml
request:
  - get_terminal_session
  - >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118"
    }
response:
  message: >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118",
      "timestamp": "2024-09-24T16:27:25",
      "command": "bash",
      "os": "Mac",
      "output": "The default interactive shell is now zsh.\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\nFor more details, please visit https://support.apple.com/kb/HT208050.\nbash-3.2$ "
    }
sideEffects:
  database:
    startStateDump: command-run-bash
    endStateDump: command-run-bash

```

We create `src-tauri/api/sample-calls/get_terminal_session-bash-interleaved.yaml` to return a fuller transcript of a session that involves I/O:

```yaml
request:
  - get_terminal_session
  - >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118"
    }
response:
  message: >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118",
      "timestamp": "2024-09-24T16:27:25",
      "command": "bash",
      "os": "Mac",
      "output": "The default interactive shell is now zsh.\nTo update your account to use zsh, please run `chsh -s /bin/zsh`.\nFor more details, please visit https://support.apple.com/kb/HT208050.\nbash-3.2$ \npython api/sample-terminal-sessions/interleaved.py\nstdout\nstderr\nstdout\nbash-3.2$ "
    }
sideEffects:
  database:
    startStateDump: command-run-bash-interleaved
    endStateDump: command-run-bash-interleaved

```

Both these tests pass. We expose this API call in `src-tauri/src/commands/terminal/mod.rs`:

```rs
mod get_session;
...

pub use get_session::get_terminal_session;
...
```

and again in `src-tauri/src/commands/mod.rs`:

```rs
pub use terminal::{
    get_terminal_session, ...
};
```

and register it in `src-tauri/src/main.rs` as usual.

Now, we create the frontend page for this. We update the bindings as usual, and create `src-svelte/src/routes/database/terminal-sessions/[slug]/+page.svelte` as such:

```svelte
<script lang="ts">
  import TerminalSession from "../TerminalSession.svelte";
  import { commands, type RecoveredTerminalSession } from "$lib/bindings";
  import { onMount } from "svelte";
  import { unwrap } from "$lib/tauri";
  import { page } from "$app/stores";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";

  let terminalSession: RecoveredTerminalSession | undefined = undefined;

  onMount(async () => {
    try {
      terminalSession = await unwrap(
        commands.getTerminalSession($page.params.slug),
      );
    } catch (error) {
      snackbarError(error as string | Error);
    }
  });
</script>

{#if terminalSession}
  <TerminalSession
    sessionId={terminalSession.id}
    command={terminalSession.command}
    output={terminalSession.output}
  />
{:else}
  <p>Loading...</p>
{/if}

```

Upon browsing to a past terminal session, we find that we have to edit `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte` to mark the command div as `atomic-reveal`:

```svelte
    {#if command}
      <p class="atomic-reveal">
        Current command: <span class="command">{command}</span>
      </p>
    {:else}
```

As it is, unfortunately the form to send input at the bottom is still present, even though it of course doesn't work anymore because the terminal session is no longer even active. As such, we edit `src-tauri/src/commands/terminal/get_session.rs` to return an `is_active` field. While doing so, we are reminded that our Tauri backend API can take in `Uuid` directly instead of `String`. We also use a custom test case instead of the default one constructed by the macro:

```rs
...
use crate::ZammTerminalSessions;
...

#[derive(...)]
pub struct RecoveredTerminalSession {
    ...
    is_active: bool,
}

async fn get_terminal_session_helper(
    ...,
    zamm_sessions: &ZammTerminalSessions,
    id: Uuid,
) -> ZammResult<RecoveredTerminalSession> {
    let mut db = ...;
    let conn = ...;
    let sessions = zamm_sessions.0.lock().await;

    let parsed_uuid: EntityId = id.into();
    let result: AsciiCast = asciicasts::table
        .filter(asciicasts::id.eq(&parsed_uuid))
        ...;
    ...
    let is_active = sessions.contains_key(&parsed_uuid);
    let recovered_session = RecoveredTerminalSession {
        ...,
        is_active,
    };
    ...
}

#[tauri::command(async)]
#[specta]
pub async fn get_terminal_session(
    ...,
    zamm_sessions: State<'_, ZammTerminalSessions>,
    id: Uuid,
) -> ZammResult<RecoveredTerminalSession> {
    get_terminal_session_helper(..., &zamm_sessions, id).await
}

#[cfg(test)]
mod tests {
    ...
    use crate::sample_call::SampleCall;
    use crate::test_helpers::api_testing::{standard_test_subdir, TerminalHelper};
    use crate::test_helpers::{
        SampleCallTestCase, SideEffectsHelpers, ZammResultReturn,
    };
    ...
    use stdext::function_name;

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct GetTerminalSessionRequest {
        id: Uuid,
    }

    struct GetTerminalSessionTestCase {
        test_fn_name: &'static str,
        session_should_exist: bool,
    }

    impl
        SampleCallTestCase<
            GetTerminalSessionRequest,
            ZammResult<RecoveredTerminalSession>,
        > for GetTerminalSessionTestCase
    {
        const EXPECTED_API_CALL: &'static str = "get_terminal_session";
        const CALL_HAS_ARGS: bool = true;

        fn temp_test_subdirectory(&self) -> String {
            standard_test_subdir(Self::EXPECTED_API_CALL, self.test_fn_name)
        }

        async fn make_request(
            &mut self,
            args: &GetTerminalSessionRequest,
            side_effects: &mut SideEffectsHelpers,
        ) -> ZammResult<RecoveredTerminalSession> {
            let mut terminal_helper = TerminalHelper::new();
            if self.session_should_exist {
                terminal_helper.change_mock_id(args.id).await;
            }

            get_terminal_session_helper(
                side_effects.db.as_ref().unwrap(),
                &terminal_helper.sessions,
                args.id,
            )
            .await
        }

        fn serialize_result(
            &self,
            sample: &SampleCall,
            result: &ZammResult<RecoveredTerminalSession>,
        ) -> String {
            ZammResultReturn::serialize_result(self, sample, result)
        }

        async fn check_result(
            &self,
            sample: &SampleCall,
            args: &GetTerminalSessionRequest,
            result: &ZammResult<RecoveredTerminalSession>,
        ) {
            ZammResultReturn::check_result(self, sample, args, result).await
        }
    }

    impl ZammResultReturn<GetTerminalSessionRequest, RecoveredTerminalSession>
        for GetTerminalSessionTestCase
    {
    }

    #[tokio::test]
    async fn test_start_bash() {
        let mut test_case = GetTerminalSessionTestCase {
            test_fn_name: function_name!(),
            session_should_exist: true,
        };
        test_case
            .check_sample_call("./api/sample-calls/get_terminal_session-bash.yaml")
            .await;
    }

    #[tokio::test]
    async fn test_bash_interleaved() {
        let mut test_case = GetTerminalSessionTestCase {
            test_fn_name: function_name!(),
            session_should_exist: false,
        };
        test_case
            .check_sample_call(
                "./api/sample-calls/get_terminal_session-bash-interleaved.yaml",
            )
            .await;
    }
}

```

Note that for convenience, we edit `src-tauri/src/models/llm_calls/entity_id.rs` to make it easier to construct an `EntityId`:

```rs
impl From<Uuid> for EntityId {
    fn from(uuid: Uuid) -> Self {
        EntityId { uuid }
    }
}

impl TryFrom<&str> for EntityId {
    type Error = uuid::Error;

    fn try_from(value: &str) -> Result<Self, Self::Error> {
        let uuid = Uuid::parse_str(value)?;
        Ok(EntityId { uuid })
    }
}
```

We also edit `src-tauri/src/test_helpers/api_testing.rs` to make it possible to easily modify the state of the terminal sessions:

```rs
...
use crate::commands::terminal::{ActualTerminal, ...};
...
use uuid::Uuid;
...

impl TerminalHelper {
    pub fn new() -> Self {
        let new_session_id = EntityId::new();
        let sessions = ZammTerminalSessions(Mutex::new(HashMap::from([(
            new_session_id.clone(),
            Box::new(ActualTerminal::new()) as Box<dyn Terminal>,
        )])));
        TerminalHelper {
            sessions,
            mock_session_id: new_session_id,
        }
    }

    pub async fn change_mock_id(&mut self, new_uuid: Uuid) {
        let new_id: EntityId = new_uuid.into();
        let mut sessions = self.sessions.0.lock().await;
        let mock_terminal = sessions.remove(&self.mock_session_id).unwrap();
        sessions.insert(new_id.clone(), mock_terminal);

        self.mock_session_id = new_id;
    }
}
```

We refactor `src-tauri/src/commands/terminal/send_input.rs` to use this new method as well:

```rs
    async fn make_request_helper(
        ...,
        side_effects: &mut SideEffectsHelpers,
    ) -> ZammResult<String> {
        let terminal_helper = side_effects.terminal.as_mut().unwrap();
        terminal_helper.change_mock_id(args.session_id).await;
        ...
    }
```

We edit `src-tauri/api/sample-calls/get_terminal_session-bash-interleaved.yaml` to reflect this new feature:

```yaml
response:
  message: >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118",
      ...,
      "output": "The default interactive shell is now zsh...",
      "is_active": false
    }
```

and `src-tauri/api/sample-calls/get_terminal_session-bash.yaml` for an example where it is true:

```yaml
response:
  message: >
    {
      "id": "3717ed48-ab52-4654-9f33-de5797af5118",
      ...,
      "output": "The default interactive shell is now zsh...",
      "is_active": true
    }
```

Now we update the frontend to make use of this new information. We export the bindings to `src-svelte/src/lib/bindings.ts` as usual, and edit `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte` to make use of this information. We change the command label from `Current command` to just `Command` because "current" doesn't make as much sense if the session isn't active anymore. We also wrap the input form in the `isActive` check:

```svelte
<script lang="ts">
  ...
  export let isActive = true;
  ...
</script>

<InfoBox title="Terminal Session" ...>
  <div class="terminal-container ...">
    {#if command}
      <p class="atomic-reveal">
        Command: ...
      </p>
    {:else}
      ...
    {/if}

    ...

    {#if isActive}
      <SendInputForm
        ...
      />
    {:else}
      <EmptyPlaceholder>
        This terminal session is no longer active.
      </EmptyPlaceholder>
    {/if}
```

We then create `src-svelte/src/routes/database/terminal-sessions/UnloadedTerminalSession.svelte`, which basically has the contents of the old `src-svelte/src/routes/database/terminal-sessions/[slug]/+page.svelte` except that it takes in an `id` instead of getting it from the page slug:

```svelte
<script lang="ts">
  import TerminalSession from "./TerminalSession.svelte";
  import { commands, type RecoveredTerminalSession } from "$lib/bindings";
  import { onMount } from "svelte";
  import { unwrap } from "$lib/tauri";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";

  export let id: string;
  let terminalSession: RecoveredTerminalSession | undefined = undefined;

  onMount(async () => {
    try {
      terminalSession = await unwrap(commands.getTerminalSession(id));
    } catch (error) {
      snackbarError(error as string | Error);
    }
  });
</script>

{#if terminalSession}
  <TerminalSession
    sessionId={terminalSession.id}
    command={terminalSession.command}
    output={terminalSession.output}
    isActive={terminalSession.is_active}
  />
{:else}
  <p>Loading...</p>
{/if}

```

The old `src-svelte/src/routes/database/terminal-sessions/[slug]/+page.svelte` now looks like this instead:

```svelte
<script lang="ts">
  import UnloadedTerminalSession from "../UnloadedTerminalSession.svelte";
  import { page } from "$app/stores";
</script>

<UnloadedTerminalSession id={$page.params.slug} />

```

We test the terminal session loading at `src-svelte/src/routes/database/terminal-sessions/UnloadedTerminalSession.test.ts`:

```ts
import { expect, test, vi, type Mock } from "vitest";
import "@testing-library/jest-dom";
import { render, screen, waitFor } from "@testing-library/svelte";
import UnloadedTerminalSession from "./UnloadedTerminalSession.svelte";
import userEvent from "@testing-library/user-event";
import {
  TauriInvokePlayback,
  stubGlobalInvoke,
} from "$lib/sample-call-testing";

describe("Unloaded terminal session", () => {
  let tauriInvokeMock: Mock;
  let playback: TauriInvokePlayback;

  beforeEach(() => {
    tauriInvokeMock = vi.fn();
    stubGlobalInvoke(tauriInvokeMock);
    playback = new TauriInvokePlayback();
    tauriInvokeMock.mockImplementation(
      (...args: (string | Record<string, string>)[]) =>
        playback.mockCall(...args),
    );

    window.IntersectionObserver = vi.fn(() => {
      return {
        observe: vi.fn(),
        unobserve: vi.fn(),
        disconnect: vi.fn(),
      };
    }) as unknown as typeof IntersectionObserver;
  });

  afterEach(() => {
    vi.unstubAllGlobals();
  });

  test("can resume active session", async () => {
    playback.addSamples(
      "../src-tauri/api/sample-calls/get_terminal_session-bash.yaml",
    );
    render(UnloadedTerminalSession, {
      id: "3717ed48-ab52-4654-9f33-de5797af5118",
    });

    // check that the page loads correctly
    await waitFor(() => {
      expect(tauriInvokeMock).toHaveReturnedTimes(1);
      expect(
        screen.getByText(
          new RegExp("The default interactive shell is now zsh"),
        ),
      ).toBeInTheDocument();
    });

    // check that we can still interact with the session
    playback.addSamples(
      "../src-tauri/api/sample-calls/send_command_input-bash-interleaved.yaml",
    );
    // this part differs from TerminalSession.test.ts
    const commandInput = screen.getByLabelText("Enter input for command");
    const sendButton = screen.getByRole("button", { name: "Send" });
    await userEvent.type(
      commandInput,
      "python api/sample-terminal-sessions/interleaved.py",
    );
    await userEvent.click(sendButton);
    expect(tauriInvokeMock).toHaveReturnedTimes(2);
    await waitFor(() => {
      expect(screen.getByText(new RegExp("stderr"))).toBeInTheDocument();
    });
  });

  test("can load inactive session without allowing for further input", async () => {
    playback.addSamples(
      "../src-tauri/api/sample-calls/get_terminal_session-bash-interleaved.yaml",
    );
    render(UnloadedTerminalSession, {
      id: "3717ed48-ab52-4654-9f33-de5797af5118",
    });

    // check that the page loads correctly
    await waitFor(() => {
      expect(tauriInvokeMock).toHaveReturnedTimes(1);
      const commandOutput = screen.getByText(
        new RegExp("The default interactive shell is now zsh"),
      );
      expect(commandOutput).toBeInTheDocument();
      expect(commandOutput).toHaveTextContent("stderr");
    });

    // check that we can no longer interact with the session
    const commandInput = screen.queryByLabelText("Enter input for command");
    // expect inpupt to not exist
    expect(commandInput).not.toBeInTheDocument();
    expect(
      screen.getByText("This terminal session is no longer active."),
    ).toBeInTheDocument();
  });
});

```

In the process of creating this test file, we realize that the test at `src-svelte/src/routes/database/terminal-sessions/TerminalSession.test.ts` should be changed from `"Terminal session"` (which is the name for the test *suite*):

```ts
describe("Terminal session", () => {
  ...

  test("can start and send input to command", async () => {
    ...
  });
});
```

We move the in-progress terminal session story from `src-svelte/src/routes/database/terminal-sessions/TerminalSession.stories.ts` to `src-svelte/src/routes/database/terminal-sessions/UnloadedTerminalSession.stories.ts`, which is a new file that also contains an example of a finished terminal session:

```ts
import UnloadedTerminalSession from "./UnloadedTerminalSession.svelte";
import type { StoryFn, StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";
import SvelteStoresDecorator from "$lib/__mocks__/stores";
import MockFullPageLayout from "$lib/__mocks__/MockFullPageLayout.svelte";

export default {
  component: UnloadedTerminalSession,
  title: "Screens/Database/Terminal Session",
  argTypes: {},
  decorators: [
    SvelteStoresDecorator,
    TauriInvokeDecorator,
    (story: StoryFn) => {
      return {
        Component: MockFullPageLayout,
        slot: story,
      };
    },
  ],
};

const Template = ({ ...args }) => ({
  Component: UnloadedTerminalSession,
  props: args,
});

export const InProgress: StoryObj = Template.bind({}) as any;
InProgress.args = {
  id: "3717ed48-ab52-4654-9f33-de5797af5118",
};
InProgress.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/get_terminal_session-bash.yaml",
    "/api/sample-calls/send_command_input-bash-interleaved.yaml",
  ],
};

export const Finished: StoryObj = Template.bind({}) as any;
Finished.args = {
  id: "3717ed48-ab52-4654-9f33-de5797af5118",
};
Finished.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/get_terminal_session-bash-interleaved.yaml",
  ],
};

```

We add the new story to `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "database", "terminal-session"],
    variants: [..., "finished"],
    ...
  },
  ...
];
```

While viewing the new story, we realize that there is a stray newline in between the bash prompt and the input to the bash command. We fix this in `src-tauri/src/commands/terminal/get_session.rs` by joining on an empty string instead of `\n`:

```rs
async fn get_terminal_session_helper(
    ...
) -> ZammResult<RecoveredTerminalSession> {
    ...
    let concantenated_output = result
        ...
        .join("");
    ...
}
```

We update `src-tauri/api/sample-calls/get_terminal_session-bash-interleaved.yaml` to match, so that it looks like `"...bash-3.2$ python..."` instead of `"...bash-3.2$ \npython..."`.

We take a detour below, but in doing so, we realize that we should edit `src-svelte/src/routes/database/terminal-sessions/UnloadedTerminalSession.svelte` to use our own proper loading component:

```svelte
<script lang="ts">
  ...
  import Loading from "$lib/Loading.svelte";
  ...
</script>

{#if terminalSession}
  ...
{:else}
  <Loading />
{/if}
```

### Detour: refactoring API call page to use `Unloaded` naming schema

Doing this for terminal sessions makes us want to do it for our API calls too. To keep Git history clean, we first rename `ApiCall` to `UnloadedApiCall`. This is a straightforward task that the IDE does well.

Next, we rename `ApiCallDisplay` to `ApiCall`. The IDE does this well too, but we further edit the import in `src-svelte/src/routes/database/api-calls/[slug]/UnloadedApiCall.svelte` from

```ts
  import ApiCallDisplay from "./ApiCall.svelte";
```

to

```ts
  import ApiCall from "./ApiCall.svelte";
```

This is a detail that the IDE doesn't automatically take care of, for understandable reasons.

Next, we edit the Storybook stories. For example, `src-svelte/src/routes/database/api-calls/[slug]/ApiCall.stories.ts` now only imports

```ts
import { KHMER_CALL, LOTS_OF_CODE_CALL } from "./sample-calls";
```

and the stories `Narrow`, `Wide`, `Variant` and `UnknownProviderPrompt` are all removed from this file. Instead, the new file `src-svelte/src/routes/database/api-calls/[slug]/UnloadedApiCall.stories.ts` looks like this:

```ts
import UnloadedApiCall from "./UnloadedApiCall.svelte";
import type { StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";

export default {
  component: UnloadedApiCall,
  title: "Screens/Database/LLM Call",
  argTypes: {},
  decorators: [TauriInvokeDecorator],
};

const Template = ({ ...args }) => ({
  Component: UnloadedApiCall,
  props: args,
});

export const Regular: StoryObj = Template.bind({}) as any;
Regular.args = {
  id: "c13c1e67-2de3-48de-a34c-a32079c03316",
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
Regular.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/get_api_call-continue-conversation.yaml",
  ],
};

export const Variant: StoryObj = Template.bind({}) as any;
Variant.args = {
  id: "7a35a4cf-f3d9-4388-bca8-2fe6e78c9648",
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
Variant.parameters = {
  sampleCallFiles: ["/api/sample-calls/get_api_call-edit.yaml"],
};

export const UnknownProviderPrompt: StoryObj = Template.bind({}) as any;
UnknownProviderPrompt.args = {
  id: "037b28dd-6f24-4e68-9dfb-3caa1889d886",
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
UnknownProviderPrompt.parameters = {
  sampleCallFiles: [
    "/api/sample-calls/get_api_call-unknown-provider-prompt.yaml",
  ],
};

```

Note that the `Narrow` and `Wide` stories are smushed into one because we will be manually changing the Storybook window size at test time. Instead, we edit `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "database", "llm-call"],
    variants: [
      {
        name: "narrow",
        prefix: "regular",
        ...
      },
      {
        name: "wide",
        prefix: "regular",
        ...
      },
      ...
  },
  ...
];
```

`src-svelte/src/routes/database/api-calls/[slug]/sample-calls.ts` loses the sample calls `CONTINUE_CONVERSATION_CALL`, `VARIANT_CALL`, and `UNKNOWN_PROVIDER_PROMPT_CALL`. However, the first prompt is actually still needed in other tests, so we keep a remnant of it:

```ts
export const CONTINUE_CONVERSATION_PROMPT = {
  type: "Chat",
  messages: [
    {
      role: "System",
      text: "You are ZAMM, a chat program. Respond in first person.",
    },
    {
      role: "Human",
      text: "Hello, does this work?",
    },
    {
      role: "AI",
      text: "Yes, it works. How can I assist you today?",
    },
    {
      role: "Human",
      text: "Tell me something funny.",
    },
  ],
};
```

We then edit `src-svelte/src/routes/database/api-calls/[slug]/Prompt.stories.ts` from this:

```ts
import { CONTINUE_CONVERSATION_CALL } from "./sample-calls";
...

export const Uneditable: StoryObj = ...;
Uneditable.args = {
  prompt: CONTINUE_CONVERSATION_CALL.request.prompt,
};

export const Editable: StoryObj = ...;
Editable.args = {
  ...
  prompt: CONTINUE_CONVERSATION_CALL.request.prompt,
};
```

to this:

```ts
import { CONTINUE_CONVERSATION_PROMPT } from "./sample-calls";
...

export const Uneditable: StoryObj = ...;
Uneditable.args = {
  prompt: CONTINUE_CONVERSATION_PROMPT,
};

export const Editable: StoryObj = ...;
Editable.args = {
  ...
  prompt: CONTINUE_CONVERSATION_PROMPT,
};
```

Similarly, `src-svelte/src/routes/database/api-calls/new/ApiCallEditor.stories.ts`, which also depends on the same prompt, gets edited too. In this file, we also realize we haven't been using the `EDIT_CANONICAL_REF` to cut down on redundancy:

```ts
import { CONTINUE_CONVERSATION_PROMPT } from "../[slug]/sample-calls";
import { EDIT_CANONICAL_REF, ... } from "./test.data";

...
EditContinuedConversation.parameters = {
  stores: {
    apiCallEditing: {
      canonicalRef: EDIT_CANONICAL_REF,
      prompt: CONTINUE_CONVERSATION_PROMPT,
    },
  },
};

...
Busy.parameters = {
  stores: {
    apiCallEditing: {
      canonicalRef: EDIT_CANONICAL_REF,
      prompt: CONTINUE_CONVERSATION_PROMPT,
    },
  },
};
```

In the end, the neat per-commit Git renames don't carry over to the overall PR diff on GitHub, but at least the history is clean and legible.

It turns out this doesn't quite work because as of now, `ApiCall` does not include actions, but `UnloadedApiCall` does. As such, our screenshot tests fail on CI.

We rename the existing `src-svelte/src/routes/database/api-calls/[slug]/ApiCall.svelte` back to `src-svelte/src/routes/database/api-calls/[slug]/ApiCallDisplay.svelte`. However, we keep all the imports the same because we actually create `src-svelte/src/routes/database/api-calls/[slug]/ApiCall.svelte` again, but this time encapsulating most of the rest of the logic of the unloaded component:

```svelte
<script lang="ts">
  import ApiCallDisplay from "./ApiCallDisplay.svelte";
  import { type LlmCall } from "$lib/bindings";
  import Actions from "./Actions.svelte";

  export let apiCall: LlmCall;
  export let showActions = true;
</script>

<div class="container">
  <ApiCallDisplay {...$$restProps} bind:apiCall />
  {#if showActions}
    <Actions {apiCall} />
  {/if}
</div>

<style>
  .container {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }
</style>

```

The unloaded component at `src-svelte/src/routes/database/api-calls/[slug]/UnloadedApiCall.svelte` is left with just simple loading code, along with using our own proper `Loading` component instead of Copilot's suggested ad-hoc loading HTML. This is the sort of project-specific knowledge that would be good for ZAMM to encapsulate someday, but it is unclear how the relevant information could be retrieved around which components are or are not standardized within the current project.

```svelte
<script lang="ts">
  import ApiCall from "./ApiCall.svelte";
  import { type LlmCall, commands } from "$lib/bindings";
  import { unwrap } from "$lib/tauri";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";
  import Loading from "$lib/Loading.svelte";

  export let id: string;
  let apiCall: LlmCall | undefined = undefined;

  unwrap(commands.getApiCall(id))
    ...;
</script>

{#if apiCall}
  <ApiCall {...$$restProps} {apiCall} />
{:else}
  <Loading />
{/if}

```

We get a TypeScript error in VS Code

```
Module './ApiCall.svelte' was resolved to '/Users/amos/Documents/zamm/src-svelte/src/routes/database/api-calls/[slug]/ApiCall.d.svelte.ts', but '--allowArbitraryExtensions' is not set.
```

But the Svelte build succeeds. After it succeeds, the error goes away in VS Code as well.

We use the new prop in `src-svelte/src/routes/database/api-calls/[slug]/ApiCall.stories.ts`:

```ts
export const Khmer: StoryObj = ...;
Khmer.args = {
  ...,
  showActions: false,
  ...
};

...

export const LotsOfCode: StoryObj = ...;
LotsOfCode.args = {
  ...
  showActions: false,
  ...
};

```

We repeat this for all of the stories in `src-svelte/src/routes/database/api-calls/[slug]/UnloadedApiCall.stories.ts`.

### Refactoring the slug page

The `TerminalSession` component is starting to take in too many props for comfort. While changing the frontend, we realize that we also need the backend to change. We edit `src-tauri/src/commands/terminal/run.rs` to remove `RunCommandResponse`. In the process, we realize that we didn't even need to retrieve the `command` variable from the AsciiCast header, because it was available in the function arguments all along. Also in the process, the `serde::Serialize` trait is no longer imported, and therefore must be specified explicitly in the tests section.

```rs
async fn run_command_helper(
    ...
) -> ZammResult<RecoveredTerminalSession> {
    ...
    let os = get_os();

    if let Some(conn) = db.as_mut() {
        diesel::insert_into(asciicasts::table)
            .values(NewAsciiCast {
                ...
                command,
                os,
                ...
            })
            ...;
    }

    let output = ...;
    Ok(RecoveredTerminalSession {
        ...,
        os,
        command: command.to_string(),
        is_active: true,
    })
}

#[tauri::command(async)]
#[specta]
pub async fn run_command(
    ...
) -> ZammResult<RecoveredTerminalSession> {
    ...
}

#[cfg(test)]
mod tests {
    ...

    fn to_yaml_string<T: serde::Serialize>(...) -> String {
        ...
    }

    fn parse_response(response_str: &str) -> RecoveredTerminalSession {
        ...
    }

    // ... rest of tests also have RunCommandResponse replaced by RecoveredTerminalSession ...
}
```

We need to make all fields public in `src-tauri/src/commands/terminal/get_session.rs`:

```rs
#[derive(...)]
pub struct RecoveredTerminalSession {
    pub id: EntityId,
    pub timestamp: NaiveDateTime,
    pub command: String,
    pub os: Option<OS>,
    pub output: String,
    pub is_active: bool,
}
```

`src-tauri/api/sample-calls/run_command-bash.yaml` are now populated with new fields like this:

```yaml
...
response:
  message: >
    {
      ...,
      "command": "bash",
      "os": "Mac",
      "output": "The default interactive ...",
      "is_active": true
    }
...
```

`src-tauri/api/sample-calls/run_command-cmd.yaml` and `src-tauri/api/sample-calls/run_command-date.yaml` are updated similarly.

We export bindings as usual, and edit `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte` to render the terminal session from a single `session` variable:

```svelte
<script lang="ts">
  ...
  import { ..., type RecoveredTerminalSession } from "$lib/bindings";
  ...

  export let session: RecoveredTerminalSession | undefined = undefined;
  ...
  $: awaitingSession = session === undefined;
  ...

  async function sendCommand(newInput: string) {
    try {
      ...
      if (session === undefined) {
        session = await unwrap(...);
      } else {
        let result = await unwrap(
          commands.sendCommandInput(session.id, ...),
        );
        session.output += result;
      }
      ...
    } ...
  }
</script>

<InfoBox title="Terminal Session" ...>
  <div class="terminal-container ...">
    {#if session?.command}
      <p class="atomic-reveal">
        Command: <span class="command">{session.command}</span>
      </p>
    {:else}
      ...
    {/if}

    <Scrollable ...>
      <pre>{session?.output ?? ""}</pre>
    </Scrollable>

    {#if session === undefined || session.is_active}
      ...
    {/if}
  </div>
</div>
```

We also edit `src-svelte/src/routes/database/terminal-sessions/UnloadedTerminalSession.svelte` to pass in the right prop:

```svelte
<script lang="ts">
  ...
  let session: RecoveredTerminalSession | undefined = undefined;

  onMount(async () => {
    try {
      session = await unwrap(...);
    } ...
  });
</script>

{#if session}
  <TerminalSession {session} />
{:else}
  ...
{/if}
```

Finally, because `RecoveredTerminalSession` is no longer a good name for the data structure if it's also used for active sessions, we rename it to `TerminalSessionInfo` and move it to `src-tauri/src/commands/terminal/models.rs`. We do the refactor on the backend, export the bindings, and update the frontend as well.

### Returning to terminal session when navigating back to tab

We make it so that when the user navigates to a different tab and back, the terminal session is preserved (or, in reality, the backend returns the session data back to the frontend to re-render). However, we realize that we also need to tell the sidebar to update its icon location, or else when the user navigates back, they will still encounter a new terminal session page. Of [the options](https://stackoverflow.com/a/71905476) presented, we try out a new method of using Svelte contexts. We try to follow [this example](https://svelte.dev/tutorial/svelte/context-api) to create a way for other components to notify the sidebar that the URL has changed.

We realize that `src-svelte/src/routes/SidebarUI.svelte` has two separate functions with the name `updateIndicator`, one of which is in `onMount`. We rename the one in `onMount` to `updateIndicatorPosition` because that more accurately describes its purpose:

```ts
    const updateIndicatorPosition = () => {
      transitionDuration = "0";
      indicatorPosition = getIndicatorPosition(getMatchingRoute(currentRoute));
      setTimeout(() => {
        transitionDuration = REGULAR_TRANSITION_DURATION;
      }, 10);
    };
```

We see

```
[HMR][Svelte]"Unrecoverable HMR error in <Root>: next update will trigger a full reload"

[Error] Unhandled Promise Rejection: Error: Function called outside component initialization

Avoid using `history.pushState(...)` and `history.replaceState(...)` as these will conflict with SvelteKit's router. Use the `pushState` and `replaceState` imports from `$app/navigation` instead.

undefined is not an object (evaluating 'sidebarContext.updateIndicator')
```

We tackle each of these.

```
[HMR][Svelte]"Unrecoverable HMR error in <Root>: next update will trigger a full reload"

[Error] Unhandled Promise Rejection: Error: Function called outside component initialization
```

It turns out that what `Function called outside component initialization` means is calling it from `onMount` instead of outside of it. We make sure to do that instead.

```
Avoid using `history.pushState(...)` and `history.replaceState(...)` as these will conflict with SvelteKit's router. Use the `pushState` and `replaceState` imports from `$app/navigation` instead.
```

We import `replaceState` from `$app/navigation` and use that as suggested.

```
undefined is not an object (evaluating 'sidebarContext.updateIndicator')
```

It turns out Svelte contexts [only work for children](https://stackoverflow.com/a/69070541). As such, we modify a store instead.

It is unclear how we test this. Trying to edit `src-svelte/src/routes/database/terminal-sessions/TerminalSession.test.ts` as such:

```ts
describe("Terminal session", () => {
  ...

  beforeEach(() => {
    ...
    window.history.replaceState = vi.fn();
    ...
  });

  ...
});
```

only gives us

```
stderr | src/routes/database/terminal-sessions/TerminalSession.test.ts > Terminal session > can start and send input to command
__vite_ssr_import_10__.replaceState is not a function
```

We find [this gist](https://gist.github.com/tkrotoff/52f4a29e919445d6e97f9a9e44ada449), but unfortunately it gives us the same error.

Upon further investigation, we find that this error is actually not due to the test code, but due to calling `replaceState` from `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte` itself. We work around this by doing a null-check first:

```ts
        if (replaceState) {
          replaceState(newUrl, $page.state);
        } else {
          window.history.replaceState($page.state, "", newUrl);
        }
```

We would want to notify the page transition page of the change too, so that it does not play a page transition animation when the user goes back to the terminal session that they had just created.

Our final solution looks like this:

The sidebar page at `src-svelte/src/routes/SidebarUI.svelte` exports a store:

```svelte
<script lang="ts" context="module">
  import { writable } from "svelte/store";

  export interface SidebarContext {
    updateIndicator: (newRoute: string) => void;
  }

  export const sidebar = writable<SidebarContext | null>(null);
</script>

<script lang="ts">

<script lang="ts">
  ...

  onMount(() => {
    ...
    sidebar.set({ updateIndicator });
    ...
  });

  ...
</script>
```

The page transition page at `src-svelte/src/routes/PageTransition.svelte` exports a page transition store too:

```svelte
<script lang="ts" context="module">
  import { writable } from "svelte/store";
  ...

  export interface PageTransitionContext {
    addVisitedRoute: (newRoute: string) => void;
  }
  export const pageTransition = writable<PageTransitionContext | null>(null);
</script>

<script lang="ts">
  ...

  onMount(async () => {
    ...
    pageTransition.set({
      addVisitedRoute: (newRoute: string) => {
        visitedKeys.add(newRoute);
      },
    });
  });
  ...
</script>
```

We make use of these stores in `src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte`:

```ts
  ...
  import { sidebar } from "../../SidebarUI.svelte";
  import { replaceState } from "$app/navigation";
  import { page } from "$app/stores";
  import { pageTransition } from "../../PageTransition.svelte";

  ...

  async function sendCommand(newInput: string) {
    try {
      ...
      if (session === undefined) {
        session = await unwrap(...);
        const newUrl = `/database/terminal-sessions/${session.id}/`;
        if (replaceState) {
          // replaceState undefined in Vitest
          replaceState(newUrl, $page.state);
        } else {
          window.history.replaceState($page.state, "", newUrl);
        }
        $sidebar?.updateIndicator(newUrl);
        $pageTransition?.addVisitedRoute(newUrl);
      }
    } ...
  }
```

We finall get the tests to work with `src-svelte/src/routes/database/terminal-sessions/TerminalSession.test.ts` by checking that the relevant functions have been invoked after the user's interactions with the terminal session:

```ts
...
import { sidebar } from "../../SidebarUI.svelte";
import { pageTransition } from "../../PageTransition.svelte";
import { get } from "svelte/store";

describe("Terminal session", () => {
  ...

  beforeEach(() => {
    ...
    window.history.replaceState = vi.fn();
    sidebar.set({ updateIndicator: vi.fn() });
    pageTransition.set({ addVisitedRoute: vi.fn() });
    ...
  });

  ...

  test("can start and send input to command", async () => {
    ...
    await userEvent.type(...);
    await userEvent.click(...);
    expect(tauriInvokeMock).toHaveReturnedTimes(...);
    await waitFor(() => {
      expect(...).toBeInTheDocument();
    });
    expect(window.history.replaceState).toHaveBeenCalledWith(
      undefined,
      "",
      "/database/terminal-sessions/3717ed48-ab52-4654-9f33-de5797af5118/",
    );
    expect(get(sidebar)?.updateIndicator).toHaveBeenCalledWith(
      "/database/terminal-sessions/3717ed48-ab52-4654-9f33-de5797af5118/",
    );
    expect(get(pageTransition)?.addVisitedRoute).toHaveBeenCalledWith(
      "/database/terminal-sessions/3717ed48-ab52-4654-9f33-de5797af5118/",
    );
    ...
  });

  ...
});

```

### CI errors

We get the error

```
node:internal/event_target:1054
  process.nextTick(() => { throw err; });
                           ^
Error: The following routes were marked as prerenderable, but were not prerendered because they were not found while crawling your app:
  - /database/terminal-sessions/[slug]

See https://kit.svelte.dev/docs/page-options#prerender-troubleshooting for info on how to solve this
    at prerender (file:///__w/zamm/zamm/node_modules/@sveltejs/kit/src/core/postbuild/prerender.js:495:9)
    at async MessagePort.<anonymous> (file:///__w/zamm/zamm/node_modules/@sveltejs/kit/src/utils/fork.js:22:16)
Emitted 'error' event on Worker instance at:
    at [kOnErrorMessage] (node:internal/worker:326:10)
```

As such, we edit `src-svelte/svelte.config.js`:

```js
/** @type {import('@sveltejs/kit').Config} */
const config = {
  ...

  kit: {
    ...
    prerender: {
      ...
      entries: [
        ...,
        "/database/terminal-sessions/[slug]",
      ],
    },
  },
};
```

We also see these errors in the e2e tests:

```
[wry 0.46.3 linux #0-0] 2) App should allow navigation to the new terminal session page
[wry 0.46.3 linux #0-0] element ("a=start") still not clickable after 5000ms
[wry 0.46.3 linux #0-0] Error: element ("a=start") still not clickable after 5000ms
[wry 0.46.3 linux #0-0]     at async findAndClick (e2e.test.js:12:3)
[wry 0.46.3 linux #0-0]     at async Context.<anonymous> (e2e.test.js:117:5)
[wry 0.46.3 linux #0-0]
[wry 0.46.3 linux #0-0] 3) App should successfully interact with the terminal
[wry 0.46.3 linux #0-0] element ("a=start") still not clickable after 5000ms
[wry 0.46.3 linux #0-0] Error: element ("a=start") still not clickable after 5000ms
[wry 0.46.3 linux #0-0]     at async findAndClick (e2e.test.js:12:3)
[wry 0.46.3 linux #0-0]     at async Context.<anonymous> (e2e.test.js:131:5)
```

This is because we've changed the wording on that screen. We edit `webdriver/test/specs/e2e.test.js` to update the selectors:

```js
  it("should allow navigation to the new terminal session page", async function () {
    ...
    await findAndClick("a=started");
    ...
  });

  it("should successfully interact with the terminal", async function () {
    ...
    await findAndClick("a=started");
    ...
  });
```

The second test still fails on CI for some reason, but it is not reproducible on our own local Linux machine. We try taking a screenshot of the page right before the failure, to debug:

```js
  it("should successfully interact with the terminal", async function () {
    ...
    expect(
      await browser.checkFullPageScreen("running-terminal-session", {}),
    ).toBeLessThanOrEqual(maxMismatch);
    await findAndClick("a=started");
    ...
  });
```

The "started" link appears in the screenshot, as expected. We try adding a `await browser.pause(500);` instead. This still doesn't work, as it turns out the default waits for 5,000 ms already on the selector itself. This error doesn't pop up for manual testing either. We try to instead click on the + button on that page:

```js
await findAndClick('a[title="New Terminal Session"]');
```

This finally works.

We also get

```
⎯⎯⎯⎯⎯ Uncaught Exception ⎯⎯⎯⎯⎯
TypeError: Cannot read properties of null (reading 'classList')
 ❯ src/lib/FixedScrollable.svelte:35:16
     33|       let indicator = entries[0];
     34|       if (indicator.isIntersecting) {
     35|         shadow.classList.remove("visible");
       |                ^
     36|       } else {
     37|         shadow.classList.add("visible");
 ❯ src/lib/FixedScrollable.svelte:55:42
 ❯ Timeout._onTimeout src/routes/database/DatabaseView.test.ts:23:7
 ❯ listOnTimeout node:internal/timers:573:17
 ❯ processTimers node:internal/timers:514:7

This error originated in "src/routes/database/DatabaseView.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
```

This appears to be perhaps similar to the `"scrollable.getDimensions() is not a function"` error we handled above, as sometimes the test locally succeeds with that and sometimes it fails with this. As such, we edit `src-svelte/src/lib/FixedScrollable.svelte` to introduce a new guard:

```ts
  function intersectionCallback(shadow: HTMLDivElement) {
    return (entries: IntersectionObserverEntry[]) => {
      if (!shadow) {
        console.warn("Shadow not mounted");
        return;
      }

      ...
    };
  }
```
