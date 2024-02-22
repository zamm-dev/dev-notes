# Test infrastructure refactors

## Setting up the ZAMM DB shared variable

We have multiple copies of `setup_zamm_db` running around. We edit `src-tauri\src\test_helpers.rs` to include a canonical version of this:

```rs
use crate::setup::db::MIGRATIONS;
use crate::ZammDatabase;
use diesel::prelude::*;
use diesel_migrations::MigrationHarness;
...
use tokio::sync::Mutex;

...

pub fn setup_database() -> SqliteConnection {
    let mut conn = SqliteConnection::establish(":memory:").unwrap();
    conn.run_pending_migrations(MIGRATIONS).unwrap();
    conn
}

pub fn setup_zamm_db() -> ZammDatabase {
    ZammDatabase(Mutex::new(Some(setup_database())))
}

```

We now edit every place that we use these functions, removing their definitions and importing it from `crate::test_helpers` instead. We start with `src-tauri\src\commands\keys\set.rs`:

```rs
#[cfg(test)]
pub mod tests {
    ...
    use crate::test_helpers::{get_temp_test_dir, setup_database, setup_zamm_db};
    ...
}
```

We do the same in `src-tauri\src\commands\keys\mod.rs`, where we no longer import it from `set::tests`:

```rs
#[cfg(test)]
mod tests {
    ...
    use crate::test_helpers::setup_zamm_db;
    ...
    use set::tests::check_set_api_key_sample;
    ...
}
```

We finally do this in `src-tauri\src\commands\llms\chat.rs`, once again removing the local definition of those functions there:

```rs
#[cfg(test)]
mod tests {
    ...
    use crate::test_helpers::setup_zamm_db;
    ...
}
```

We check that all our tests still pass, and commit.

## Test helpers module

We split up `src-tauri/src/test_helpers.rs` into `src-tauri\src\test_helpers\mod.rs`:

```rs
pub mod database;
pub mod temp_files;

pub use database::{setup_database, setup_zamm_db};
pub use temp_files::get_temp_test_dir;

```

and `src-tauri\src\test_helpers\temp_files.rs`:

```rs
use std::env;
use std::fs;
use std::path::PathBuf;

pub fn get_temp_test_dir(test_name: &str) -> PathBuf {
    let mut test_dir = env::temp_dir();
    test_dir.push("zamm/tests");
    test_dir.push(test_name);
    if test_dir.exists() {
        fs::remove_dir_all(&test_dir).unwrap_or_else(|_| {
            panic!("Can't remove temp test dir at {}", test_dir.display())
        });
    }
    fs::create_dir_all(&test_dir).unwrap_or_else(|_| {
        panic!("Can't create temp test dir at {}", test_dir.display())
    });
    test_dir
}

```

and `src-tauri\src\test_helpers\database.rs`:

```rs
use crate::setup::db::MIGRATIONS;
use crate::ZammDatabase;
use diesel::prelude::*;
use diesel_migrations::MigrationHarness;
use tokio::sync::Mutex;

pub fn setup_database() -> SqliteConnection {
    let mut conn = SqliteConnection::establish(":memory:").unwrap();
    conn.run_pending_migrations(MIGRATIONS).unwrap();
    conn
}

pub fn setup_zamm_db() -> ZammDatabase {
    ZammDatabase(Mutex::new(Some(setup_database())))
}

```

## Sample calls refactor

There's a lot of copied code for testing sample calls. We'll do this by refactoring things little by little. We start by at least refactoring the API call check and request parsing into `src-tauri\src\test_helpers\api_testing.rs`:

