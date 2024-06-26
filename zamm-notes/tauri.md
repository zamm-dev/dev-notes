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

We realize at some point that we forgot to refactor out the remaining uses of `setup_database` in `src-tauri\src\models\api_keys.rs` and `src-tauri\src\setup\api_keys.rs`, so we go ahead and do the refactor there as well.

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

### Disk side-effects

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

Next, to prepare for refactoring the API keys API, we move the preference sample disk IO into the `src-tauri/api/sample-disk-writes/preferences/` folder, and update our test files to match. Then, we move the API keys sample disk IO into the `src-tauri/api/sample-disk-writes/sample-init-files/` folder, for example:

- `src-tauri/api/sample-init-files/no-newline/.bashrc` gets moved to `src-tauri/api/sample-disk-writes/shell-init/no-newline/start/.bashrc`, and `src-tauri/api/sample-init-files/no-newline/expected.bashrc` gets moved to `src-tauri/api/sample-disk-writes/shell-init/no-newline/end/.bashrc`, to keep the files for the same test together
- `src-tauri/api/sample-init-files/no-file/expected.bashrc` gets moved to `src-tauri/api/sample-disk-writes/shell-init/new-file/.bashrc` with no `end` folder, because in this case we start off with an initially empty directory anyways
- `src-tauri/api/sample-init-files/unset/.bashrc` and `src-tauri/api/sample-init-files/unset/expected.bashrc` are removed entirely because now we can just use any other test's `start` and `end` directories to test the unset functionality

We then edit the sample calls, such as `src-tauri\api\sample-calls\set_api_key-existing-with-newline.yaml`, to point to the disk side effects files:

```yaml
request:
  - set_api_key
  - >
    {
      "filename": ".bashrc",
      ...
    }
...
sideEffects:
  disk:
    startStateDirectory: shell-init/with-newline/start
    endStateDirectory: shell-init/with-newline/end

```

Because the filename is no longer qualified by the path to the specific test file, we create a new set of tests at `src-tauri\api\sample-disk-writes\shell-init\nested\start\folder\.bashrc`:

```bash
# don't mind me
```

and `src-tauri\api\sample-disk-writes\shell-init\nested\end\folder\.bashrc`:
    
```bash
# don't mind me
export OPENAI_API_KEY="0p3n41-4p1-k3y"
```

and create the corresponding `src-tauri\api\sample-calls\set_api_key-nested-folder.yaml`:

```yaml
request:
  - set_api_key
  - >
    {
      "filename": "folder/.bashrc",
      "service": "OpenAI",
      "api_key": "0p3n41-4p1-k3y"
    }
response:
  message: "null"
sideEffects:
  disk:
    startStateDirectory: shell-init/nested/start
    endStateDirectory: shell-init/nested/end

```

We edit `src-tauri\src\test_helpers\api_testing.rs` to support comparing nested directories:

```rs
...
use std::ffi::OsString;
...

fn compare_dir_all(
    expected_output_dir: impl AsRef<Path>,
    actual_output_dir: impl AsRef<Path>,
) {
    let mut expected_outputs = vec![];
    for entry in fs::read_dir(expected_output_dir).unwrap() {
        let entry = entry.unwrap();
        expected_outputs.push(entry);
    }

    let mut actual_outputs = vec![];
    for entry in fs::read_dir(actual_output_dir).unwrap() {
        let entry = entry.unwrap();
        actual_outputs.push(entry);
    }

    assert_eq!(
        expected_outputs
            .iter()
            .map(|e| e.file_name())
            .collect::<Vec<OsString>>(),
        actual_outputs
            .iter()
            .map(|e| e.file_name())
            .collect::<Vec<OsString>>()
    );
    for (expected_output, actual_output) in
        expected_outputs.iter().zip(actual_outputs.iter())
    {
        let file_type = expected_output.file_type().unwrap();
        if file_type.is_dir() {
            compare_dir_all(expected_output.path(), actual_output.path());
        } else {
            let expected_file = fs::read(expected_output.path()).unwrap();
            let actual_file = fs::read(actual_output.path()).unwrap();

            let expected_file_str = String::from_utf8(expected_file).unwrap();
            let actual_file_str = String::from_utf8(actual_file).unwrap();
            assert_eq!(expected_file_str, actual_file_str);
        }
    }
}

...

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

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
        compare_dir_all(&expected_output_dir, actual_output_dir);
    }

    ...
}
```

We edit `src-tauri\src\commands\keys\set.rs` to use this new functionality:

```rs
#[cfg(test)]
pub mod tests {
    ...
    use crate::test_helpers::{
        ..., SideEffectsHelpers, ...
    };
    ...
    use std::env;
    use stdext::function_name;
    ...

    struct SetApiKeyTestCase<'a> {
        ...
        test_fn_name: &'static str,
        ...
    }

    impl<'a> SampleCallTestCase<SetApiKeyRequest, ZammResult<()>>
        for SetApiKeyTestCase<'a>
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
            args: &Option<SetApiKeyRequest>,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<()> {
            let request = args.as_ref().unwrap();
            let temp_test_dir = side_effects.disk.as_ref().unwrap();
            let current_dir = env::current_dir().unwrap();
            env::set_current_dir(temp_test_dir).unwrap();

            let result = set_api_key_helper(
                self.api_keys,
                self.db,
                request.filename.as_deref(),
                &request.service,
                request.api_key.clone(),
            )
            .await;
            env::set_current_dir(current_dir).unwrap();
            result
        }

        ...
    }

    ...

    pub async fn check_set_api_key_sample<'a>(
        test_fn_name: &'static str,
        ...
    ) {
        let mut test_case = SetApiKeyTestCase {
            ...,
            test_fn_name,
            ...
        };
        test_case.check_sample_call(sample_file).await;
    }

    async fn check_set_api_key_sample_unit(
        test_fn_name: &'static str,
        ...
    ) {
        check_set_api_key_sample(
            test_fn_name,
            ...
        )
        .await;
    }

    #[tokio::test]
    async fn test_write_new_init_file() {
        ...
        check_set_api_key_sample_unit(
            function_name!(),
            ...
        )
        .await;
    }

    ...

    #[tokio::test]
    async fn test_nested_folder() {
        let api_keys = ZammApiKeys(Mutex::new(ApiKeys::default()));
        check_set_api_key_sample_unit(
            function_name!(),
            &setup_zamm_db(),
            "api/sample-calls/set_api_key-nested-folder.yaml",
            &api_keys,
        )
        .await;
    }

    ...
}
```

Note that we are using `env::current_dir` and `env::set_current_dir` to temporarily change the current directory so that the test code doesn't have to do test-specific directory munging -- for example, setting the path for the init file to be the fully qualified temp test directory, but only when the input path is non-empty. If we don't do this, then there might be bugs that we don't catch: for example, writing out to the current directory when nothing should be written at all. If output is written to the current directory where the tests are running from rather than the temp test directory, then it will appear as if no files were written at all and therefore the test will erroneously pass.

However, because we're changing the current directory now, we need to ensure that this doesn't interfere with other tests running simultaneously. We do a quick fix in `src-tauri\Makefile` to remove test parallelism for now:

```Makefile
...

tests:
	cargo test -- --include-ignored --test-threads=1

...
```

and we do the same in `.github\workflows\tests.yaml`:

```yaml
...

jobs:
  ...
  rust:
    ...
    steps:
      ...
      - name: Run Rust Tests
        run: cargo test -- --test-threads=1
        ...
```

We have to edit `src-tauri\src\commands\keys\mod.rs` as well to conform to the new signature:

```rs
#[cfg(test)]
mod tests {
    ...
    use stdext::function_name;
    ...

    #[tokio::test]
    async fn test_get_after_set() {
        ...

        check_set_api_key_sample(
            function_name!(),
            ...
        )
        .await;
        ...
    }
}
```

Finally, because we changed the API calls, frontend tests are also failing. We edit `src-svelte\src\routes\components\api-keys\Display.test.ts`:

```ts
  test("can edit API key", async () => {
    systemInfo.set({
      ...,
      shell_init_file: ".bashrc",
    });
    ...
  });

  ...

  test("can submit with custom file", async () => {
    ...
    playback.addSamples(
      "../src-tauri/api/sample-calls/set_api_key-nested-folder.yaml",
      ...
    );

    ...
    await userEvent.type(fileInput, "folder/.bashrc");
    ...
  });
```

We commit these changes, and do another refactor of `src-tauri\src\test_helpers\api_testing.rs` to give a default implementation of `standard_test_subdir`:

```rs
pub fn standard_test_subdir(api_call: &str, test_fn_name: &str) -> String {
    let test_logical_path =
        test_fn_name.split("::").collect::<Vec<&str>>();
    let test_name = test_logical_path[test_logical_path.len() - 2];
    format!("{}/{}", api_call, test_name)
}
```

We then use this in files such as `src-tauri\src\commands\keys\set.rs`:

```rs
    impl<'a> SampleCallTestCase<SetApiKeyRequest, ZammResult<()>>
        for SetApiKeyTestCase<'a>
    {
        ...

        fn temp_test_subdirectory(&self) -> String {
            standard_test_subdir(Self::EXPECTED_API_CALL, self.test_fn_name)
        }

        ...
    }
```

