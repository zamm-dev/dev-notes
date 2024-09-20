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