```rs
use crate::sample_call::SampleCall;
use serde::{Deserialize, Serialize};
use std::fs;

fn read_sample(filename: &str) -> SampleCall {
    let sample_str = fs::read_to_string(filename)
        .unwrap_or_else(|_| panic!("No file found at {filename}"));
    serde_yaml::from_str(&sample_str).unwrap()
}

pub struct SampleCallResult<T>
where
    T: Serialize + for<'de> Deserialize<'de>,
{
    pub sample: SampleCall,
    pub args: Option<T>,
}

pub trait SampleCallTestCase<T>
where
    T: Serialize + for<'de> Deserialize<'de>,
{
    const EXPECTED_API_CALL: &'static str;
    const CALL_HAS_ARGS: bool;

    fn parse_request(&self, request_str: &str) -> T {
        serde_json::from_str(request_str).unwrap()
    }

    fn check_sample_call(&self, sample_file: &str) -> SampleCallResult<T> {
        let sample = read_sample(sample_file);
        if Self::CALL_HAS_ARGS {
            assert_eq!(sample.request.len(), 2);
        } else {
            assert_eq!(sample.request.len(), 1);
        }
        assert_eq!(sample.request[0], Self::EXPECTED_API_CALL);

        let request = if Self::CALL_HAS_ARGS {
            Some(self.parse_request(&sample.request[1]))
        } else {
            None
        };
        SampleCallResult {
            sample,
            args: request,
        }
    }
}

```

and exporting that from `src-tauri\src\test_helpers\mod.rs`:

```rs
pub mod api_testing;
...

pub use api_testing::SampleCallTestCase;
...
```

We edit files such as `src-tauri\src\commands\keys\get.rs` to use this new sample testing API:

```rs
#[cfg(test)]
pub mod tests {
    ...
    use crate::test_helpers::SampleCallTestCase;
    ...

    struct GetApiKeysTestCase {
        // pass
    }

    impl SampleCallTestCase<()> for GetApiKeysTestCase {
        const EXPECTED_API_CALL: &'static str = "get_api_keys";
        const CALL_HAS_ARGS: bool = false;
    }

    pub async fn check_get_api_keys_sample(
        file_prefix: &str,
        rust_input: &ZammApiKeys,
    ) {
        let test_case = GetApiKeysTestCase {};
        let result = test_case.check_sample_call(file_prefix);

        ...
        let expected_json = result.sample.response.message.trim();
        ...
    }

    ...
}
```

or `src-tauri\src\commands\keys\set.rs`:

```rs
#[cfg(test)]
pub mod tests {
    ...
    use crate::test_helpers::{
        get_temp_test_dir, setup_database, setup_zamm_db, SampleCallTestCase,
    };
    ...

    struct SetApiKeyTestCase {
        // pass
    }

    impl SampleCallTestCase<SetApiKeyRequest> for SetApiKeyTestCase {
        const EXPECTED_API_CALL: &'static str = "set_api_key";
        const CALL_HAS_ARGS: bool = true;
    }

    ...

    pub async fn check_set_api_key_sample(
        ...
    ) {
        let test_case = SetApiKeyTestCase {};
        let result = test_case.check_sample_call(sample_file);
        let request = result.args.unwrap();

        ...
        if result.sample.response.success == Some(false) {
            ...
        } ...

        let expected_json = result.sample.response.message.trim();
        ...
    }

    ...
}
```

We do this for all of the other tests, and make sure that everything still passes before committing.

Next, we rename `request` to `args` and try to refactor out the part where we call the function. At first, we get the error

```
error[E0277]: the trait bound `errors::Error: sample_call::_::_serde::Deserialize<'_>` is not satisfied
   --> src\commands\llms\chat.rs:257:61
    |
257 | ..._call(sample_path).await;
    |                       ^^^^^ the trait `sample_call::_::_serde::Deserialize<'_>` is not implemented for `errors::Error`
    |
    = help: the following other types implement trait `sample_call::_::_serde::Deserialize<'de>`:
              <bool as sample_call::_::_serde::Deserialize<'de>>    
              <char as sample_call::_::_serde::Deserialize<'de>>    
              <isize as sample_call::_::_serde::Deserialize<'de>>   
              <i8 as sample_call::_::_serde::Deserialize<'de>>      
              <i16 as sample_call::_::_serde::Deserialize<'de>>     
              <i32 as sample_call::_::_serde::Deserialize<'de>>     
              <i64 as sample_call::_::_serde::Deserialize<'de>>     
              <i128 as sample_call::_::_serde::Deserialize<'de>>    
            and 456 others
    = note: required for `Result<models::llm_calls::LlmCall, errors::Error>` to implement `for<'de> sample_call::_::_serde::Deserialize<'de>`
note: required by a bound in `SampleCallResult`
   --> src\test_helpers\api_testing.rs:14:20
    |
11  | pub struct SampleCallResult<T, U>
    |            ---------------- required by a bound in this struct