We now investigate possibilities for easily allowing tests to run in parallel again. We find out that there is the [`sealed_test`](https://docs.rs/sealed_test/latest/sealed_test/) crate, which seems too heavyweight for our purposes, but which in turn uses the [`two-rusty-forks`](https://crates.io/crates/two-rusty-forks) crate. We try installing `two-rusty-forks` and enclosing our tests in the macro as instructed, but unfortunately, because our tests are async, we run into:

```
error: no rules expected the token `async`                          
   --> src\commands\keys\set.rs:258:9
    |
258 |         async fn test_write_new_init_file() {
    |         ^^^^^ no rules expected this token in macro call      
    |
note: while trying to match `fn`
   --> C:\Users\Amos Ng\.cargo\registry\src\index.crates.io-6f17d22bba15001f\two-rusty-forks-0.4.0\src\fork_test.rs:104:10
    |
104 |          fn $test_name:ident() $body:block
    |          ^^

```

Since fixing this would involve more work, we leave it be for now.

Next, we realize that our function signatures have gotten awkward. As such, we rearrange them so that the mocked state variables are together:

```rs
        check_set_api_key_sample(
            function_name!(),
            &setup_zamm_db(),
            &api_keys,
            "api/sample-calls/set_api_key-existing-no-newline.yaml",
            HashMap::new(),
        )
        .await;
```

Next, we put the current directory changing logic into `src-tauri\src\test_helpers\api_testing.rs`, because it is broadly applicable to tests:

```rs
    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...
        // prepare side-effects
        let current_dir = env::current_dir().unwrap();
        ...
        if let Some(side_effects) = &sample.side_effects {
            // prepare disk if necessary
            if let Some(disk_side_effect) = &side_effects.disk {
                let test_temp_dir = self.get_temp_dir();
                self.initialize_temp_dir_inputs(
                    disk_side_effect,
                    &test_temp_dir,
                );
                println!(
                    "Test will use temp directory at {}",
                    &test_temp_dir.display()
                );
                env::set_current_dir(&test_temp_dir).unwrap();
                temp_dir = Some(test_temp_dir);
            }
        }
        ...

        let result = self.make_request(...).await;
        env::set_current_dir(current_dir).unwrap();

        ...
    }
```

Note that we reset the current directory as soon as the request is made. This is to ensure that test failures don't prevent the current directory from being reset, since Rust doesn't have any `finally` blocks. The actual requests shouldn't fail because they should be designed to return `Err` in case of failure -- but if they do because the `make_request` test code fails, then a stackoverflow appears to occur.

Now, we can edit `src-tauri\src\commands\keys\set.rs` to greatly simplify our request function:

```rs
        async fn make_request(
            &mut self,
            ...,
            _: &SideEffectsHelpers,
        ) -> ZammResult<()> {
            let request = args.as_ref().unwrap();

            set_api_key_helper(
                ...
            )
            .await
        }
```

### Database side effects

#### Initial implementation

We edit `src-tauri/api/sample-call-schema.json` to include a database object:

```json
{
  "$schema": "http://json-schema.org/draft-06/schema#",
  "$ref": "#/definitions/SampleCall",
  "definitions": {
    "SampleCall": {
      ...,
      "properties": {
        ...
        "sideEffects": {
          ...
          "properties": {
            ...
            "database": {
              "type": "object",
              "additionalProperties": false,
              "properties": {
                "startStateDump": {
                  "type": "string"
                },
                "endStateDump": {
                  "type": "string"
                }
              },
              "required": ["endStateDump"]
            }
          }
        }
      },
      ...
    }
  }
}
```

As usual, `src-svelte/src/lib/sample-call.ts` and `src-tauri/src/sample_call.rs` are updated automatically by quicktype.

We initially get the output

```yaml
!Ok
api_keys: []
llm_calls:
- id:
    id: d9f5ef39-0095-49d9-89fe-0b20d42963c2
  timestamp: 2024-02-23T03:12:32.673979400
  provider: OpenAI
  llm_requested: gpt-4
  llm: gpt-4-0613
  temperature: 1.0
  prompt_tokens: 32
  response_tokens: 12
  total_tokens: 44
  prompt:
    type: Chat
    messages:
    - role: System
      text: You are ZAMM, a chat program. Respond in first person.
    - role: Human
      text: Hello, does this work?
  completion:
    role: AI
    text: Yes, it works. How can I assist you today?
```

We don't want the redundant "id > id". But if we define `EntityId` as such in `src-tauri\src\models\llm_calls.rs`:

```rs
#[derive(
    ...,
    Serialize, Deserialize
)]
#[diesel(sql_type = Text)]
#[serde(transparent)]
pub struct EntityId {
    pub uuid: Uuid,
}
```

then we get the error

```
---- commands::llms::chat::tests::test_start_conversation stdout ----
Test will use temp directory at C:\Users\AMOSNG~1\AppData\Local\Temp\zamm/tests\chat/test_start_conversation
thread 'commands::llms::chat::tests::test_start_conversation' panicked at src\commands\llms\chat.rs:261:44:
called `Result::unwrap()` on an `Err` value: Error("can only flatten structs and maps", line: 36, column: 1)
```

when trying to parse things back. Unsurprisingly, the same error happens when we try to implement it manually based on [uuid's implementation](https://docs.rs/uuid/1.7.0/src/uuid/external/serde_support.rs.html#58) linked to from the [uuid documentation](https://docs.rs/uuid/1.7.0/uuid/struct.Uuid.html#impl-Deserialize%3C'de%3E-for-Uuid):

```rs
impl Serialize for EntityId {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer,
    {
        self.uuid.serialize(serializer)
    }
}

impl<'de> Deserialize<'de> for EntityId {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        let uuid: Uuid = Deserialize::deserialize(deserializer)?;
        Ok(EntityId { uuid })
    }
}

```

In the end, this turns out to be because we did another flatten in `src-tauri\src\models\llm_calls.rs`:

```rs
pub struct LlmCall {
    #[serde(flatten)]
    pub id: EntityId,
    ...
}
```

We remove this second flatten, and it now works as expected.

Next, we tackle the problem of the added `!Ok` at the beginning of the YAML file. This is because we are using the `!` tag to indicate that the value is a Result. We can remove this by making `db_contents` the Ok variant of a Result by adding a `?` at the end:

```rs
pub async fn write_database_contents(
    ...
) -> ZammResult<()> {
    let db_contents = get_database_contents(...).await?;
    ...
}
```

While coding up the rest of this, we realized that the YAML serializer isn't going to faithfully represent the contents of the SQL row anyways, so we might as well serialize the pretty YAML version of the row.

At the end, we finally end up with a `src-tauri\src\test_helpers\database_contents.rs` that looks like this:

```rs
use crate::commands::errors::ZammResult;
use crate::models::llm_calls::{LlmCall, LlmCallRow, NewLlmCallRow};
use crate::models::{ApiKey, NewApiKey};
use crate::schema::{api_keys, llm_calls};
use crate::ZammDatabase;
use anyhow::anyhow;
use diesel::prelude::*;
use std::fs;
use std::path::PathBuf;
use tokio::sync::MutexGuard;

#[derive(Debug, Default, serde::Serialize, serde::Deserialize)]
pub struct DatabaseContents {
    api_keys: Vec<ApiKey>,
    llm_calls: Vec<LlmCall>,
}

impl DatabaseContents {
    pub fn insertable_api_keys(&self) -> Vec<NewApiKey> {
        self.api_keys.iter().map(|k| k.as_insertable()).collect()
    }

    pub fn insertable_llm_calls(&self) -> Vec<NewLlmCallRow> {
        self.llm_calls.iter().map(|k| k.as_sql_row()).collect()
    }
}

pub async fn get_database_contents(
    zamm_db: &ZammDatabase,
) -> ZammResult<DatabaseContents> {
    let db_mutex: &mut MutexGuard<'_, Option<SqliteConnection>> =
        &mut zamm_db.0.lock().await;
    let db = db_mutex.as_mut().ok_or(anyhow!("Error getting db"))?;
    let api_keys = api_keys::table.load::<ApiKey>(db)?;
    let llm_call_rows = llm_calls::table.load::<LlmCallRow>(db)?;
    let llm_calls: Vec<LlmCall> = llm_call_rows.into_iter().map(|r| r.into()).collect();
    Ok(DatabaseContents {
        api_keys,
        llm_calls,
    })
}

pub async fn write_database_contents(
    zamm_db: &ZammDatabase,
    file_path: &PathBuf,
) -> ZammResult<()> {
    let db_contents = get_database_contents(zamm_db).await?;
    let serialized = serde_yaml::to_string(&db_contents)?;
    fs::write(file_path, serialized)?;
    Ok(())
}

pub async fn read_database_contents(
    zamm_db: &ZammDatabase,
    file_path: &str,
) -> ZammResult<()> {
    let db_mutex: &mut MutexGuard<'_, Option<SqliteConnection>> =
        &mut zamm_db.0.lock().await;
    let db = db_mutex.as_mut().ok_or(anyhow!("Error getting db"))?;

    let serialized = fs::read_to_string(file_path)?;
    let db_contents: DatabaseContents = serde_yaml::from_str(&serialized)?;
    db.transaction::<(), diesel::result::Error, _>(|conn| {
        diesel::insert_into(api_keys::table)
            .values(&db_contents.insertable_api_keys())
            .execute(conn)?;
        diesel::insert_into(llm_calls::table)
            .values(&db_contents.insertable_llm_calls())
            .execute(conn)?;
        Ok(())
    })?;
    Ok(())
}

```

which in turn involves editing `src-tauri\src\models\api_keys.rs` to implement serde's Serialize and Deserialize, as well as a new test-only function to retrieve an insertable version of the struct:

```rs
#[derive(..., serde::Serialize, serde::Deserialize)]
pub struct ApiKey {
    ...
}

#[cfg(test)]
impl ApiKey {
    pub fn as_insertable(&self) -> NewApiKey {
        NewApiKey {
            service: self.service,
            api_key: &self.api_key,
        }
    }
}
```

This in turn involves marking `Service` as `Copy` in `src-tauri\src\setup\api_keys.rs`. *That* in turn involves editing `src-tauri/src/commands/keys/set.rs` to make use of this new Copy trait:

```rs
async fn set_api_key_helper(
    ...
) -> ZammResult<()> {
    ...
    let db_update_result = || -> ZammResult<()> {
        ...
                diesel::replace_into(...)
                    .values(crate::models::NewApiKey {
                        service: *service,
                        ...
                    })
                    ...;
        ...
    }
    ...
}
```

We need to expose this new module in `src-tauri/src/test_helpers/mod.rs`:

```rs
...
pub mod database_contents;
...
```

and then make the changes discussed above to `EntityId` and `LlmCall` in `src-tauri\src\models\llm_calls.rs`, and also mark `LlmCallRow` as both Serializable and Deserializable.

Next, to support the `?` operator in the usage of `serde_yaml` above, we add this to `src-tauri\src\commands\errors.rs`:

```rs
pub enum SerdeError {
    ...,
    #[error(transparent)]
    Yaml {
        #[from]
        source: serde_yaml::Error,
    },
    ...
}

...

impl From<serde_yaml::Error> for Error {
    fn from(err: serde_yaml::Error) -> Self {
        let serde_err: SerdeError = err.into();
        serde_err.into()
    }
}
```

To support this, we will have to move `serde_yaml` into the regular dependencies section of `src-tauri/Cargo.toml`, instead of staying in the dev dependencies section. We could mark everything involved here as `#[cfg(test)]` instead, but since we will be exposing this functionality to the frontend anyways, we'll just do it the easy way.

We make use of this new database functionality in `src-tauri\src\test_helpers\api_testing.rs`:

```rs
...
use crate::test_helpers::database::setup_zamm_db;
use crate::test_helpers::database_contents::{
    read_database_contents, write_database_contents,
};
use crate::ZammDatabase;
use path_absolutize::Absolutize;
use std::collections::HashMap;
...

fn compare_files(
    expected_file_path: impl AsRef<Path>,
    actual_file_path: impl AsRef<Path>,
    output_replacements: &HashMap<String, String>,
) {
    let expected_path_abs = expected_file_path.as_ref().absolutize().unwrap();
    let actual_path_abs = actual_file_path.as_ref().absolutize().unwrap();

    let expected_file = fs::read(&expected_path_abs).unwrap_or_else(|_| {
        panic!(
            "Error reading expected file at {}",
            expected_path_abs.as_ref().display()
        )
    });
    let actual_file = fs::read(&actual_path_abs).unwrap_or_else(|_| {
        panic!(
            "Error reading actual file at {}",
            actual_path_abs.as_ref().display()
        )
    });

    let expected_file_str = String::from_utf8(expected_file).unwrap();
    let actual_file_str = String::from_utf8(actual_file).unwrap();

    let replaced_actual_str = output_replacements
        .iter()
        .fold(actual_file_str, |acc, (k, v)| acc.replace(k, v));
    assert_eq!(expected_file_str, replaced_actual_str);
}

fn compare_dir_all(
    ...,
    output_replacements: &HashMap<String, String>,
) {
    ...
    for (expected_output, actual_output) in
        ...
    {
        ...
        if file_type.is_dir() {
            compare_dir_all(
                ...,
                output_replacements,
            );
        } else {
            compare_files(
                ...,
                output_replacements,
            );
        }
    }
}

#[derive(Default)]
pub struct SideEffectsHelpers {
    pub temp_test_dir: Option<PathBuf>,
    pub disk: Option<PathBuf>,
    pub db: Option<ZammDatabase>,
}

...

impl SampleCall {
    pub fn db_start_dump(&self) -> Option<String> {
        self.side_effects
            .as_ref()
            .and_then(|se| se.database.as_ref())
            .and_then(|db| db.start_state_dump.as_deref())
            .map(|p: &str| format!("api/sample-database-writes/{}", p))
    }

    pub fn db_end_dump(&self) -> Option<String> {
        self.side_effects
            .as_ref()
            .and_then(|se| se.database.as_ref())
            .map(|db| db.end_state_dump.as_ref())
            .map(|p: &str| format!("api/sample-database-writes/{}", p))
    }
}

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    fn output_replacements(
        &self,
        _sample: &SampleCall,
        _result: &U,
    ) -> HashMap<String, String> {
        HashMap::new()
    }

    fn initialize_temp_dir_inputs(&self, ..., temp_dir: &PathBuf) {
        fs::create_dir_all(temp_dir).unwrap();
        ...
    }

    fn compare_temp_dir_outputs(
        &self,
        ...,
        output_replacements: &HashMap<String, String>,
    ) {
        ...
        compare_dir_all(expected_output_dir, actual_output_dir, output_replacements);
    }

    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...
        // prepare side-effects
        let current_dir = env::current_dir().unwrap();
        let mut side_effects_helpers = SideEffectsHelpers::default();
        if let Some(side_effects) = &sample.side_effects {
            let temp_test_dir = self.get_temp_dir();
            println!(
                "Test will use temp directory at {}",
                &temp_test_dir.display()
            );

            // prepare disk if necessary
            if let Some(disk_side_effect) = &side_effects.disk {
                let mut test_disk_dir = temp_test_dir.clone();
                test_disk_dir.push("disk");
                ...
                side_effects_helpers.disk = Some(test_disk_dir);
            }

            // prepare db if necessary
            if side_effects.database.is_some() {
                let test_db = setup_zamm_db();
                if let Some(initial_contents) = sample.db_start_dump() {
                    read_database_contents(&test_db, &initial_contents)
                        .await
                        .unwrap();
                }
                side_effects_helpers.db = Some(test_db);
            }

            side_effects_helpers.temp_test_dir = Some(temp_test_dir);
        }

        ...
        let replacements = self.output_replacements(&sample, &result);
        println!("Replacements:");
        for (k, v) in &replacements {
            println!("  {} -> {}", k, v);
        }

        ...

        // check the call against disk side-effects
        if let Some(test_disk_dir) = &side_effects_helpers.disk {
            ...
            self.compare_temp_dir_outputs(
                ...,
                &replacements,
            );
        }

        // check the call against db side-effects
        if let Some(test_db) = &side_effects_helpers.db {
            let mut test_db_file = side_effects_helpers.temp_test_dir.unwrap().clone();
            test_db_file.push("db.yaml");
            write_database_contents(test_db, &test_db_file)
                .await
                .unwrap();

            compare_files(sample.db_end_dump().unwrap(), &test_db_file, &replacements);
        }
    }
}
```

where `temp_test_dir` is a rename we did for naming consistency.

We edit `src-tauri\src\commands\llms\chat.rs` to make use of this new API:

```rs
#[cfg(test)]
mod tests {
    ...
    use crate::models::llm_calls::ChatMessage;
    use crate::test_helpers::api_testing::standard_test_subdir;
    use std::collections::HashMap;
    use stdext::function_name;
    ...

    fn to_yaml_string<T: Serialize>(obj: &T) -> String {
        serde_yaml::to_string(obj).unwrap().trim().to_string()
    }

    ...

    struct ChatTestCase {
        test_fn_name: &'static str,
        ...
    }

    impl SampleCallTestCase<ChatRequest, ZammResult<LlmCall>> for ChatTestCase {
        ...
        fn temp_test_subdirectory(&self) -> String {
            standard_test_subdir(Self::EXPECTED_API_CALL, self.test_fn_name)
        }

        async fn make_request(
            ...,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<LlmCall> {
            ...
            chat_helper(
                ...,
                side_effects.db.as_ref().unwrap(),
                ...
            )
            .await
        }

        fn output_replacements(
            &self,
            sample: &SampleCall,
            result: &ZammResult<LlmCall>,
        ) -> HashMap<String, String> {
            let expected_output = parse_response(&sample.response.message);
            let actual_output = result.as_ref().unwrap();
            HashMap::from([
                (
                    to_yaml_string(&actual_output.id),
                    to_yaml_string(&expected_output.id),
                ),
                (
                    to_yaml_string(&actual_output.timestamp),
                    to_yaml_string(&expected_output.timestamp),
                ),
            ])
        }

        ...
    }

    ...

    async fn test_llm_api_call(
        test_fn_name: &'static str,
        recording_path: &str,
        sample_path: &str,
    ) {
        ...
        let mut test_case = ChatTestCase {
            test_fn_name,
            ...
        };
        test_case.check_sample_call(sample_path).await;
    }

    #[tokio::test]
    async fn test_start_conversation() {
        test_llm_api_call(
            function_name!(),
            ...
        )
        .await;
    }

    ...
}
```

We create `src-tauri\api\sample-database-writes\conversation-continued.yaml` to reflect the expected state of the DB after two chat request calls:

```yaml
api_keys: []
llm_calls:
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
  timestamp: 2024-01-16T08:50:19.738093890
  llm:
    name: gpt-4-0613
    requested: gpt-4
    provider: OpenAI
  request:
    prompt:
      type: Chat
      messages:
      - role: System
        text: You are ZAMM, a chat program. Respond in first person.
      - role: Human
        text: Hello, does this work?
    temperature: 1.0
  response:
    completion:
      role: AI
      text: Yes, it works. How can I assist you today?
  tokens:
    prompt: 32
    response: 12
    total: 44
- id: c13c1e67-2de3-48de-a34c-a32079c03316
  timestamp: 2024-01-16T09:50:19.738093890
  llm:
    name: gpt-4-0613
    requested: gpt-4
    provider: OpenAI
  request:
    prompt:
      type: Chat
      messages:
      - role: System
        text: You are ZAMM, a chat program. Respond in first person.
      - role: Human
        text: Hello, does this work?
      - role: AI
        text: Yes, it works. How can I assist you today?
      - role: Human
        text: Tell me something funny.
    temperature: 1.0
  response:
    completion:
      role: AI
      text: 'Sure, here''s a joke for you: Why don''t scientists trust atoms? Because they make up everything!'
  tokens:
    prompt: 57
    response: 22
    total: 79

```

We do the same for `src-tauri\api\sample-database-writes\conversation-started.yaml`, which is only a smaller version of the above. We now have to edit `.pre-commit-config.yaml` to prevent these files from being formatted at commit, because the test will be doing an exact string match, and there appears to be [no way](https://users.rust-lang.org/t/is-there-a-crate-that-let-me-customize-code-style-of-printed-yaml-json-indent-and-stuff/84757) to get `serde-yaml` to format the output differently:

```yaml
      - id: prettier
        ...
        exclude: ^src-tauri/api/sample-database-writes/
```

Finally, we edit our sample API calls, such as `src-tauri\api\sample-calls\chat-continue-conversation.yaml`, to refer to these new database dumps:

```yaml
...
sideEffects:
  database:
    startStateDump: conversation-started.yaml
    endStateDump: conversation-continued.yaml
```

`src-tauri\api\sample-calls\chat-start-conversation.yaml` only has the end state dump because it doesn't start off with any data:

```yaml
...
sideEffects:
  database:
    endStateDump: conversation-started.yaml

```

We commit this.

#### API key settings

Now we try to do it for the API key setting as well. During implementation, we edit `src-tauri\src\test_helpers\database_contents.rs`:

```rs
...
use path_absolutize::Absolutize;
...

pub async fn read_database_contents(
    ...
) -> ZammResult<()> {
    ...
    let file_path_buf = PathBuf::from(file_path);
    let file_path_abs = file_path_buf.absolutize()?;
    let serialized = fs::read_to_string(&file_path_abs).map_err(|e| {
        anyhow!("Error reading file at {}: {}", &file_path_abs.display(), e)
    })?;
    ...
}
```

in order to debug the error

```
The system cannot find the path specified. (os error 3)
```

After doing so, we finally see the reason for the failure:

```
called `Result::unwrap()` on an `Err` value: Other { source: Error reading file at C:\Users\AMOSNG~1\AppData\Local\Temp\zamm\tests\set_api_key\test_unset\disk\api\sample-database-writes\empty.yaml: The system cannot find the path specified. (os error 3) }
```

We are trying to access the gold file *after* the current directory has been moved.

We end up with editing `src-tauri\src\test_helpers\api_testing.rs` such that

```rs
fn read_sample(filename: &str) -> SampleCall {
    let file_path = Path::new(filename);
    let abs_file_path = file_path.absolutize().unwrap();
    let sample_str = fs::read_to_string(&abs_file_path)
        .unwrap_or_else(|_| panic!("No file found at {}", abs_file_path.display()));
    ...
}

...

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...

        // prepare side-effects
        ...
        if let Some(side_effects) = &sample.side_effects {
            ...

            // prepare disk if necessary
            if let Some(disk_side_effect) = &side_effects.disk {
                let mut test_disk_dir = temp_test_dir.clone();
                test_disk_dir.push("disk");
                self.initialize_temp_dir_inputs(disk_side_effect, &test_disk_dir);

                env::set_current_dir(&test_disk_dir).unwrap();
                side_effects_helpers.disk = Some(test_disk_dir);
            }

            side_effects_helpers.temp_test_dir = Some(temp_test_dir);
        }

        ...
    }

    ...
}
```

We can now edit `src-tauri\src\commands\keys\set.rs` to make use of this. The code simplifies down a lot:

```rs
#[cfg(test)]
pub mod tests {
    ...

    impl<'a> SampleCallTestCase<SetApiKeyRequest, ZammResult<()>>
        for SetApiKeyTestCase<'a>
    {
        ...

        async fn make_request(
            ...,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<()> {
            let request = args.as_ref().unwrap();

            set_api_key_helper(
                ...,
                side_effects.db.as_ref().unwrap(),
                ...
            )
            .await
        }

        ...
    }

    ...

    #[tokio::test]
    async fn test_overwrite_different_key() {
        let api_keys = ZammApiKeys(Mutex::new(ApiKeys {
            openai: Some("0p3n41-4p1-k3y".to_string()),
        }));
        check_set_api_key_sample_unit(
            function_name!(),
            &api_keys,
            "api/sample-calls/set_api_key-different.yaml",
        )
        .await;
    }

    ...
}
```

Most of the other changes in that file just involve ripping out code that is no longer used. This new test comes in because this new structure makes it obvious that we never test the case where an existing API key is overwritten with a different value.

We edit `src-tauri\src\commands\keys\mod.rs` as well to update the call to `check_set_api_key_sample`.

We create the sample file for that test case at `src-tauri\api\sample-calls\set_api_key-different.yaml`:

```yaml
request:
  - set_api_key
  - >
    {
      "filename": ".bashrc",
      "service": "OpenAI",
      "api_key": "4-d1ff3r3n7-k3y"
    }
response:
  message: "null"
sideEffects:
  disk:
    endStateDirectory: shell-init/different
  database:
    startStateDump: openai-api-key.yaml
    endStateDump: different-openai-api-key.yaml

```

As that sample file indicates, we create `src-tauri\api\sample-disk-writes\shell-init\different\.bashrc`:

```bash
export OPENAI_API_KEY="4-d1ff3r3n7-k3y"

```

We define `src-tauri\api\sample-database-writes\openai-api-key.yaml` as

```yaml
api_keys:
- service: OpenAI
  api_key: 0p3n41-4p1-k3y
llm_calls: []

```

This will be the starting state for this test, but the ending state for other tests.

We create `src-tauri/api/sample-database-writes/different-openai-api-key.yaml` as well:

```yaml
api_keys:
- service: OpenAI
  api_key: 4-d1ff3r3n7-k3y
llm_calls: []

```

We define `src-tauri\api\sample-database-writes\empty.yaml` for other tests:

```yaml
api_keys: []
llm_calls: []

```

Now we can edit all the other tests, such as `src-tauri\api\sample-calls\set_api_key-unset.yaml`:

```yaml
...
sideEffects:
  ...
  database:
    startStateDump: openai-api-key.yaml
    endStateDump: empty.yaml
```

All the other sample files follow a similar pattern.

#### Raw SQL dump

As mentioned above, the YAML dump isn't a faithful representation of the actual strings being stored in the SQL database. We'll keep it, because it is a more interpretable dump in case any diffs occur. But we'll also add a SQL dump, in case something with the JSON-specific serialization changes.

First we edit `src-tauri\src\test_helpers\database_contents.rs` to provide a new dump function. We run the `sqlite3` command because it's unclear if there's a way to do this from our Diesel connection.

```rs
pub fn dump_sqlite_database(db_path: &PathBuf, dump_path: &PathBuf) {
    let dump_output = std::process::Command::new("sqlite3")
        .arg(db_path)
        // avoid the inserts into __diesel_schema_migrations
        .arg(".dump api_keys llm_calls")
        .output()
        .expect("Error running sqlite3 .dump command");
    // filter output by lines starting with "INSERT"
    let inserts = String::from_utf8_lossy(&dump_output.stdout)
        .lines()
        .filter(|line| line.starts_with("INSERT"))
        .collect::<Vec<&str>>()
        .join("\n");
    fs::write(dump_path, inserts).expect("Error writing dump file");
}
```

Then, we edit `src-tauri\src\test_helpers\database.rs` to provide a way to specify a file for the database:

```rs
...
use std::path::PathBuf;
...

pub fn setup_database(file: Option<&PathBuf>) -> SqliteConnection {
    let mut conn = match file {
        Some(file) => {
            SqliteConnection::establish(file.as_path().to_str().unwrap()).unwrap()
        }
        None => SqliteConnection::establish(":memory:").unwrap(),
    };

    ...
}

pub fn setup_zamm_db(file: Option<&PathBuf>) -> ZammDatabase {
    ZammDatabase(...setup_database(file))
}

```

We'll have to edit all existing calls to `setup_database` and `setup_zamm_db` to pass in `None` by default. For example, we edit this test in `src-tauri\src\setup\api_keys.rs`:

```rs
    #[test]
    fn test_empty_db_doesnt_crash() {
        temp_env::with_var(..., || {
            let conn = setup_database(None);

            ...
        });
    }
```

We do the same in `src-tauri\src\models\api_keys.rs`. However, `src-tauri/src/test_helpers/api_testing.rs` is where we make use of this new option:

```rs
impl SampleCall {
    pub fn db_start_dump(&self) -> Option<String> {
        self.side_effects
            ...
            .map(|p: &str| format!("api/sample-database-writes/{}/dump.yaml", p))
    }

    pub fn db_end_dump(&self, extension: &str) -> String {
        let end_state_dump_dir = &self
            .side_effects
            .as_ref()
            .unwrap()
            .database
            .as_ref()
            .unwrap()
            .end_state_dump;

        format!(
            "api/sample-database-writes/{}/dump.{}",
            end_state_dump_dir, extension
        )
    }
}

struct TestDatabaseInfo {
    pub temp_db_dir: PathBuf,
    pub temp_db_file: PathBuf,
}

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    async fn check_sample_call(&mut self, ...) -> SampleCallResult<T, U> {
        ...
        // prepare side-effects
        ...
        let mut test_db_info: Option<TestDatabaseInfo> = None;
        if let Some(side_effects) = &sample.side_effects {
            ...

            // prepare db if necessary
            if side_effects.database.is_some() {
                let temp_db_dir = temp_test_dir.join("database");
                fs::create_dir_all(&temp_db_dir).unwrap();
                let temp_db_file = temp_db_dir.join("db.sqlite3");

                let test_db = setup_zamm_db(Some(&temp_db_file));
                ...

                side_effects_helpers.db = Some(test_db);
                test_db_info = Some(TestDatabaseInfo {
                    temp_db_dir,
                    temp_db_file,
                });
            }

            ...
        }

        ...

        // check the call against db side-effects
        if let Some(test_db) = &side_effects_helpers.db {
            let db_info = test_db_info.unwrap();
            let actual_db_yaml_dump = db_info.temp_db_dir.join("dump.yaml");
            let actual_db_sql_dump = db_info.temp_db_dir.join("dump.sql");
            write_database_contents(test_db, &actual_db_yaml_dump)
                .await
                .unwrap();
            dump_sqlite_database(&db_info.temp_db_file, &actual_db_sql_dump);

            compare_files(
                sample.db_end_dump("yaml"),
                &actual_db_yaml_dump,
                &replacements,
            );
            compare_files(
                sample.db_end_dump("sql"),
                &actual_db_sql_dump,
                &replacements,
            );
        }

        ...
    }
}

```

Note that in doing this, we have changed the directory structure. For example, we move `src-tauri/api/sample-database-writes/different-openai-api-key.yaml` into `src-tauri/api/sample-database-writes/different-openai-api-key/dump.yaml` and create a `src-tauri/api/sample-database-writes/different-openai-api-key/dump.sql` that looks like:

```sql
INSERT INTO api_keys VALUES('open_ai','4-d1ff3r3n7-k3y');
```

We also edit our sample files to match. For example, `src-tauri/api/sample-calls/set_api_key-different.yaml` becomes:

```yaml
...
sideEffects:
  ...
  database:
    startStateDump: openai-api-key
    endStateDump: different-openai-api-key
```

where the dump variable now refers to a directory instead of a file. Unfortunately, this sort of change cannot be caught by a type checker, because it is merely a string as far as configuration is concerned.

Finally, we edit `src-tauri\src\commands\llms\chat.rs`, because the replacements hashmap now needs an extra entry to handle how the timestamps are represented differently for SQLite and for serde-yaml:

```rs
        fn output_replacements(
            &self,
            ...
        ) -> HashMap<String, String> {
            ...
            let expected_output_timestamp = to_yaml_string(&expected_output.timestamp);
            let actual_output_timestamp = to_yaml_string(&actual_output.timestamp);
            HashMap::from([
                ...,
                (
                    // sqlite dump produces timestamps with space instead of T
                    actual_output_timestamp.replace("T", " "),
                    expected_output_timestamp.replace("T", " "),
                ),
                (actual_output_timestamp, expected_output_timestamp),
            ])
        }
```

#### Using the replacement HashMap in src-tauri\src\commands\keys\set.rs

We'll use the new standard replacement API in `src-tauri\src\commands\keys\set.rs`:

```rs
    impl<'a> SampleCallTestCase<SetApiKeyRequest, ZammResult<()>>
        for SetApiKeyTestCase<'a>
    {
        ...

        fn output_replacements(
            &self,
            _: &SampleCall,
            _: &ZammResult<()>,
        ) -> HashMap<String, String> {
            self.json_replacements.clone()
        }

        ...
    }

    impl<'a> ZammResultReturn<SetApiKeyRequest, ()> for SetApiKeyTestCase<'a> {}

    ...

    #[tokio::test]
    async fn test_invalid_filename() {
        ...
        check_set_api_key_sample(
            ...,
            HashMap::from([(
                // error on Windows
                "The system cannot find the path specified. (os error 3)"
                    .to_string(),
                // should be replaced by equivalent error on Linux
                "Is a directory (os error 21)".to_string(),
            )]),
        )
        .await;
    }
```

We remove the quotes for the hashmap in `test_invalid_filename` for clarity.

Our tests fail. It turns out we aren't doing this replacement for standard API calls. We edit `src-tauri\src\test_helpers\api_testing.rs`:

```rs
...

fn apply_replacements(
    input: &str,
    replacements: &HashMap<String, String>,
) -> String {
    replacements
        .iter()
        .fold(input.to_string(), |acc, (k, v)| acc.replace(k, v))
}

fn compare_files(
    ...
) {
    ...
    let replaced_actual_str = apply_replacements(&actual_file_str, output_replacements);
    assert_eq!(expected_file_str, replaced_actual_str);
}

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...

        // check the call against sample outputs
        let actual_json = ...;
        let replaced_actual_json = apply_replacements(&actual_json, &replacements);
        ...
        assert_eq!(replaced_actual_json, expected_json);
        ...
    }

    ...
}
```

### Network side effects

Once again, we edit `src-tauri\api\sample-call-schema.json`:

```json
{
  "$schema": "http://json-schema.org/draft-06/schema#",
  "$ref": "#/definitions/SampleCall",
  "definitions": {
    "SampleCall": {
      ...,
      "properties": {
        ...,
        "sideEffects": {
          ...,
          "properties": {
            ...,
            "network": {
              "type": "object",
              "additionalProperties": false,
              "properties": {
                "recordingFile": {
                  "type": "string"
                }
              },
              "required": ["recordingFile"]
            }
          }
        }
      },
      ...
    }
  }
}
```

As usual, `src-svelte/src/lib/sample-call.ts` and `src-tauri/src/sample_call.rs` are automatically updated by quicktype.

We move everything from `src-tauri/api/sample-call-requests` to `src-tauri/api/sample-network-requests`. We then edit `src-tauri/src/test_helpers/api_testing.rs` to make use of this:

```rs
...
use reqwest_middleware::{ClientBuilder, ClientWithMiddleware};
use rvcr::{VCRMiddleware, VCRMode};
use vcr_cassette::Headers;
...

pub struct NetworkHelper {
    pub network_client: ClientWithMiddleware,
    pub mode: VCRMode,
}

#[derive(Default)]
pub struct SideEffectsHelpers {
    ...,
    pub network: Option<NetworkHelper>,
}

...

impl SampleCall {
    pub fn network_recording(&self) -> String {
        let recording_file = &self
            .side_effects
            .as_ref()
            .unwrap()
            .network
            .as_ref()
            .unwrap()
            .recording_file;

        format!("api/sample-network-requests/{}", recording_file)
    }

    ...
}

...

const CENSORED: &str = "<CENSORED>";

fn censor_headers(headers: &Headers, blacklisted_keys: &[&str]) -> Headers {
    return headers
        .clone()
        .iter()
        .map(|(k, v)| {
            if blacklisted_keys.contains(&k.as_str()) {
                (k.clone(), vec![CENSORED.to_string()])
            } else {
                (k.clone(), v.clone())
            }
        })
        .collect();
}

pub trait SampleCallTestCase<T, U>
    ...
{
    ...

    async fn check_sample_call(&mut self, sample_file: &str) -> SampleCallResult<T, U> {
        ...

        if let Some(side_effects) = &sample.side_effects {
            ...

            // prepare network if necessary
            if side_effects.network.is_some() {
                let recording_path = PathBuf::from(sample.network_recording());
                let vcr_mode = if !recording_path.exists() {
                    VCRMode::Record
                } else {
                    VCRMode::Replay
                };
                let middleware = VCRMiddleware::try_from(recording_path)
                    .unwrap()
                    .with_mode(vcr_mode.clone())
                    .with_modify_request(|req| {
                        req.headers = censor_headers(&req.headers, &["authorization"]);
                    })
                    .with_modify_response(|resp| {
                        resp.headers =
                            censor_headers(&resp.headers, &["openai-organization"]);
                    });

                let network_client: ClientWithMiddleware =
                    ClientBuilder::new(reqwest::Client::new())
                        .with(middleware)
                        .build();

                side_effects_helpers.network = Some(NetworkHelper {
                    network_client,
                    mode: vcr_mode,
                });
            }

            ...
        }

        ...
    }
}
```

It doesn't appear as if there is any way to assert with `rvcr` that all requests have been played, so we leave that be for now.

We edit `src-tauri\src\commands\llms\chat.rs`, which features much simplified code now:

```rs
    impl SampleCallTestCase<ChatRequest, ZammResult<LlmCall>> for ChatTestCase {
        ...

        async fn make_request(
            &mut self,
            ...
        ) -> ZammResult<LlmCall> {
            let actual_args = args.as_ref().unwrap().clone();

            let network_helper = side_effects.network.as_ref().unwrap();
            let api_keys = match network_helper.mode {
                VCRMode::Record => ZammApiKeys(Mutex::new(ApiKeys {
                    openai: env::var("OPENAI_API_KEY").ok(),
                })),
                VCRMode::Replay => ZammApiKeys(Mutex::new(ApiKeys {
                    openai: Some("dummy".to_string()),
                })),
            };

            chat_helper(
                &api_keys,
                ...,
                network_helper.network_client.clone(),
            )
            .await
        }
      
        ...
    }
```

We also remove the network file argument from `test_llm_api_call`, because that is now handled by the YAML file.

Finally, we modify the sample recordings such as `src-tauri/api/sample-calls/chat-continue-conversation.yaml` to point to the new network recording file:

```yaml
...
sideEffects:
  ...
  network:
    recordingFile: continue-conversation.json
```

All tests pass on the first go. To make sure that they are not passing spuriously, we modify a sample network recording file to ensure that tests are in fact reading from it.

#### CI

We edit `.github\workflows\tests.yaml` to install `sqlite3` before the tests run:

```yaml
...
jobs:
  ...
  rust:
    ...
    steps:
      ...
      - name: Install Tauri dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y ... sqlite3
```

We find that it fails on CI:

```
error[E0706]: functions in traits cannot be declared `async`
  --> src/commands/keys/get.rs:35:9
   |
35 |           async fn make_request(
   |           ^----
   |           |
   |  _________`async` because of this
   | |
36 | |             &mut self,
37 | |             _: &Option<()>,
38 | |             _: &SideEffectsHelpers,
39 | |         ) -> ZammResult<ApiKeys> {
   | |________________________________^
   |
   = note: `async` trait functions are not currently supported
   = note: consider using the `async-trait` crate: https://crates.io/crates/async-trait
   = note: see issue #91611 <https://github.com/rust-lang/rust/issues/91611> for more information
```

We can reproduce this on our Linux box with Rust 1.71.1. We try installing the latest version of Rust on our Linux box:

```bash
$ asdf install rust latest
$ asdf global rust 1.76.0
```

Now everything passes. We update the versions everywhere else, in `Dockerfile`, in `.github/workflows/publish.yaml`, and in `.github/workflows/tests.yaml`.

## Sample call database refactor

We decide to further refactor the test database YAML dumps to be independent of the display logic for the frontend API call.

### Automatically dumping YAML contents from SQL.

To do that, however, we first make sure we generate the YAML dump from the SQL dump if the YAML dump is missing. This is because the YAML dump is a more readable but less precise version of the SQL dump. We edit `src-tauri/src/test_helpers/api_testing.rs` to do the copying before any file comparisons are made, and to fail the test as soon as the copy has been made because we don't want any CI tests inadvertently passing due to a lack of copies:

```rs
...
use crate::test_helpers::database::{setup_database, ...};
...
use crate::test_helpers::database_contents::{
    ..., load_sqlite_database, ...,
};
...
use tokio::sync::Mutex;
...

fn copy_missing_gold_file(expected_path_abs: &Path, actual_path_abs: &Path) {
    fs::create_dir_all(expected_path_abs.parent().unwrap()).unwrap();
    fs::copy(actual_path_abs, expected_path_abs).unwrap();
    eprintln!(
        "Gold file not found at {}, copied actual file from {}",
        expected_path_abs.display(),
        actual_path_abs.display(),
    );
}

async fn dump_sql_to_yaml(
    expected_sql_dump_abs: &PathBuf,
    expected_yaml_dump_abs: &PathBuf,
) {
    let mut db = setup_database(None);
    load_sqlite_database(&mut db, expected_sql_dump_abs);
    let zamm_db = ZammDatabase(Mutex::new(Some(db)));
    write_database_contents(&zamm_db, expected_yaml_dump_abs)
        .await
        .unwrap();
}

async fn setup_gold_db_files(
    expected_yaml_dump: impl AsRef<Path>,
    actual_yaml_dump: impl AsRef<Path>,
    expected_sql_dump: impl AsRef<Path>,
    actual_sql_dump: impl AsRef<Path>,
) {
    let expected_yaml_dump_abs = expected_yaml_dump.as_ref().absolutize().unwrap();
    let actual_yaml_dump_abs = actual_yaml_dump.as_ref().absolutize().unwrap();
    let expected_sql_dump_abs = expected_sql_dump.as_ref().absolutize().unwrap();
    let actual_sql_dump_abs = actual_sql_dump.as_ref().absolutize().unwrap();

    if !expected_yaml_dump_abs.exists() && !expected_sql_dump_abs.exists() {
        copy_missing_gold_file(&expected_yaml_dump_abs, &actual_yaml_dump_abs);
        copy_missing_gold_file(&expected_sql_dump_abs, &actual_sql_dump_abs);
        panic!(
            "Copied gold files to {}",
            expected_yaml_dump_abs.parent().unwrap().display()
        );
    } else if !expected_yaml_dump_abs.exists() && expected_sql_dump_abs.exists() {
        dump_sql_to_yaml(
            &expected_sql_dump_abs.to_path_buf(),
            &expected_yaml_dump_abs.to_path_buf(),
        )
        .await;
        panic!(
            "Dumped YAML from SQL to {}",
            expected_yaml_dump_abs.display()
        );
    } else {
        if !expected_yaml_dump_abs.exists() {
            panic!("No YAML dump found at {}", expected_yaml_dump_abs.display());
        }
        if !expected_sql_dump_abs.exists() {
            panic!("No SQL dump found at {}", expected_sql_dump_abs.display());
        }
    }
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

            // prepare db if necessary
            if side_effects.database.is_some() {
                ...

                if let Some(initial_yaml_dump) = sample.db_start_dump() {
                    let initial_yaml_dump_abs = Path::new(&initial_yaml_dump)
                        .absolutize()
                        .unwrap()
                        .to_path_buf();
                    if !initial_yaml_dump_abs.exists() {
                        let initial_sql_dump_abs =
                            initial_yaml_dump_abs.with_extension("sql");
                        dump_sql_to_yaml(&initial_sql_dump_abs, &initial_yaml_dump_abs)
                            .await;
                        panic!(
                            "Dumped YAML from SQL to {}",
                            initial_yaml_dump_abs.display()
                        );
                    }

                    read_database_contents(&test_db, &initial_yaml_dump)
                        .await
                        .unwrap();
                }

                ...
            }

            setup_gold_db_files(
                sample.db_end_dump("yaml"),
                &actual_db_yaml_dump,
                sample.db_end_dump("sql"),
                &actual_db_sql_dump,
            )
            .await;

            compare_files(
                sample.db_end_dump("yaml"),
                ...
            );
            compare_files(
                sample.db_end_dump("sql"),
                ...
            );
        }

        ...
    }
}
```

This requires the addition of this function to `src-tauri/src/test_helpers/database_contents.rs`:

```rs
...
use diesel::connection::SimpleConnection;
...

pub fn load_sqlite_database(conn: &mut SqliteConnection, dump_path: &PathBuf) {
    let dump = fs::read_to_string(dump_path).expect("Error reading dump file");
    conn.batch_execute(&dump)
        .expect("Error loading dump into database");
}
```

### Reflecting SQL tables in YAML output dump

We edit `src-tauri/src/models/llm_calls/linkage.rs` to add new data structures reflecting individual rows in the tables:

```rs
...
use serde::{Deserialize, Serialize};
...

#[derive(Debug, Queryable, Selectable, Clone, Serialize, Deserialize)]
#[diesel(table_name = llm_call_follow_ups)]
pub struct LlmCallFollowUp {
    pub previous_call_id: EntityId,
    pub next_call_id: EntityId,
}

#[cfg(test)]
impl LlmCallFollowUp {
    pub fn as_insertable(&self) -> NewLlmCallFollowUp {
        NewLlmCallFollowUp {
            previous_call_id: &self.previous_call_id,
            next_call_id: &self.next_call_id,
        }
    }
}

...

#[derive(Debug, Queryable, Selectable, Clone, Serialize, Deserialize)]
#[diesel(table_name = llm_call_variants)]
pub struct LlmCallVariant {
    pub canonical_id: EntityId,
    pub variant_id: EntityId,
}

#[cfg(test)]
impl LlmCallVariant {
    pub fn as_insertable(&self) -> NewLlmCallVariant {
        NewLlmCallVariant {
            canonical_id: &self.canonical_id,
            variant_id: &self.variant_id,
        }
    }
}
```

We edit `src-tauri/src/models/llm_calls/mod.rs` as well to export these new data structures:

```rs
#[allow(unused_imports)]
pub use linkage::{
    LlmCallFollowUp, LlmCallVariant, ...
};
```

We add the `as_insertable` function to `src-tauri/src/models/llm_calls/row.rs` as well:

```rs
#[cfg(test)]
impl LlmCallRow {
    pub fn as_insertable(&self) -> NewLlmCallRow {
        NewLlmCallRow {
            id: &self.id,
            timestamp: &self.timestamp,
            provider: &self.provider,
            llm_requested: &self.llm_requested,
            llm: &self.llm,
            temperature: &self.temperature,
            prompt_tokens: self.prompt_tokens.as_ref(),
            response_tokens: self.response_tokens.as_ref(),
            total_tokens: self.total_tokens.as_ref(),
            prompt: &self.prompt,
            completion: &self.completion,
        }
    }
}
```

Then, in `src-tauri/src/models/llm_calls/llm_call.rs` we remove the `as_sql_row`, `as_follow_up_row`, and `as_variant_rows` functions from `impl LlmCall`, because those are no longer needed with the new `as_insertable` functions.

Finally, we are ready to redefine the way our test database dump is structured. We edit `src-tauri/src/test_helpers/database_contents.rs` to drastically change the structure of the `DatabaseContents` struct, which in turn radically simplifies our YAML dump code because it no longer relies on a join:

```rs
...
use crate::models::llm_calls::{
    LlmCallFollowUp, LlmCallRow, LlmCallVariant, NewLlmCallFollowUp, NewLlmCallRow,
    NewLlmCallVariant,
};
...

#[derive(Debug, Default, serde::Serialize, serde::Deserialize)]
pub struct LlmCallData {
    instances: Vec<LlmCallRow>,
    #[serde(skip_serializing_if = "Vec::is_empty", default)]
    follow_ups: Vec<LlmCallFollowUp>,
    #[serde(skip_serializing_if = "Vec::is_empty", default)]
    variants: Vec<LlmCallVariant>,
}

impl LlmCallData {
    pub fn is_default(&self) -> bool {
        self.instances.is_empty()
            && self.follow_ups.is_empty()
            && self.variants.is_empty()
    }
}

#[derive(Debug, Default, serde::Serialize, serde::Deserialize)]
pub struct DatabaseContents {
    #[serde(skip_serializing_if = "Vec::is_empty", default)]
    api_keys: Vec<ApiKey>,
    #[serde(skip_serializing_if = "LlmCallData::is_default", default)]
    llm_calls: LlmCallData,
}

impl DatabaseContents {
    ...

    pub fn insertable_llm_calls(&self) -> Vec<NewLlmCallRow> {
        self.llm_calls
            .instances
            ...
            .map(|k| k.as_insertable())
            ...
    }

    pub fn insertable_call_follow_ups(&self) -> Vec<NewLlmCallFollowUp> {
        self.llm_calls
            .follow_ups
            ...
            .map(|k| k.as_insertable())
            ...
    }

    pub fn insertable_call_variants(&self) -> Vec<NewLlmCallVariant> {
        self.llm_calls
            .variants
            ...
            .map(|k| k.as_insertable())
            ...
    }
}

pub async fn get_database_contents(
    ...
) -> ZammResult<DatabaseContents> {
    ...
    let api_keys = ...;
    let llm_calls_instances = llm_calls::table.load::<LlmCallRow>(db)?;
    let follow_ups = llm_call_follow_ups::table.load::<LlmCallFollowUp>(db)?;
    let variants = llm_call_variants::table.load::<LlmCallVariant>(db)?;

    Ok(DatabaseContents {
        api_keys,
        llm_calls: LlmCallData {
            instances: llm_calls_instances,
            follow_ups,
            variants,
        },
    })
}
```

Now we have to redefine the YAML generation in `src-tauri/api/sample-database-writes/many-api-calls/generate.py` as well:

```py
...

yaml_preamble = """llm_calls:
  instances:
"""

...

def generate_api_call_yaml(i: int) -> str:
    return f"""  - id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a{i:02d}
    timestamp: 2024-01-16T08:{i:02d}:50.738093890
    provider: OpenAI
    llm_requested: gpt-4
    llm: gpt-4-0613
    temperature: 1.0
    prompt_tokens: 15
    response_tokens: 3
    total_tokens: 18
    prompt:
      type: Chat
      messages:
      - role: System
        text: You are ZAMM, a chat program. Respond in first person.
      - role: Human
        text: This is a mock conversation.
    completion:
      role: AI
      text: Mocking number {i}.
"""
```

### Removing the option from `make_request` and `serialize_result`

We edit `src-tauri/src/test_helpers/api_testing.rs` to simplify the signatures a bit:

```rs
...

pub struct SampleCallResult<T, U>
where
    ...
{
    ...
    pub args: T,
    ...
}

...

pub trait SampleCallTestCase<T, U>
where
    ...
{
    ...

    async fn make_request(..., args: &T, ...) -> U;

    ...

    async fn check_result(..., args: &T, ...);

    ...

    async fn check_sample_call(...) -> SampleCallResult<T, U> {
        ...

        // make the call
        let args = if Self::CALL_HAS_ARGS {
            self.parse_args(&sample.request[1])
        } else {
            self.parse_args("null")
        };
        let result = self.make_request(&args, ...).await;
        ...

        // check the call against sample outputs
        ...
        self.check_result(..., &args, ...).await;
    }

    ...
}

pub trait DirectReturn<T, U>
where
    ...
{
    ...

    async fn check_result(..., _args: &T, ...) {}
}

pub trait ZammResultReturn<T, U>
where
    ...
{
    ...
    async fn check_result(
        ...,
        _args: &T,
        ...
    ) {
        ...
    }
}
```

We update the test code across various files to match. For example, `src-tauri/src/commands/database/export.rs`:

```rs
        async fn make_request(
            ...
            args: &ExportDbRequest,
            ...
        ) -> ZammResult<DatabaseCounts> {
            export_db_helper(..., &args.path).await
        }

        ...

        async fn check_result(
            &self,
            sample: &SampleCall,
            args: &ExportDbRequest,
            result: &ZammResult<DatabaseCounts>,
        ) {
            ...
        }
```

The ones with no inputs are simplified as well now, such as `src-tauri/src/commands/keys/get.rs`:

```rs
        async fn make_request(
            ...,
            _: &(),
            ...
        ) -> ZammResult<ApiKeys> {
            ...
        }

        ...

         async fn check_result(
            ...
            args: &(),
            ...
        ) {
            ...
        }
```

Some of them require more extensive changes, in particular to remove any `unwrap()`'s associated with the former Option. For example, `src-tauri/src/commands/keys/set.rs` is now simplified:

```rs
        async fn make_request(
            ...
            args: &SetApiKeyRequest,
            ...
        ) -> ZammResult<()> {
            set_api_key_helper(
                ...
                args.filename.as_deref(),
                &args.service,
                args.api_key.clone(),
            )
            .await
        }

        ...

        async fn check_result(
            ...
            args: &SetApiKeyRequest,
            ...
        ) {
            ...
            if args.api_key.is_empty() {
                ...
            } else {
                ...
                let arg_api_key = Some(args.api_key.clone());
                ...
            }
        }
```

We also edit `src-tauri/src/commands/sounds.rs` to make `Sound` also derive `Copy`, for simplicity. We had previously avoided doing so to practice and learn how to avoid unnecessary copying, but that is no longer necessary for us.

```rs
#[derive(..., Copy, ...)]
pub enum Sound {
    ...
}
```

### Using a macro

We define these macros in `src-tauri/src/test_helpers/api_testing.rs`:

```rs
#[macro_export]
macro_rules! impl_result_test_case {
    ($test_case:ident, $api_call_name:ident, $has_args:ident, $req_type:ident, $resp_type:ty) => {
        struct $test_case {
            test_fn_name: &'static str,
        }

        impl $crate::test_helpers::SampleCallTestCase<$req_type, ZammResult<$resp_type>>
            for $test_case
        {
            const EXPECTED_API_CALL: &'static str = stringify!($api_call_name);
            const CALL_HAS_ARGS: bool = $has_args;

            fn temp_test_subdirectory(&self) -> String {
                $crate::test_helpers::api_testing::standard_test_subdir(
                    Self::EXPECTED_API_CALL,
                    self.test_fn_name,
                )
            }

            async fn make_request(
                &mut self,
                args: &$req_type,
                side_effects: &$crate::test_helpers::SideEffectsHelpers,
            ) -> ZammResult<$resp_type> {
                make_request_helper(args, side_effects).await
            }

            fn serialize_result(
                &self,
                sample: &$crate::sample_call::SampleCall,
                result: &ZammResult<$resp_type>,
            ) -> String {
                $crate::test_helpers::ZammResultReturn::serialize_result(
                    self, sample, result,
                )
            }

            async fn check_result(
                &self,
                sample: &$crate::sample_call::SampleCall,
                args: &$req_type,
                result: &ZammResult<$resp_type>,
            ) {
                $crate::test_helpers::ZammResultReturn::check_result(
                    self, sample, args, result,
                )
                .await
            }
        }

        impl $crate::test_helpers::ZammResultReturn<$req_type, $resp_type>
            for $test_case
        {
        }
    };
}

#[macro_export]
macro_rules! check_sample {
    ($test_case:ident, $test_name:ident, $sample_call_file:expr) => {
        #[tokio::test]
        async fn $test_name() {
            let mut test_case = $test_case {
                test_fn_name: stdext::function_name!(),
            };
            $crate::test_helpers::SampleCallTestCase::check_sample_call(
                &mut test_case,
                $sample_call_file,
            )
            .await;
        }
    };
}

```

Note that we fully qualify our paths so as to make imports less confusing in the places where these macros are used.

We also have to mark that submodule as exporting a macro, in `src-tauri/src/test_helpers/mod.rs`:

```rs
#[macro_use]
pub mod api_testing;
...
```


Now our files such as `src-tauri\src\commands\database\export.rs` feature drastically less testing boilerplate:

```rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::test_helpers::SideEffectsHelpers;
    use crate::{check_sample, impl_result_test_case};
    use serde::{Deserialize, Serialize};

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct ExportDbRequest {
        path: String,
    }

    async fn make_request_helper(
        args: &ExportDbRequest,
        side_effects: &SideEffectsHelpers,
    ) -> ZammResult<DatabaseCounts> {
        export_db_helper(side_effects.db.as_ref().unwrap(), &args.path).await
    }

    impl_result_test_case!(
        ExportDbTestCase,
        export_db,
        true,
        ExportDbRequest,
        DatabaseCounts
    );

    check_sample!(
        ExportDbTestCase,
        test_export_llm_calls,
        "./api/sample-calls/export_db-populated.yaml"
    );

    check_sample!(
        ExportDbTestCase,
        test_export_api_key,
        "./api/sample-calls/export_db-api-key.yaml"
    );
}
```

While some files like `src-tauri\src\commands\llms\chat.rs` won't benefit from the first macro due to having a custom `output_replacements`, we realize even for that file that the custom `ZammResultReturn::serialize_result` is no longer needed, precisely due to the custom implementation of `output_replacements`. Even then, it could still make use of the `check_sample!` macro.

We skip the following files as well:

- `src-tauri\src\commands\keys\get.rs` due to having a custom test case definition
- `src-tauri\src\commands\keys\set.rs` due to having custom test case and `check_result` definitions

At some point we run into the problem

```
error: cannot determine resolution for the macro `check_sample`       
   --> src\commands\database\import.rs:155:5
    |
155 |     check_sample!(
    |     ^^^^^^^^^^^^
    |
    = note: import resolution is stuck, try simplifying macro imports 

error: cannot determine resolution for the macro `impl_result_test_case`
   --> src\commands\database\import.rs:147:5
    |
147 |     impl_result_test_case!(
    |     ^^^^^^^^^^^^^^^^^^^^^
    |
    = note: import resolution is stuck, try simplifying macro imports 
```

Based on [this answer](https://stackoverflow.com/questions/77573472/rust-macro-crate-cant-find-crate-import-resolution-is-stuck), we try cleaning the project, but the problem persists. Based on [this issue](https://github.com/rust-lang/rust/issues/57966), it seems as if the compiler somehow cannot find our macros anymore. Upon further inspection, it turns out that one of our pre-commit commands (perhaps cargo clippy, or perhaps pre-commit itself) somehow removed our entire `src-tauri\src\test_helpers\api_testing.rs` file. Fortunately, our editor had a record of the file before it was wiped on disk.

Next, we add a macro for direct returns to `src-tauri/src/test_helpers/api_testing.rs`, and to edit `req_type` for consistent to accept a `ty` so that it can take in `()`:

```rs
#[macro_export]
macro_rules! impl_direct_test_case {
    ($test_case:ident, $api_call_name:ident, $has_args:ident, $req_type:ty, $resp_type:ty) => {
        struct $test_case {
            test_fn_name: &'static str,
        }

        impl $crate::test_helpers::SampleCallTestCase<$req_type, $resp_type>
            for $test_case
        {
            const EXPECTED_API_CALL: &'static str = stringify!($api_call_name);
            const CALL_HAS_ARGS: bool = $has_args;

            fn temp_test_subdirectory(&self) -> String {
                $crate::test_helpers::api_testing::standard_test_subdir(
                    Self::EXPECTED_API_CALL,
                    self.test_fn_name,
                )
            }

            async fn make_request(
                &mut self,
                args: &$req_type,
                side_effects: &$crate::test_helpers::SideEffectsHelpers,
            ) -> $resp_type {
                make_request_helper(args, side_effects).await
            }

            fn serialize_result(
                &self,
                sample: &$crate::sample_call::SampleCall,
                result: &$resp_type,
            ) -> String {
                $crate::test_helpers::DirectReturn::serialize_result(
                    self, sample, result,
                )
            }

            async fn check_result(
                &self,
                sample: &$crate::sample_call::SampleCall,
                args: &$req_type,
                result: &$resp_type,
            ) {
                $crate::test_helpers::DirectReturn::check_result(
                    self, sample, args, result,
                )
                .await
            }
        }

        impl $crate::test_helpers::DirectReturn<$req_type, $resp_type> for $test_case {}
    };
}

#[macro_export]
macro_rules! impl_result_test_case {
    (..., $req_type:ty, ...) => {
    }
}
```

Now, our direct return tests are simplified, for example with `src-tauri\src\commands\preferences\read.rs`:

```rs
#[cfg(test)]
mod tests {
    use super::*;
    use crate::test_helpers::SideEffectsHelpers;
    use crate::{check_sample, impl_direct_test_case};

    async fn make_request_helper(
        _: &(),
        side_effects: &SideEffectsHelpers,
    ) -> Preferences {
        get_preferences_helper(&side_effects.disk)
    }

    impl_direct_test_case!(
        GetPreferencesTestCase,
        get_preferences,
        false,
        (),
        Preferences
    );

    #[cfg(not(target_os = "windows"))]
    check_sample!(
        GetPreferencesTestCase,
        test_without_file,
        "./api/sample-calls/get_preferences-no-file.yaml"
    );

    #[cfg(target_os = "windows")]
    check_sample!(
        GetPreferencesTestCase,
        test_without_file,
        "./api/sample-calls/get_preferences-no-file-windows.yaml"
    );

    ...
}
```

We once again skip doing this for `src-tauri\src\commands\system.rs` because it involves custom test cases.

# Build

## Update signature

We follow the instructions [here](https://tauri.app/v1/guides/distribution/updater/):

```bash
$ yarn tauri signer generate -- -w ~/.tauri/zamm.key 
yarn run v1.22.19
warning From Yarn 1.0 onwards, scripts don't require "--" for options to be forwarded. In a future version, any explicit "--" will be forwarded as-is to the scripts.
$ tauri signer generate -w /root/.tauri/zamm.key
Please enter a password to protect the secret key.
Password: 
Password (one more time): 
Deriving a key from the password in order to encrypt the secret key... done

Your keypair was generated successfully
Private: /root/.tauri/zamm.key (Keep it secret!)
Public: /root/.tauri/zamm.key.pub
---------------------------

Environment variables used to sign:
`TAURI_PRIVATE_KEY`  Path or String of your private key
`TAURI_KEY_PASSWORD`  Your private key password (optional)

ATTENTION: If you lose your private key OR password, you'll not be able to sign your update package and updates will not work.
---------------------------

Done in 75.88s.
```

Next, we find that we must point Tauri to a specific URL endpoint for its version check. We could implement this as a server, but then we'll need to keep the server running, point the subdomain to our server, and setup and configure SSL for the subdomain. Instead, we use the approach recommended in [this thread](https://github.com/tauri-apps/tauri/discussions/2776) and simply use GitHub's gists feature to host a static JSON file.

By default, GitHub's gist's "Raw" button refers to a specific version of a Gist, for example:

> https://gist.githubusercontent.com/amosjyng/b3bbcb4ea176009732ea6898f87fe102/raw/161d0a8cc3a5b76e550a06b93311791b6eb22202/zamm-latest-version.json

Instead, as noted in the GitHub discussion on Tauri auto-update, we refer to the most up to date version of the gist with

> https://gist.githubusercontent.com/amosjyng/b3bbcb4ea176009732ea6898f87fe102/raw

Now we edit `src-tauri/tauri.conf.json` as such:

```json
{
  ...,
  "tauri": {
    ...,
    "updater": {
      "active": true,
      "dialog": true,
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6IG1pbmlzaWduIHB1YmxpYyBrZXk6IDkzNkRBRTE1N0QzNTkyRjkKUldUNWtqVjlGYTV0azNqYjI1VHYxbHdBTDQxcVQ1WS8wKzI0dXhGbjZ3VnJKaXZCTWtuR09FaE4K",
      "endpoints": [
        "https://gist.githubusercontent.com/amosjyng/b3bbcb4ea176009732ea6898f87fe102/raw"
      ]
    },
    ...
  }
}
```

We set the `TAURI_PRIVATE_KEY` and `TAURI_KEY_PASSWORD` environment variables locally, and check that the new signatures are being generated as expected:

```bash
$ make
...
    Finished 2 bundles at:
        /root/zamm/src-tauri/target/release/bundle/deb/zamm_0.1.1_amd64.deb
        /root/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage
        /root/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage.tar.gz (updater)

    Finished 1 updater signature at:
        /root/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage.tar.gz.sig
```

We set these on the GitHub repo secrets as well, and edit `.github/workflows/publish.yaml` to put these into the CI run:

```yaml
name: publish

on: workflow_dispatch

env:
  ...
  TAURI_PRIVATE_KEY: ${{ secrets.TAURI_PRIVATE_KEY }}
  TAURI_KEY_PASSWORD: ${{ secrets.TAURI_KEY_PASSWORD }}
  ...

...
```

We run the publish workflow on CI, and find that the Windows build is now failing with

```
yarn run v1.22.21
error Command "build" not found.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

This is odd, because from [this answer](https://stackoverflow.com/a/66849577), the error implies that the `build` script is not defined in `package.json`. We check `src-svelte/package.json` and find that it is indeed defined:

```json
{
  ...,
  "scripts": {
    ...,
    "build": "vite build",
    ...
  },
  ...
}
```

We eventually realize that this is because the Svelte fork location was moved in [`yarn-fork.md`](/general-notes/setup/repo/yarn-fork.md), and as part of that we erroneously changed the cd to end up in the `forks` folder rather than `src-svelte`. We make the publish script clearer instead of using a `cd`:

```yaml
      - name: Setup frontend dependencies
        run: |
          npm install -g pnpm
          pnpm install
          pnpm compile
        working-directory: forks/neodrag
      - name: Build frontend
        run: |
          yarn install
          yarn svelte-kit sync
          yarn build
        working-directory: src-svelte
```

We see that upon a successful run, the publish script generates a `latest.json` for us:

```json
{
  "version": "0.1.1",
  "notes": "",
  "pub_date": "2024-03-10T04:33:18.684Z",
  "platforms": {
    "linux-x86_64": {
      "signature": "dW50cnVzdGVkIGNvbW1lbnQ6IHNpZ25hdHVyZSBmcm9tIHRhdXJpIHNlY3JldCBrZXkKUlVUNWtqVjlGYTV0azBEWElNMlo0TnUvclg0V012NS9RVWZHZ2JUcjhRZy9CSmNxcEJFNjVkUFhVL0ZMRHNmMTF3ZVgvK3FqVWw3NHE1aExKRU5TZUI2T05yYXNObnFOL1FnPQp0cnVzdGVkIGNvbW1lbnQ6IHRpbWVzdGFtcDoxNzEwMDQ0MzUyCWZpbGU6emFtbV8wLjEuMV9hbWQ2NC5BcHBJbWFnZS50YXIuZ3oKNWxBeXkyUTllSnhFTzNOb0FuRUVIQ0xkUmRSOHpZK0ZjUnVqNGdnQjFwaW4xMFRnaVBUbEEvOGh0YkJXS2JndkRLWHA0dmoyQ21JbXRRdGZpcGpRQ0E9PQo=",
      "url": "https://github.com/zamm-dev/zamm/releases/download/v0.1.1/zamm_0.1.1_amd64.AppImage.tar.gz"
    },
    "windows-x86_64": {
      "signature": "dW50cnVzdGVkIGNvbW1lbnQ6IHNpZ25hdHVyZSBmcm9tIHRhdXJpIHNlY3JldCBrZXkKUlVUNWtqVjlGYTV0azFlQWdXQ2dnaWlKTDZhdGJBNXd4cmlGNVZ4MTBvWll4UFlGYnVtOTU4dzdmSlhJL3EzUFRLbHZJSGhpNjg2Y0k0bW13RWNTQlJvVmdBQlFTcFdUcWcwPQp0cnVzdGVkIGNvbW1lbnQ6IHRpbWVzdGFtcDoxNzEwMDQ0NjMwCWZpbGU6emFtbV8wLjEuMV94NjRfZW4tVVMubXNpLnppcApzQ1VFS05ZUzdPU00rK2w1QWl3N2RjblJuVEgyVFpnWWl3YUxaV0RZV1RrWlByd0VGOGJCdmlKT3J6MUw3SVhWeHRZK3FzczlJSDVtbjhnMjd2My9CUT09Cg==",
      "url": "https://github.com/zamm-dev/zamm/releases/download/v0.1.1/zamm_0.1.1_x64_en-US.msi.zip"
    },
    "darwin-aarch64": {
      "signature": "dW50cnVzdGVkIGNvbW1lbnQ6IHNpZ25hdHVyZSBmcm9tIHRhdXJpIHNlY3JldCBrZXkKUlVUNWtqVjlGYTV0ay9Vd2FLbjgzM21rVjkxencxamVlNFJ2c0ZHQTFVTERsOGFGR013RTlsY3JZUWxDRXV3QlRuRTZTK1puNmkvL2Q0VFNVK0RiTGY4bWVGU0tNV3prZkFzPQp0cnVzdGVkIGNvbW1lbnQ6IHRpbWVzdGFtcDoxNzEwMDQ1MTk0CWZpbGU6emFtbS5hcHAudGFyLmd6Ck1iTnNSMTJ0cGx1ZFVFYzl4bEp2N1Vad0UreWdiR2lYbHNSeU5FSWJhQ2syNE5pQ2ltKzZnYWZTWExpSnkyUGFZUDhDUGRmcGZ5OWhzYStDNzZja0JBPT0K",
      "url": "https://github.com/zamm-dev/zamm/releases/download/v0.1.1/zamm_universal.app.tar.gz"
    },
    "darwin-x86_64": {
      "signature": "dW50cnVzdGVkIGNvbW1lbnQ6IHNpZ25hdHVyZSBmcm9tIHRhdXJpIHNlY3JldCBrZXkKUlVUNWtqVjlGYTV0ay9Vd2FLbjgzM21rVjkxencxamVlNFJ2c0ZHQTFVTERsOGFGR013RTlsY3JZUWxDRXV3QlRuRTZTK1puNmkvL2Q0VFNVK0RiTGY4bWVGU0tNV3prZkFzPQp0cnVzdGVkIGNvbW1lbnQ6IHRpbWVzdGFtcDoxNzEwMDQ1MTk0CWZpbGU6emFtbS5hcHAudGFyLmd6Ck1iTnNSMTJ0cGx1ZFVFYzl4bEp2N1Vad0UreWdiR2lYbHNSeU5FSWJhQ2syNE5pQ2ltKzZnYWZTWExpSnkyUGFZUDhDUGRmcGZ5OWhzYStDNzZja0JBPT0K",
      "url": "https://github.com/zamm-dev/zamm/releases/download/v0.1.1/zamm_universal.app.tar.gz"
    }
  }
}
```

We update our gist with this new `latest.json` file, except that we edit it to spoof it as version `0.1.2`. We find that sure enough, our app successfully notifies us of an update on launch. A more rigorous test would involve actually releasing a mock 0.1.2 release and verifying that it is successfully downloaded and installed. However, we content ourselves with just the notification for now.

On CI, we find the error

```
   Compiling zamm v0.1.1 (/__w/zamm/zamm/src-tauri)
    Finished release [optimized] target(s) in 19.39s
    Bundling zamm_0.1.1_amd64.deb (/__w/zamm/zamm/src-tauri/target/release/bundle/deb/zamm_0.1.1_amd64.deb)
    Bundling zamm_0.1.1_amd64.AppImage (/__w/zamm/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage)
    Bundling /__w/zamm/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage.tar.gz (/__w/zamm/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage.tar.gz)
    Finished 2 bundles at:
        /__w/zamm/zamm/src-tauri/target/release/bundle/deb/zamm_0.1.1_amd64.deb
        /__w/zamm/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage
        /__w/zamm/zamm/src-tauri/target/release/bundle/appimage/zamm_0.1.1_amd64.AppImage.tar.gz (updater)

       Error A public key has been found, but no private key. Make sure to set `TAURI_PRIVATE_KEY` environment variable.
make: *** [Makefile:7: build] Error 1
```

We add the keys to `.github/workflows/tests.yaml` as well:

```yaml
...

env:
  ...
  TAURI_PRIVATE_KEY: ${{ secrets.TAURI_PRIVATE_KEY }}
  TAURI_KEY_PASSWORD: ${{ secrets.TAURI_KEY_PASSWORD }}

...
```