...
14  |     U: Serialize + for<'de> Deserialize<'de>,
    |                    ^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `SampleCallResult`
```

We realize that this is because we have marked the output type as supporting deserialization, but deserialization is not implemented for our error types. As such, we fix this by removing the requirement for output deserialization.

Then for code such as

```rs
    pub async fn check_get_api_keys_sample(
        file_prefix: &str,
        rust_input: &ZammApiKeys,
    ) {
        let test_case = GetApiKeysTestCase {
            api_keys: rust_input,
        };
        let call = test_case.check_sample_call(file_prefix).await;
        let actual_json = serde_json::to_string_pretty(&call.result).unwrap();
        let expected_json = call.sample.response.message.trim();
        assert_eq!(actual_json, expected_json);
    }
```

we get the error

```
error[E0521]: borrowed data escapes outside of function
  --> src\commands\keys\get.rs:44:20
   |
39 | ...   rust_input: &ZammApiKeys,
   |       ----------  - let's call the lifetime of this reference `'1`
   |       |
   |       `rust_input` is a reference that is only valid in the function body
...
44 | ...   let call = test_case.check_sample_call(file_prefix).aw...
   |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^      
   |                  |
   |                  `rust_input` escapes the function body here   
   |                  argument requires that `'1` must outlive `'static`
```

We fix this by explicitly declaring several lifetimes to be the same lifetime `'a`:

```rs
    pub async fn check_get_api_keys_sample<'a>(
        file_prefix: &str,
        rust_input: &'a ZammApiKeys,
    ) {
        let mut test_case = GetApiKeysTestCase<'a> {
            api_keys: rust_input,
        };
        ...
    }
```

But then later we get

```
error: comparison operators cannot be chained
  --> src\commands\keys\get.rs:41:47
   |
41 |         let mut test_case = GetApiKeysTestCase<'a> {
   |                                               ^  ^
```

As such, we remove the `<'a>` when constructing the test case.

We finally get all tests compiling and passing. Our edits to `src-tauri\src\test_helpers\api_testing.rs` are:

```rs
pub struct SampleCallResult<T, U>
where
    T: ...,
    U: Serialize,
{
    ...,
    pub result: U,
}

pub trait SampleCallTestCase<T, U>
where
    T: ...,
    U: Serialize,
{
    ...

    fn parse_args(&self, request_str: &str) -> T {
        ...
    }

    async fn make_request(&mut self, args: &Option<T>) -> U;

    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...
        // make the call
        let args = if Self::CALL_HAS_ARGS {
            Some(self.parse_args(&sample.request[1]))
        } else {
            None
        };
        let result = self.make_request(&args).await;

        SampleCallResult {
            ...,
            result,
        }
    }
}
```

Some files are relatively easy to adapt to this new API, for example `src-tauri\src\commands\keys\get.rs`:

```rs
#[cfg(test)]
pub mod tests {
    ...

    struct GetApiKeysTestCase<'a> {
        api_keys: &'a ZammApiKeys,
    }

    impl<'a> SampleCallTestCase<(), ZammResult<ApiKeys>> for GetApiKeysTestCase<'a> {
        ...

        async fn make_request(&mut self, _args: &Option<()>) -> ZammResult<ApiKeys> {
            Ok(get_api_keys_helper(&self.api_keys).await)
        }
    }

    pub async fn check_get_api_keys_sample<'a>(
        file_prefix: &str,
        rust_input: &'a ZammApiKeys,
    ) {
        let mut test_case = GetApiKeysTestCase {
            api_keys: rust_input,
        };
        let call = test_case.check_sample_call(file_prefix).await;
        let actual_json = serde_json::to_string_pretty(&call.result.unwrap()).unwrap();
        let expected_json = call.sample.response.message.trim();
        assert_eq!(actual_json, expected_json);
    }

    ...
}
```

(Every single sample call test now needs a `#[tokio::test]` marker and `async` and `await` on the call to `check_sample_call`.)

Some other files are harder to adapt, most notably `src-tauri\src\commands\keys\set.rs`:

```rs
#[cfg(test)]
pub mod tests {
    ...

    struct SetApiKeyTestCase<'a> {
        api_keys: &'a ZammApiKeys,
        db: &'a ZammDatabase,
        test_dir_name: &'a str,
        valid_request_path_specified: Option<bool>,
        request_path: Option<PathBuf>,
        test_init_file: Option<String>,
    }

    impl<'a> SampleCallTestCase<SetApiKeyRequest, ZammResult<()>>
        for SetApiKeyTestCase<'a>
    {
        ...

        async fn make_request(
            &mut self,
            args: &Option<SetApiKeyRequest>,
        ) -> ZammResult<()> {
            let request = args.as_ref().unwrap();
            let valid_request_path_specified = request
                ...;
            ...

            self.valid_request_path_specified = Some(valid_request_path_specified);
            self.request_path = request_path;
            self.test_init_file = test_init_file;

            set_api_key_helper(
                self.api_keys,
                self.db,
                self.test_init_file.as_deref(),
                &request.service,
                request.api_key.clone(),
            )
            .await
        }
    }

    ...

    pub async fn check_set_api_key_sample<'a>(
        db: &'a ZammDatabase,
        sample_file: &str,
        existing_zamm_api_keys: &'a ZammApiKeys,
        test_dir_name: &'a str,
        json_replacements: HashMap<String, String>,
    ) {
        let mut test_case = SetApiKeyTestCase {
            api_keys: existing_zamm_api_keys,
            db,
            test_dir_name,
            valid_request_path_specified: None,
            request_path: None,
            test_init_file: None,
        };
        let call = test_case.check_sample_call(sample_file).await;
        let request = call.args.unwrap();

        // check that the API call returns the expected success or failure signal
        if call.sample.response.success == Some(false) {
            ...
        } ...

        // check that the API call returns the expected JSON
        let actual_json = match call.result {
            ...
        };

        ...

        if test_case.valid_request_path_specified.unwrap() {
            let p = test_case.request_path.unwrap();
            ...
        }
    }

    ...
}
```

Next, we will check that the returned results look correct. We edit `src-tauri\src\test_helpers\api_testing.rs` as such:

```rs
use crate::commands::ZammResult;
...

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    fn serialize_result(&self, sample: &SampleCall, result: &U) -> String;

    fn check_result(&self, sample: &SampleCall, result: &U);

    async fn check_sample_call(&mut self, ...) -> SampleCallResult<T, U> {
        ...
        // check the call against sample outputs
        let actual_json = self.serialize_result(&sample, &result);
        let expected_json = sample.response.message.trim();
        assert_eq!(actual_json, expected_json);
        self.check_result(&sample, &result);
        ...
    }
}

pub trait DirectReturn<U>
where
    U: Serialize,
{
    fn serialize_result(&self, _sample: &SampleCall, result: &U) -> String {
        serde_json::to_string_pretty(result).unwrap()
    }

    fn check_result(&self, _sample: &SampleCall, _result: &U) {}
}

pub fn serialize_zamm_result<T>(result: &ZammResult<T>) -> String
where
    T: Serialize,
{
    match result {
        Ok(r) => serde_json::to_string_pretty(&r).unwrap(),
        Err(e) => serde_json::to_string_pretty(&e).unwrap(),
    }
}

pub trait ZammResultReturn<U>
where
    U: Serialize + std::fmt::Debug,
{
    fn serialize_result(&self, _sample: &SampleCall, result: &ZammResult<U>) -> String {
        serialize_zamm_result(result)
    }

    fn check_result(&self, sample: &SampleCall, result: &ZammResult<U>) {
        if sample.response.success == Some(false) {
            assert!(result.is_err(), "API call should have thrown error");
        } else {
            assert!(result.is_ok(), "API call failed: {:?}", result);
        }
    }
}

```

We see from [this discussion](https://users.rust-lang.org/t/multiple-default-implementations-of-a-trait/12271) that this appears to be the best we can do without using a macro.

Finally now, some test cases are completely refactored away into that common code. For example, `src-tauri\src\commands\system.rs`:

```rs
#[cfg(test)]
mod tests {
    ...
    use crate::sample_call::SampleCall;
    use crate::test_helpers::{DirectReturn, SampleCallTestCase};
    ...

    impl SampleCallTestCase<(), SystemInfo> for GetSystemInfoTestCase {
        ...

        fn serialize_result(&self, sample: &SampleCall, result: &SystemInfo) -> String {
            DirectReturn::serialize_result(self, sample, result)
        }

        fn check_result(&self, sample: &SampleCall, result: &SystemInfo) {
            DirectReturn::check_result(self, sample, result)
        }
    }

    impl DirectReturn<SystemInfo> for GetSystemInfoTestCase {}

    async fn check_get_system_info_sample(file_prefix: &str, actual_info: &SystemInfo) {
        let mut test_case = GetSystemInfoTestCase {
            system_info: actual_info.clone(),
        };
        test_case.check_sample_call(file_prefix).await;
    }

    ...
}
```

In fact, for this file, we can now get rid of `check_get_system_info_sample` entirely, and just directly do

```rs
    #[tokio::test]
    async fn test_get_linux_system_info() {
        let mut test_case = GetSystemInfoTestCase {
            system_info: SystemInfo {
                zamm_version: "0.0.0".to_string(),
                os: Some(OS::Linux),
                shell: Some(Shell::Zsh),
                shell_init_file: Some("/root/.zshrc".to_string()),
            },
        };
        test_case
            .check_sample_call("./api/sample-calls/get_system_info-linux.yaml")
            .await;
    }
```

Some other files are a bit harder to do this for. For example, for `src-tauri\src\commands\llms\chat.rs` we have this custom logic:

```rs
    struct SetApiKeyTestCase<'a> {
        ...,
        json_replacements: HashMap<String, String>,
        ...
    }

    ...

    impl ZammResultReturn<LlmCall> for ChatTestCase {
        fn serialize_result(
            &self,
            sample: &SampleCall,
            result: &ZammResult<LlmCall>,
        ) -> String {
            let expected_llm_call = parse_response(&sample.response.message);
            // swap out non-deterministic parts before JSON comparison
            let deterministic_llm_call = LlmCall {
                id: expected_llm_call.id,
                timestamp: expected_llm_call.timestamp,
                ..result.as_ref().unwrap().clone()
            };
            serde_json::to_string_pretty(&deterministic_llm_call).unwrap()
        }
    }

    ...

    pub async fn check_set_api_key_sample<'a>(
        ...,
        json_replacements: HashMap<String, String>,
    ) {
        let mut test_case = SetApiKeyTestCase {
            ...,
            json_replacements,
            ...
        };
        ...
    }
```

and for `src-tauri\src\commands\keys\set.rs`:

```rs
    impl<'a> ZammResultReturn<()> for SetApiKeyTestCase<'a> {
        fn serialize_result(
            &self,
            _sample: &SampleCall,
            result: &ZammResult<()>,
        ) -> String {
            let actual_json = serialize_zamm_result(result);
            self.json_replacements
                .iter()
                .fold(actual_json, |acc, (k, v)| acc.replace(k, v))
        }
    }
```

We now realize that we can actually put the logic for checking whether the API keys were successfully set or not into the `check_result` function -- at least, once its signature changes a little, and the signatures of adjacent classes and whatnot change to adjust as well. We edit `src-tauri\src\test_helpers\api_testing.rs`:

```rs
pub trait SampleCallTestCase<T, U>
    ...
{
    ...
    async fn check_result(&self, sample: &SampleCall, args: Option<&T>, result: &U);
    
    async fn check_sample_call(&mut self, ...) -> SampleCallResult<T, U> {
        ...
        self.check_result(&sample, args.as_ref(), &result).await;
        ...
    }
}

pub trait DirectReturn<T, U>
where
    T: Serialize + for<'de> Deserialize<'de>,
    U: Serialize,
{
    ...

    async fn check_result(&self, _sample: &SampleCall, _args: Option<&T>, _result: &U) {
    }
}

...

pub fn check_zamm_result<T>(sample: &SampleCall, result: &ZammResult<T>)
where
    T: std::fmt::Debug,
{
    if sample.response.success == Some(false) {
        assert!(result.is_err(), "API call should have thrown error");
    } else {
        assert!(result.is_ok(), "API call failed: {:?}", result);
    }
}

pub trait ZammResultReturn<T, U>
where
    T: Serialize + for<'de> Deserialize<'de>,
    U: Serialize + std::fmt::Debug,
{
    ...

    async fn check_result(
        &self,
        sample: &SampleCall,
        _args: Option<&T>,
        result: &ZammResult<U>,
    ) {
        check_zamm_result(sample, result)
    }
}

```

Changes for the other files are straightforward, except for `src-tauri\src\commands\keys\set.rs`, where we move the functionality out of `check_set_api_key_sample` into this function:

```rs
    impl<'a> SampleCallTestCase<SetApiKeyRequest, ZammResult<()>>
        for SetApiKeyTestCase<'a>
    {
        ...

        async fn check_result(
            &self,
            sample: &SampleCall,
            args: Option<&SetApiKeyRequest>,
            result: &ZammResult<()>,
        ) {
            check_zamm_result(sample, result);

            // check that the API call actually modified the in-memory API keys,
            // regardless of success or failure. check the database as well
            let existing_api_keys = &self.api_keys.0.lock().await;
            let actual_args = args.unwrap();
            if actual_args.api_key.is_empty() {
                assert_eq!(existing_api_keys.openai, None);
                assert_eq!(get_openai_api_key_from_db(self.db).await, None);
            } else {
                let arg_api_key = Some(actual_args.api_key.clone());
                assert_eq!(existing_api_keys.openai, arg_api_key);
                assert_eq!(
                    get_openai_api_key_from_db(self.db).await,
                    arg_api_key,
                );
            }
        }
    }

    ...

    pub async fn check_set_api_key_sample<'a>(
        ...
    ) {
        ...
        test_case.check_sample_call(sample_file).await;
        ...
    }
```

Next, we wish for the test infrastructure to take care of tests involving disk side effects. We edit `src-tauri\api\sample-call-schema.json` to take additional information into account in the test files:

```json
{
  "$schema": "http://json-schema.org/draft-06/schema#",
  "$ref": "#/definitions/SampleCall",
  "definitions": {
    "SampleCall": {
      ...
      "properties": {
        ...,
        "sideEffects": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "disk": {
              "type": "object",
              "additionalProperties": false,
              "properties": {
                "startStateDirectory": {
                  "type": "string"
                },
                "endStateDirectory": {
                  "type": "string"
                }
              },
              "required": ["endStateDirectory"]
            }
          }
        }
      },
      ...
    }
  }
}
```

We run

```bash
$ make quicktype
```

to update the Rust and TypeScript code for this schema. We then edit `src-tauri\src\test_helpers\api_testing.rs` to include this new information:

```rs
...
use crate::sample_call::{Disk, SampleCall};
use crate::test_helpers::temp_files::get_temp_test_dir;
...
use std::path::{Path, PathBuf};
use std::{fs, io};

...

fn copy_dir_all(src: impl AsRef<Path>, dst: impl AsRef<Path>) -> io::Result<()> {
    fs::create_dir_all(&dst)?;
    for entry in fs::read_dir(src)? {
        let entry = entry?;
        let ty = entry.file_type()?;
        if ty.is_dir() {
            copy_dir_all(entry.path(), dst.as_ref().join(entry.file_name()))?;
        } else {
            fs::copy(entry.path(), dst.as_ref().join(entry.file_name()))?;
        }
    }
    Ok(())
}

...

pub struct SideEffectsHelpers {
    pub disk: Option<PathBuf>,
}

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    fn temp_test_subdirectory(&self) -> String {
        unimplemented!()
    }

    async fn make_request(
        &mut self,
        args: &Option<T>,
        side_effects: &SideEffectsHelpers,
    ) -> U;

    ...

    fn get_temp_dir(&self) -> PathBuf {
        get_temp_test_dir(&self.temp_test_subdirectory())
    }

    fn initialize_temp_dir_inputs(&self, disk_side_effect: &Disk, temp_dir: &PathBuf) {
        if let Some(input_dir) = &disk_side_effect.start_state_directory {
            let relative_input_dir = format!("api/sample-disk-writes/{}", input_dir);
            copy_dir_all(relative_input_dir, temp_dir).unwrap();
        }
    }

    fn compare_temp_dir_outputs(
        &self,
        disk_side_effect: &Disk,
        actual_output_dir: &PathBuf,
    ) {
        let relative_expected_output_dir = format!(
            "api/sample-disk-writes/{}",
            &disk_side_effect.end_state_directory
        );
        let expected_output_dir = Path::new(&relative_expected_output_dir);
        let mut expected_output_files = vec![];
        for entry in fs::read_dir(expected_output_dir).unwrap() {
            let entry = entry.unwrap();
            expected_output_files.push(entry.file_name());
        }

        let mut actual_output_files = vec![];
        for entry in fs::read_dir(actual_output_dir).unwrap() {
            let entry = entry.unwrap();
            actual_output_files.push(entry.file_name());
        }

        assert_eq!(expected_output_files, actual_output_files);
        for file in expected_output_files {
            let expected_file = fs::read(expected_output_dir.join(&file)).unwrap();
            let actual_file = fs::read(actual_output_dir.join(&file)).unwrap();

            let expected_file_str = String::from_utf8(expected_file).unwrap();
            let actual_file_str = String::from_utf8(actual_file).unwrap();
            assert_eq!(expected_file_str, actual_file_str);
        }
    }

    async fn check_sample_call(&mut self, ...) -> SampleCallResult<T, U> {
        ...

        // prepare side-effects
        let mut temp_dir: Option<PathBuf> = None;
        if let Some(side_effects) = &sample.side_effects {
            // prepare disk if necessary
            if let Some(disk_side_effect) = &side_effects.disk {
                temp_dir = Some(self.get_temp_dir());
                self.initialize_temp_dir_inputs(
                    disk_side_effect,
                    &temp_dir.as_ref().unwrap(),
                );
                println!(
                    "Test will use temp directory at {}",
                    temp_dir.as_ref().unwrap().display()
                );
            }
        }
        let side_effects_helpers = SideEffectsHelpers { disk: temp_dir };

        // make the call
        ...
        let result = self.make_request(&args, &side_effects_helpers).await;

        ...

        // check the call against disk side-effects
        if let Some(temp_test_dir) = &side_effects_helpers.disk {
            let disk_side_effect =
                &sample.side_effects.as_ref().unwrap().disk.as_ref().unwrap();
            self.compare_temp_dir_outputs(disk_side_effect, &temp_test_dir);
        }

        ...
    }
}
```

where the `copy_dir_all` function is one we've copied from [this answer](https://stackoverflow.com/a/65192210). We edit `src-tauri\src\test_helpers\mod.rs` to expose this new struct:

```rs
pub use api_testing::{
    ..., SideEffectsHelpers, ...
};
```

We are only targeting the preferences API at this point. So for other tests such as `src-tauri\src\commands\system.rs`, the only edit is to change the function signature of `make_request` to include the unused `side_effects` parameter:

```rs
    impl SampleCallTestCase<(), SystemInfo> for GetSystemInfoTestCase {
        ...

        async fn make_request(
            &mut self,
            _: &Option<()>,
            _: &SideEffectsHelpers,
        ) -> SystemInfo {
            ...
        }

        ...
    }
```

As for preference reading and writing, we want some way to specify a unique test directory to put temporary test files into, without risking forgetting to edit the directory name during a copy-paste operation. From [this discussion](https://internals.rust-lang.org/t/discovering-the-current-test-name/15325/19), we discover that there is a [`stdext::function_name`](https://docs.rs/stdext/0.3.2/stdext/macro.function_name.html) function we can use for this purpose. We run

```bash
$ cargo add --dev stdext
```

and then edit `src-tauri\src\commands\preferences\write.rs`:

```rs
#[cfg(test)]
mod tests {
    ...
    use crate::test_helpers::{
        SampleCallTestCase, SideEffectsHelpers, ZammResultReturn,
    };
    ...
    use stdext::function_name;

    ...

    struct SetPreferencesTestCase {
        test_fn_name: &'static str,
    }

    impl SampleCallTestCase<SetPreferencesRequest, ZammResult<()>>
        for SetPreferencesTestCase
    {
        ...

        fn temp_test_subdirectory(&self) -> String {
            let test_logical_path =
                self.test_fn_name.split("::").collect::<Vec<&str>>();
            let test_name = test_logical_path[test_logical_path.len() - 2];
            format!("{}/{}", Self::EXPECTED_API_CALL, test_name)
        }

        async fn make_request(
            &mut self,
            args: &Option<SetPreferencesRequest>,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<()> {
            set_preferences_helper(
                &side_effects.disk,
                &args.as_ref().unwrap().preferences,
            )
        }

        ...
    }

    ...

    async fn check_set_preferences_sample(
        test_fn_name: &'static str,
        file_prefix: &str,
    ) {
        let mut test_case = SetPreferencesTestCase { test_fn_name };
        test_case.check_sample_call(file_prefix).await;
    }

    #[tokio::test]
    async fn test_set_preferences_sound_off_without_file() {
        check_set_preferences_sample(
            function_name!(),
            "./api/sample-calls/set_preferences-sound-off.yaml",
        )
        .await;
    }

    ...
}
```

We get the second-to-last element of `test_logical_path` because the value looks like `zamm::commands::preferences::write::tests::test_set_preferences_sound_off_without_file::{{closure}}`. If we do an `assert!(false);` to force the test to fail, we see the output

```
Test will use temp directory at C:\Users\AMOSNG~1\AppData\Local\Temp\zamm/tests\set_preferences/test_set_preferences_sound_off_without_file
```

We rename some of the sample disk files, such as `src-tauri/api/sample-settings/extra-settings/preferences.toml` to `src-tauri/api/sample-disk-writes/sound-off-extra-settings/preferences.toml`. One file in particular wasn't called `preferences.toml`, but was renamed to fit the new format: `src-tauri/api/sample-settings/extra-settings/sound-on.toml` was renamed to `src-tauri/api/sample-disk-writes/sound-on-extra-settings/preferences.toml`.

We now edit the files, such as `src-tauri\api\sample-calls\set_preferences-volume-partial.yaml`, to include the disk side effects:

```yaml
...
sideEffects:
  disk:
    endStateDirectory: volume-override
```

Because no `startStateDirectory` is specified, the test will start off with an empty directory and end up modifying the directory to have the exact same contents as `src-tauri\api\sample-disk-writes\sound-override`.

`src-tauri\api\sample-calls\set_preferences-sound-on.yaml` in particular will have both specified:

```yaml
...
sideEffects:
  disk:
    startStateDirectory: sound-off-extra-settings
    endStateDirectory: sound-on-extra-settings
```

The preference writing tests are now succeeding, but the reading ones are failing. As such, we edit `src-tauri/src/commands/preferences/read.rs`:

```rs
#[cfg(test)]
mod tests {
    ...
    use crate::test_helpers::{..., SideEffectsHelpers};
    use stdext::function_name;

    struct GetPreferencesTestCase {
        test_fn_name: &'static str,
    }

    impl SampleCallTestCase<(), Preferences> for GetPreferencesTestCase {
        ...

        fn temp_test_subdirectory(&self) -> String {
            let test_logical_path =
                self.test_fn_name.split("::").collect::<Vec<&str>>();
            let test_name = test_logical_path[test_logical_path.len() - 2];
            format!("{}/{}", Self::EXPECTED_API_CALL, test_name)
        }

        async fn make_request(
            &mut self,
            _: &Option<()>,
            side_effects: &SideEffectsHelpers,
        ) -> Preferences {
            get_preferences_helper(&side_effects.disk)
        }

        ...
    }

    impl DirectReturn<(), Preferences> for GetPreferencesTestCase {}

    async fn check_get_preferences_sample<'a>(
        test_fn_name: &'static str,
        ...
    ) {
        let mut test_case = GetPreferencesTestCase { test_fn_name };
        ...
    }

    #[tokio::test]
    async fn test_get_preferences_without_file() {
        check_get_preferences_sample(
            function_name!(),
            "./api/sample-calls/get_preferences-no-file.yaml",
        )
        .await;
    }

    ...
}
```

Note that `GetPreferencesTestCase` has simplified to the point where it doesn't even need a lifetime parameter anymore.

We create an empty `src-tauri\api\sample-disk-writes\empty\.gitkeep` to make it possible to test against empty directories to double check that no disk writes are being made. Now we can edit `src-tauri\api\sample-calls\get_preferences-no-file.yaml`:

```yaml
...
sideEffects:
  disk:
    startStateDirectory: empty
    endStateDirectory: empty
```

All the other get-preference API calls have the exact same start and end state for their directories as well, because this API call does not do any disk writes.

`src-tauri\src\commands\keys\set.rs` involves more advanced disk testing functionality, so we'll just leave that alone for now and commit our changes first.
