# Displaying API calls

## Initial implementation

### Returning API call info from backend

First we create `src-tauri\src\commands\llms\get_api_call.rs`:

```rust
use crate::commands::errors::ZammResult;
use crate::models::llm_calls::{EntityId, LlmCall, LlmCallRow};
use crate::schema::llm_calls;
use crate::ZammDatabase;
use anyhow::anyhow;
use diesel::prelude::*;
use diesel::RunQueryDsl;
use specta::specta;
use tauri::State;
use uuid::Uuid;

async fn get_api_call_helper(
    zamm_db: &ZammDatabase,
    api_call_id: &str,
) -> ZammResult<LlmCall> {
    let parsed_uuid = EntityId {
        uuid: Uuid::parse_str(api_call_id)?,
    };
    let mut db = zamm_db.0.lock().await;
    let conn = db.as_mut().ok_or(anyhow!("Failed to lock database"))?;
    let result: LlmCallRow = llm_calls::table
        .filter(llm_calls::id.eq(parsed_uuid))
        .first::<LlmCallRow>(conn)?;
    Ok(result.into())
}

#[tauri::command(async)]
#[specta]
pub async fn get_api_call(
    database: State<'_, ZammDatabase>,
    id: &str,
) -> ZammResult<LlmCall> {
    get_api_call_helper(&database, id).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sample_call::SampleCall;
    use crate::test_helpers::api_testing::standard_test_subdir;
    use crate::test_helpers::{
        SampleCallTestCase, SideEffectsHelpers, ZammResultReturn,
    };
    use serde::{Deserialize, Serialize};
    use stdext::function_name;

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct GetApiCallRequest {
        id: String,
    }

    struct GetApiCallTestCase {
        test_fn_name: &'static str,
    }

    impl SampleCallTestCase<GetApiCallRequest, ZammResult<LlmCall>> for GetApiCallTestCase {
        const EXPECTED_API_CALL: &'static str = "get_api_call";
        const CALL_HAS_ARGS: bool = true;

        fn temp_test_subdirectory(&self) -> String {
            standard_test_subdir(Self::EXPECTED_API_CALL, self.test_fn_name)
        }

        async fn make_request(
            &mut self,
            args: &Option<GetApiCallRequest>,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<LlmCall> {
            get_api_call_helper(
                side_effects.db.as_ref().unwrap(),
                &args.as_ref().unwrap().id,
            )
            .await
        }

        fn serialize_result(
            &self,
            sample: &SampleCall,
            result: &ZammResult<LlmCall>,
        ) -> String {
            ZammResultReturn::serialize_result(self, sample, result)
        }

        async fn check_result(
            &self,
            sample: &SampleCall,
            args: Option<&GetApiCallRequest>,
            result: &ZammResult<LlmCall>,
        ) {
            ZammResultReturn::check_result(self, sample, args, result).await
        }
    }

    impl ZammResultReturn<GetApiCallRequest, LlmCall> for GetApiCallTestCase {}

    async fn check_get_api_key_sample(test_fn_name: &'static str, file_prefix: &str) {
        let mut test_case = GetApiCallTestCase { test_fn_name };
        test_case.check_sample_call(file_prefix).await;
    }

    #[tokio::test]
    async fn test_get_api_call_start_conversation() {
        check_get_api_key_sample(
            function_name!(),
            "./api/sample-calls/get_api_call-start-conversation.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_get_api_call_continued_conversation() {
        check_get_api_key_sample(
            function_name!(),
            "./api/sample-calls/get_api_call-continue-conversation.yaml",
        )
        .await;
    }
}

```

We need to define a new error in `src-tauri\src\commands\errors.rs`:

```rust
#[derive(thiserror::Error, Debug)]
pub enum Error {
    ...
    #[error(transparent)]
    Uuid {
        #[from]
        source: uuid::Error,
    },
    ...
}
```

We expose this in `src-tauri\src\commands\llms\mod.rs`:

```rust
...
pub mod get_api_call;

...
pub use get_api_call::get_api_call;

```

and `src-tauri\src\commands\mod.rs`:

```rust
pub use llms::{..., get_api_call};
```

and `src-tauri/src/main.rs`:

```rust
...
use commands::{
    ..., get_api_call, ...
};

...

fn main() {
    #[cfg(all(debug_assertions, not(target_os = "windows")))]
    ts::export(
        collect_types![
            ...,
            get_api_call,
        ],
        "../src-svelte/src/lib/bindings.ts",
    )
    .unwrap();

    ...

    tauri::Builder::default()
        ...
        .invoke_handler(tauri::generate_handler![
            ...,
            get_api_call,
        ])
        ...
}

```

We then create `src-tauri/api/sample-calls/get_api_call-start-conversation.yaml` based on the output of the mocked chat calls:

```yaml
request:
  - get_api_call
  - >
    {
      "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74"
    }
response:
  message: >
    {
      "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
      "timestamp": "2024-01-16T08:50:19.738093890",
      "llm": {
        "name": "gpt-4-0613",
        "requested": "gpt-4",
        "provider": "OpenAI"
      },
      "request": {
        "prompt": {
          "type": "Chat",
          "messages": [
            {
              "role": "System",
              "text": "You are ZAMM, a chat program. Respond in first person."
            },
            {
              "role": "Human",
              "text": "Hello, does this work?"
            }
          ]
        },
        "temperature": 1.0
      },
      "response": {
        "completion": {
          "role": "AI",
          "text": "Yes, it works. How can I assist you today?"
        }
      },
      "tokens": {
        "prompt": 32,
        "response": 12,
        "total": 44
      }
    }
sideEffects:
  database:
    startStateDump: conversation-continued
    endStateDump: conversation-continued

```

We similarly create `src-tauri\api\sample-calls\get_api_call-continue-conversation.yaml` to return the other API call from that sample database dump. We make sure to run the dev app on a Unix machine to generate the new bindings.

We realize later on that we should rename `check_get_api_key_sample` to `check_get_api_call_sample` in `src-tauri\src\commands\llms\get_api_call.rs`.

### Displaying on frontend

We create an initial display for API calls at `src-svelte\src\routes\api-calls\[slug]\ApiCall.svelte`:

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import SubInfoBox from "$lib/SubInfoBox.svelte";
  import { getApiCall } from "$lib/bindings";
  import Loading from "$lib/Loading.svelte";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";

  const formatter = new Intl.DateTimeFormat("en-US", {
    year: "numeric",
    month: "long",
    day: "numeric",
    hour: "numeric",
    minute: "numeric",
    second: "numeric",
    hour12: true,
  });

  export let id: string;
  let humanTime: string | undefined = undefined;
  let temperature: string | undefined = undefined;
  let apiCallPromise = getApiCall(id)
    .then((apiCall) => {
      const timestamp = apiCall.timestamp + "Z";
      const date = new Date(timestamp);
      humanTime = formatter.format(date);
      temperature = apiCall.request.temperature.toFixed(2);

      return apiCall;
    })
    .catch((error) => {
      snackbarError(error);
    });
</script>

<InfoBox title="API Call">
  {#await apiCallPromise}
    <Loading />
  {:then apiCall}
    <table>
      <tr>
        <td>ID</td>
        <td>{apiCall?.id ?? "Unknown"}</td>
      </tr>
      <tr>
        <td>Time</td>
        <td>{humanTime ?? "Unknown"}</td>
      </tr>
      <tr>
        <td>LLM</td>
        <td>
          {apiCall?.llm.requested ?? "Unknown"}
          {#if apiCall?.llm.requested !== apiCall?.llm.name}
            → {apiCall?.llm.name}
          {/if}
        </td>
      </tr>
      <tr>
        <td>Temperature</td>
        <td>
          {temperature ?? "Unknown"}
        </td>
      </tr>
      <tr>
        <td>Tokens</td>
        <td>
          {apiCall?.tokens.prompt ?? "Unknown"} prompt +
          {apiCall?.tokens.response ?? "Unknown"} response =
          {apiCall?.tokens.total ?? "Unknown"} total tokens
        </td>
      </tr>
    </table>

    <SubInfoBox subheading="Prompt">
      <div class="prompt">
        {#each apiCall?.request.prompt.messages ?? [] as message}
          <div class={"message " + message.role.toLowerCase()}>
            <span class="role">{message.role}</span>
            <pre>{message.text}</pre>
          </div>
        {/each}
      </div>
    </SubInfoBox>

    <SubInfoBox subheading="Response">
      <pre class="response">{apiCall?.response.completion.text ??
          "Unknown"}</pre>
    </SubInfoBox>
  {/await}
</InfoBox>

<style>
  td:first-child {
    color: var(--color-faded);
    padding-right: 1rem;
  }

  .prompt {
    margin-bottom: 1rem;
  }

  .message {
    margin: 0.5rem 0;
    padding: 0.5rem;
    border-radius: var(--corner-roundness);
    display: flex;
    gap: 1rem;
    align-items: center;
  }

  .message.system {
    background-color: var(--color-system);
  }

  .message.human {
    background-color: var(--color-human);
  }

  .message.ai {
    background-color: var(--color-ai);
  }

  .message .role {
    color: var(--color-faded);
    width: 4rem;
    min-width: 4rem;
    text-align: center;
  }

  pre {
    white-space: pre-wrap;
    font-family: var(--font-mono);
    margin: 0;
    text-align: left;
  }
</style>

```

We refactor these colors out of `src-svelte\src\routes\chat\MessageUI.svelte`:

```css
  .message.human {
    --message-color: var(--color-human);
  }

  .message.ai {
    --message-color: var(--color-ai);
  }
```

and into `src-svelte\src\routes\styles.css`:

```css
:root {
  ...
  --color-system: #FFF7CC;
  --color-human: #E5FFE5;
  --color-ai: #E5E5FF;
  ...
}
```


We create `src-svelte\src\routes\api-calls\[slug]\+page.svelte` to show this on that page. We see from [this answer](https://stackoverflow.com/a/68579586) that the way to access page params client-side should be:

```svelte
<script>
  import { page } from "$app/stores";
  import ApiCall from "./ApiCall.svelte";
</script>

<ApiCall id={$page.params.slug} />
```

We display this new component in `src-svelte\src\routes\api-calls\[slug]\ApiCall.stories.ts`:

```ts
import ApiCallComponent from "./ApiCall.svelte";
import type { StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";

export default {
  component: ApiCallComponent,
  title: "Screens/LLM Call/Individual",
  argTypes: {},
  decorators: [TauriInvokeDecorator],
};

const Template = ({ ...args }) => ({
  Component: ApiCallComponent,
  props: args,
});

export const Narrow: StoryObj = Template.bind({}) as any;
Narrow.args = {
  id: "c13c1e67-2de3-48de-a34c-a32079c03316",
};
Narrow.parameters = {
  viewport: {
    defaultViewport: "mobile2",
  },
  sampleCallFiles: [
    "/api/sample-calls/get_api_call-continue-conversation.yaml",
  ],
};

export const Wide: StoryObj = Template.bind({}) as any;
Wide.args = {
  id: "c13c1e67-2de3-48de-a34c-a32079c03316",
};
Wide.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
  sampleCallFiles: [
    "/api/sample-calls/get_api_call-continue-conversation.yaml",
  ],
};

```

and register it in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "individual"],
    variants: ["narrow", "wide"],
  },
];
```

Because the new story does not have the Snackbar, we decide to add debugging to `src-svelte/src/lib/snackbar/Snackbar.svelte` once again:

```svelte
  export function snackbarError(msg: string) {
    console.warn(msg);
    ...
  }
```

### Displaying API calls in list

To get to the individual API calls page, we must first display a list of them to choose from.

#### Backend

We create `src-tauri\src\commands\llms\get_api_calls.rs`:

```rust
use crate::commands::errors::ZammResult;
use crate::models::llm_calls::{LlmCall, LlmCallRow};
use crate::schema::llm_calls;
use crate::ZammDatabase;
use anyhow::anyhow;
use diesel::prelude::*;
use diesel::RunQueryDsl;
use specta::specta;
use tauri::State;

async fn get_api_calls_helper(
    zamm_db: &ZammDatabase,
    offset: i64,
) -> ZammResult<Vec<LlmCall>> {
    let mut db = zamm_db.0.lock().await;
    let conn = db.as_mut().ok_or(anyhow!("Failed to lock database"))?;
    let result: Vec<LlmCallRow> = llm_calls::table
        .order(llm_calls::timestamp.desc())
        .offset(offset)
        .limit(10)
        .load::<LlmCallRow>(conn)?;
    let calls: Vec<LlmCall> = result.into_iter().map(|row| row.into()).collect();
    Ok(calls)
}

#[tauri::command(async)]
#[specta]
pub async fn get_api_calls(
    database: State<'_, ZammDatabase>,
    offset: i64,
) -> ZammResult<Vec<LlmCall>> {
    get_api_calls_helper(&database, offset).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sample_call::SampleCall;
    use crate::test_helpers::api_testing::standard_test_subdir;
    use crate::test_helpers::{
        SampleCallTestCase, SideEffectsHelpers, ZammResultReturn,
    };
    use serde::{Deserialize, Serialize};
    use stdext::function_name;

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct GetApiCallsRequest {
        offset: i64,
    }

    struct GetApiCallsTestCase {
        test_fn_name: &'static str,
    }

    impl SampleCallTestCase<GetApiCallsRequest, ZammResult<Vec<LlmCall>>>
        for GetApiCallsTestCase
    {
        const EXPECTED_API_CALL: &'static str = "get_api_calls";
        const CALL_HAS_ARGS: bool = true;

        fn temp_test_subdirectory(&self) -> String {
            standard_test_subdir(Self::EXPECTED_API_CALL, self.test_fn_name)
        }

        async fn make_request(
            &mut self,
            args: &Option<GetApiCallsRequest>,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<Vec<LlmCall>> {
            get_api_calls_helper(
                side_effects.db.as_ref().unwrap(),
                args.as_ref().unwrap().offset,
            )
            .await
        }

        fn serialize_result(
            &self,
            sample: &SampleCall,
            result: &ZammResult<Vec<LlmCall>>,
        ) -> String {
            ZammResultReturn::serialize_result(self, sample, result)
        }

        async fn check_result(
            &self,
            sample: &SampleCall,
            args: Option<&GetApiCallsRequest>,
            result: &ZammResult<Vec<LlmCall>>,
        ) {
            ZammResultReturn::check_result(self, sample, args, result).await
        }
    }

    impl ZammResultReturn<GetApiCallsRequest, Vec<LlmCall>> for GetApiCallsTestCase {}

    async fn check_get_api_calls_sample(test_fn_name: &'static str, file_prefix: &str) {
        let mut test_case = GetApiCallsTestCase { test_fn_name };
        test_case.check_sample_call(file_prefix).await;
    }

    #[tokio::test]
    async fn test_get_api_calls_empty() {
        check_get_api_calls_sample(
            function_name!(),
            "./api/sample-calls/get_api_calls-empty.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_get_api_calls_less_than_10() {
        check_get_api_calls_sample(
            function_name!(),
            "./api/sample-calls/get_api_calls-small.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_get_api_calls_full() {
        check_get_api_calls_sample(
            function_name!(),
            "./api/sample-calls/get_api_calls-full.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_get_api_calls_offset() {
        check_get_api_calls_sample(
            function_name!(),
            "./api/sample-calls/get_api_calls-offset.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_get_api_calls_offset_empty() {
        check_get_api_calls_sample(
            function_name!(),
            "./api/sample-calls/get_api_calls-offset-empty.yaml",
        )
        .await;
    }
}
```

Note that for simplicity and development speed, we're simply returning the entire LLM object so as to not have to define a new struct for the API call. We expose this in `src-tauri\src\commands\llms\mod.rs`:

```rust
...
pub mod get_api_calls;

...
pub use get_api_calls::get_api_calls;

```

and in `src-tauri/src/commands/mod.rs`:

```rust
pub use llms::{..., get_api_calls};
```

and we make use of this in `src-tauri\src\main.rs`:

```rust
...
use commands::{
    ..., get_api_calls, ...
};

...

fn main() {
    #[cfg(all(debug_assertions, not(target_os = "windows")))]
    ts::export(
        collect_types![
            ...,
            get_api_calls,
        ],
        "../src-svelte/src/lib/bindings.ts",
    )
    .unwrap();

    ...

    tauri::Builder::default()
        ...
        .invoke_handler(tauri::generate_handler![
            ...,
            get_api_calls,
        ])
        ...
}

```

We set up the tests by creating a demo database with 12 API calls at `src-tauri\api\sample-database-writes\many-api-calls\dump.yaml`:

```yaml
api_keys: []
llm_calls:
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a00
  timestamp: 2024-01-16T08:50:00.738093890
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
        text: This is a mock conversation.
    temperature: 1.0
  response:
    completion:
      role: AI
      text: Mocking number 0.
  tokens:
    prompt: 15
    response: 3
    total: 18
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a02
  timestamp: 2024-01-16T08:50:02.738093890
  ...
  response:
    completion:
      role: AI
      text: Mocking number 2.
  ...
...
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a12
  timestamp: 2024-01-16T08:50:12.738093890
  ...
  ...
  response:
    completion:
      role: AI
      text: Mocking number 12.
  ...
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a01
  timestamp: 2024-01-16T08:50:01.738093890
  ...
  response:
    completion:
      role: AI
      text: Mocking number 1.
  ...
...
```

Note that the only thing we change each time is the last two digits of the ID, the last two seconds of the timestamp, and the number mentioned in the response. We also do all the even-numbered calls first, and then all the odd-numbered calls next, to test that the sorting is working correctly. We use a Python script to automate this. The `src-tauri\api\sample-database-writes\many-api-calls\dump.sql` will be created automatically once we run the tests; we just need to copy it from the temporary test directory to the correct location.

Now that we have that, we can start defining all our test files, starting with the most trivial case at `src-tauri\api\sample-calls\get_api_calls-empty.yaml`:

```yaml
request:
  - get_api_calls
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

We create `src-tauri/api/sample-calls/get_api_calls-small.yaml` based on responses from the individual API calls:

```yaml
request:
  - get_api_calls
  - >
    {
      "offset": 0
    }
response:
  message: >
    [
      {
        "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
        "timestamp": "2024-01-16T09:50:19.738093890",
        "llm": {
          "name": "gpt-4-0613",
          "requested": "gpt-4",
          "provider": "OpenAI"
        },
        "request": {
          "prompt": {
            "type": "Chat",
            "messages": [
              {
                "role": "System",
                "text": "You are ZAMM, a chat program. Respond in first person."
              },
              {
                "role": "Human",
                "text": "Hello, does this work?"
              },
              {
                "role": "AI",
                "text": "Yes, it works. How can I assist you today?"
              },
              {
                "role": "Human",
                "text": "Tell me something funny."
              }
            ]
          },
          "temperature": 1.0
        },
        "response": {
          "completion": {
            "role": "AI",
            "text": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
          }
        },
        "tokens": {
          "prompt": 57,
          "response": 22,
          "total": 79
        }
      },
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
        "timestamp": "2024-01-16T08:50:19.738093890",
        "llm": {
          "name": "gpt-4-0613",
          "requested": "gpt-4",
          "provider": "OpenAI"
        },
        "request": {
          "prompt": {
            "type": "Chat",
            "messages": [
              {
                "role": "System",
                "text": "You are ZAMM, a chat program. Respond in first person."
              },
              {
                "role": "Human",
                "text": "Hello, does this work?"
              }
            ]
          },
          "temperature": 1.0
        },
        "response": {
          "completion": {
            "role": "AI",
            "text": "Yes, it works. How can I assist you today?"
          }
        },
        "tokens": {
          "prompt": 32,
          "response": 12,
          "total": 44
        }
      }
    ]
sideEffects:
  database:
    startStateDump: conversation-continued
    endStateDump: conversation-continued

```

We test that a full response exists at `src-tauri\api\sample-calls\get_api_calls-full.yaml`:

```yaml
request:
  - get_api_calls
  - >
    {
      "offset": 0
    }
response:
  message: >
    [
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a12",
        "timestamp": "2024-01-16T08:50:12.738093890",
        "llm": {
          "name": "gpt-4-0613",
          "requested": "gpt-4",
          "provider": "OpenAI"
        },
        "request": {
          "prompt": {
            "type": "Chat",
            "messages": [
              {
                "role": "System",
                "text": "You are ZAMM, a chat program. Respond in first person."
              },
              {
                "role": "Human",
                "text": "This is a mock conversation."
              }
            ]
          },
          "temperature": 1.0
        },
        "response": {
          "completion": {
            "role": "AI",
            "text": "Mocking number 12."
          }
        },
        "tokens": {
          "prompt": 15,
          "response": 3,
          "total": 18
        }
      },
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a11",
        "timestamp": "2024-01-16T08:50:11.738093890",
        ...,
        "response": {
          "completion": {
            "role": "AI",
            "text": "Mocking number 11."
          }
        },
        ...
      },
      ...,
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a03",
        "timestamp": "2024-01-16T08:50:03.738093890",
        ...,
        "response": {
          "completion": {
            "role": "AI",
            "text": "Mocking number 3."
          }
        },
        ...
      }
    ]
sideEffects:
  database:
    startStateDump: many-api-calls
    endStateDump: many-api-calls

```

Note that the responses start from the most recent, and end at 3. The rest will come in the offset test at `src-tauri/api/sample-calls/get_api_calls-offset.yaml`:

```yaml
request:
  - get_api_calls
  - >
    {
      "offset": 10
    }
response:
  message: >
    [
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a02",
        "timestamp": "2024-01-16T08:50:02.738093890",
        "llm": {
          "name": "gpt-4-0613",
          "requested": "gpt-4",
          "provider": "OpenAI"
        },
        "request": {
          "prompt": {
            "type": "Chat",
            "messages": [
              {
                "role": "System",
                "text": "You are ZAMM, a chat program. Respond in first person."
              },
              {
                "role": "Human",
                "text": "This is a mock conversation."
              }
            ]
          },
          "temperature": 1.0
        },
        "response": {
          "completion": {
            "role": "AI",
            "text": "Mocking number 2."
          }
        },
        "tokens": {
          "prompt": 15,
          "response": 3,
          "total": 18
        }
      },
      ...,
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a00",
        "timestamp": "2024-01-16T08:50:00.738093890",
        ...,
        "response": {
          "completion": {
            "role": "AI",
            "text": "Mocking number 0."
          }
        },
        ...
      }
    ]
sideEffects:
  database:
    startStateDump: many-api-calls
    endStateDump: many-api-calls

```

Finally, at `src-tauri\api\sample-calls\get_api_calls-offset-empty.yaml`, we check that an offset past the total count results in another empty response:

```yaml
request:
  - get_api_calls
  - >
    {
      "offset": 20
    }
response:
  message: >
    []
sideEffects:
  database:
    startStateDump: many-api-calls
    endStateDump: many-api-calls
```

Unfortunately, when we actually run this, the app crashes on startup. We add an expect to `src-tauri/src/main.rs`:

```rs
  fn main() {
    #[cfg(all(debug_assertions, not(target_os = "windows")))]
    ts::export(
        collect_types![
            ...
        ],
        "../src-svelte/src/lib/bindings.ts",
    )
    .expect("Failed to export Specta bindings");
```

and get:

```
Failed to export Specta bindings: BigIntForbidden(i64)
```

We edit `src-tauri/src/commands/llms/get_api_calls.rs` accordingly:

```rs
async fn get_api_calls_helper(
    ...,
    offset: i32,
) -> ZammResult<Vec<LlmCall>> {
    ...
    let result: Vec<LlmCallRow> = llm_calls::table
        ...
        .offset(offset as i64)
        ...;
    ...
}

#[tauri::command(async)]
#[specta]
pub async fn get_api_calls(
    ...,
    offset: i32,
) -> ZammResult<Vec<LlmCall>> {
    get_api_calls_helper(..., offset).await
}

#[cfg(test)]
mod tests {
    ...

    #[derive(...)]
    struct GetApiCallsRequest {
        offset: i32,
    }

    ...
}

```

Now we are able to update the Specta bindings and start displaying this on the frontend.

#### Frontend

First, we do some small refactoring. We edit `src-svelte\src\routes\settings\SettingsSlider.svelte` to put the hover color:

```css
  .container:hover {
    background: var(--color-hover);
  }
```

into `src-svelte\src\routes\styles.css`

```css
:root {
  ...
  --color-hover: hsla(60, 100%, 50%, 0.2);
  ...
}
```

We do the same exact thing for `src-svelte\src\routes\settings\SettingsSwitch.svelte`, which uses the same hover color, so we don't need to edit `styles.css` again.

We then create `src-svelte\src\lib\EmptyPlaceholder.svelte`:

```svelte
<p class="empty atomic-reveal">
  <slot />
</p>

<style>
  .empty {
    color: var(--color-faded);
    font-size: 0.85rem;
    font-style: italic;
    text-align: center;
  }
</style>

```

and use this in `src-svelte\src\routes\chat\Chat.svelte`:

```svelte
<script lang="ts">
  ...
  import EmptyPlaceholder from "$lib/EmptyPlaceholder.svelte";
  ...
</script>

<InfoBox title="Chat" ...>
  ...
          <EmptyPlaceholder>
            This conversation is currently empty.<br />Get it started by typing
            a message below.
          </EmptyPlaceholder>
  ...
</InfoBox>
```

while removing the styling that previously existed there for the former `empty-conversation` class.

We now create `src-svelte\src\routes\api-calls\ApiCalls.svelte`:

```svelte
<script lang="ts">
  import { getApiCalls, type LlmCall } from "$lib/bindings";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";
  import InfoBox from "$lib/InfoBox.svelte";
  import Scrollable from "$lib/Scrollable.svelte";
  import { onMount } from "svelte";
  import EmptyPlaceholder from "$lib/EmptyPlaceholder.svelte";

  const PAGE_SIZE = 10;
  const MIN_MESSAGE_WIDTH = "5rem";

  let llmCalls: LlmCall[] = [];
  let llmCallsPromise: Promise<void> | undefined = undefined;
  let allCallsLoaded = false;
  let messageWidth = MIN_MESSAGE_WIDTH;

  const formatter = new Intl.DateTimeFormat(undefined, {
    year: "numeric",
    month: "numeric",
    day: "numeric",
    hour: "numeric",
    minute: "numeric",
    hour12: true,
  });

  export function formatTimestamp(timestamp: string): string {
    const timestampUTC = timestamp + "Z";
    const date = new Date(timestampUTC);
    return formatter.format(date);
  }

  function resizeMessageWidth() {
    const textContainer = document.querySelector(".text-container");
    if (!textContainer) {
      console.warn("Could not find text container for resize");
      return;
    }

    messageWidth = MIN_MESSAGE_WIDTH;

    requestAnimationFrame(() => {
      messageWidth = `${textContainer.clientWidth}px`;
    });
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
      })
      .catch((error) => {
        snackbarError(error);
      });
  }

  onMount(() => {
    resizeMessageWidth();
    window.addEventListener("resize", resizeMessageWidth);

    return () => {
      window.removeEventListener("resize", resizeMessageWidth);
    };
  });
</script>

<InfoBox title="LLM API Calls" fullHeight>
  <div class="container" style={`--message-width: ${messageWidth}`}>
    <div class="message header">
      <div class="text-container">Message</div>
      <div class="time">Time</div>
    </div>
    <div class="scrollable-container">
      <Scrollable on:bottomReached={loadApiCalls}>
        {#if llmCalls.length > 0}
          {#each llmCalls as call (call.id)}
            <a href={`/api-calls/${call.id}`}>
              <div class="message instance">
                <div class="text-container">
                  <div class="text">{call.response.completion.text}</div>
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
                started by using the chat functionality.
              </EmptyPlaceholder>
            </div>
            <div class="time"></div>
          </div>
        {/if}
      </Scrollable>
    </div>
  </div>
</InfoBox>

<style>
  .container {
    display: flex;
    flex-direction: column;
    gap: var(--internal-spacing);
    height: 100%;
  }

  .scrollable-container {
    flex: 1;
    display: flex;
    flex-direction: column;
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

  .message .time {
    width: 12rem;
    text-align: right;
  }
</style>

```

Note that we make use of the `--color-hover` and the `EmptyPlaceholder` that we just refactored above. We also make use of the resize trick from the Chat component to size the message quote appropriately. We changed from `<table>` to `<div>` here in the course of experimenting with ways to get the columns to resize properly, although in the future we may want to change it back to a proper table layout.

We eventually hardcode the `gap` style for the `.container` because the `--internal-spacing` variable isn't actually set anywhere in this component or its parents.

We originally try refactoring the `src-svelte\src\routes\api-calls\[slug]\ApiCall.svelte` file to export the date format:

```svelte
<script lang="ts" context="module">
  const formatter = new Intl.DateTimeFormat("en-US", {
    year: "numeric",
    month: "long",
    day: "numeric",
    hour: "numeric",
    minute: "numeric",
    second: "numeric",
    hour12: true,
  });

  export function formatTimestamp(timestamp: string): string {
    const timestampUTC = timestamp + "Z";
    const date = new Date(timestampUTC);
    return formatter.format(date);
  }
</script>

<script lang="ts">
  ...
  let humanTime: string | undefined = undefined;
  ...
  let apiCallPromise = getApiCall(...)
    .then((apiCall) => {
      humanTime = formatTimestamp(apiCall.timestamp);
      ...
    })
    ...
</script>
```

but then we realized that this date format is too verbose for the main list page, so we undo our changes.

We add an event to notify the parent when the bottom of the list is reached in `src-svelte\src\lib\FixedScrollable.svelte`:

```ts
  ...
  import { createEventDispatcher } from "svelte";

  ...
  const dispatchBottomReachedEvent = createEventDispatcher();

  ...

  onMount(() => {
    ...
    let bottomScrollObserver = new IntersectionObserver(
      (entries: IntersectionObserverEntry[]) => {
        intersectionCallback(bottomShadow)(entries);

        let indicator = entries[0];
        if (indicator.isIntersecting) {
          dispatchBottomReachedEvent("bottomReached");
        }
      },
    );
    bottomScrollObserver.observe(bottomIndicator);
    ...
  });
```

We bubble this up in `src-svelte\src\lib\Scrollable.svelte`. There are perhaps more [idiomatic](https://dev.to/mohamadharith/workaround-for-bubbling-custom-events-in-svelte-3khk) ways of [bubbling](https://github.com/baseballyama/svelte-preprocess-delegate-events) events up, but since this is our first time doing this and we are just getting the component set up, we minimize the number of failure points for easier debugging in case something doesn't work:

```svelte
<script lang="ts">
  ...
  const dispatchBottomReachedEvent = createEventDispatcher();

  ...

  function bottomReached() {
    dispatchBottomReachedEvent("bottomReached");
  }

  ...
</script>

<div ...>
  <FixedScrollable
    ...
    on:bottomReached={bottomReached}
    ...
  >
    <slot />
  </FixedScrollable>
</div>
```

We create a `src-svelte\src\routes\api-calls\+page.svelte` to display this list, although we don't actually link to it in the app yet:

```svelte
<script lang="ts">
  import ApiCalls from "./ApiCalls.svelte";
</script>

<ApiCalls />

```

As usual, we create `src-svelte\src\routes\api-calls\ApiCalls.stories.ts`:

```ts
import ApiCallsComponent from "./ApiCalls.svelte";
import MockFullPageLayout from "$lib/__mocks__/MockFullPageLayout.svelte";
import type { StoryFn, StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";

export default {
  component: ApiCallsComponent,
  title: "Screens/LLM Call/List",
  argTypes: {},
  decorators: [
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
  Component: ApiCallsComponent,
  props: args,
});

export const Empty: StoryObj = Template.bind({}) as any;
Empty.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
  sampleCallFiles: ["/api/sample-calls/get_api_calls-empty.yaml"],
};

export const Small: StoryObj = Template.bind({}) as any;
Small.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
  sampleCallFiles: ["/api/sample-calls/get_api_calls-small.yaml"],
};

export const Full: StoryObj = Template.bind({}) as any;
Full.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
  sampleCallFiles: ["/api/sample-calls/get_api_calls-full.yaml"],
};

```

We register these in `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "list"],
    variants: ["empty", "small", "full"],
  },
];
```

We notice that 10 items in the list is actually not enough to fill the whole thing up. The bottom indicator never gets triggered again because it stays visible after the new API calls are loaded. We should change the default page size. However, to do so, we'd also need to update our tests. We should write and commit a proper script to generate the test files for us. We create `src-tauri\api\sample-database-writes\many-api-calls\generate.py`:

```py
#!/usr/bin/env python3
#
# meant to be run from the src-tauri/ folder

from typing import Generator

yaml_preamble = """api_keys: []
llm_calls:
"""


def json_preamble(offset: int) -> str:
    return """request:
  - get_api_calls
  - >
    {
      "offset": {0}
    }
response:
  message: >
    [
""".replace(
        "{0}", str(offset)
    )


json_postamble = """
    ]
sideEffects:
  database:
    startStateDump: many-api-calls
    endStateDump: many-api-calls
"""


def generate_api_call_yaml(i: int) -> str:
    return f"""- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a{i:02d}
  timestamp: 2024-01-16T08:50:{i:02d}.738093890
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
        text: This is a mock conversation.
    temperature: 1.0
  response:
    completion:
      role: AI
      text: Mocking number {i}.
  tokens:
    prompt: 15
    response: 3
    total: 18
"""


def generate_api_call_sql(i: int) -> str:
    return """INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a{0:02d}','2024-01-16 08:50:{0:02d}.738093890','open_ai','gpt-4','gpt-4-0613',1.0,15,3,18,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"This is a mock conversation."}]}','{"role":"AI","text":"Mocking number {0}."}');""".replace(
        "{0:02d}", str(i).zfill(2)
    ).replace(
        "{0}", str(i)
    )


def generate_api_call_json(i: int) -> str:
    return """      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a{0:02d}",
        "timestamp": "2024-01-16T08:50:{0:02d}.738093890",
        "llm": {
          "name": "gpt-4-0613",
          "requested": "gpt-4",
          "provider": "OpenAI"
        },
        "request": {
          "prompt": {
            "type": "Chat",
            "messages": [
              {
                "role": "System",
                "text": "You are ZAMM, a chat program. Respond in first person."
              },
              {
                "role": "Human",
                "text": "This is a mock conversation."
              }
            ]
          },
          "temperature": 1.0
        },
        "response": {
          "completion": {
            "role": "AI",
            "text": "Mocking number {0}."
          }
        },
        "tokens": {
          "prompt": 15,
          "response": 3,
          "total": 18
        }
      }""".replace(
        "{0:02d}", str(i).zfill(2)
    ).replace(
        "{0}", str(i)
    )


def db_index_sequence(n: int) -> Generator[int, None, None]:
    for i in range(0, n, 2):
        yield i
    for i in range(1, n, 2):
        yield i


def generate_yaml(n: int) -> None:
    with open("api/sample-database-writes/many-api-calls/dump.yaml", "w") as f:
        f.write(yaml_preamble)
        for i in db_index_sequence(n):
            f.write(generate_api_call_yaml(i))


def generate_sql(n: int) -> None:
    with open("api/sample-database-writes/many-api-calls/dump.sql", "w") as f:
        api_calls = []
        for i in db_index_sequence(n):
            api_calls.append(generate_api_call_sql(i))
        f.write("\n".join(api_calls))


def generate_json(test_name: str, offset: int, start: int, end: int) -> None:
    with open(f"api/sample-calls/get_api_calls-{test_name}.yaml", "w") as f:
        f.write(json_preamble(offset))
        api_calls = []
        for i in range(start, end, -1):
            api_calls.append(generate_api_call_json(i))
        api_calls_json = ",\n".join(api_calls)
        f.write(api_calls_json)
        f.write(json_postamble)


def generate_json_sample_files(n: int, page_size: int) -> None:
    generate_json(
        "full",
        offset=0,
        start=n - 1,
        end=n - page_size - 1,
    )
    generate_json(
        "offset",
        offset=page_size,
        start=n - page_size - 1,
        end=-1,
    )


if __name__ == "__main__":
    n = 13
    generate_yaml(n)
    generate_sql(n)
    generate_json_sample_files(n, 10)

```

and verify that the repo remains unchanged when we run it.

Next, we edit `src-tauri\src\commands\llms\get_api_calls.rs` to change the page size:

```rs
const PAGE_SIZE: i64 = 50;

async fn get_api_calls_helper(
    ...
) -> ZammResult<Vec<LlmCall>> {
    ...
    let result: Vec<LlmCallRow> = llm_calls::table
        ...
        .limit(PAGE_SIZE)
        ...;
    ...
}
```

We edit `src-tauri\api\sample-database-writes\many-api-calls\generate.py` accordingly:

```py
if __name__ == "__main__":
    n = 60
    ...
    generate_json_sample_files(n, 50)

```

Unfortunately our tests are failing. Upon further inspection, this is because on Windows, Python inserts a "\r". We see that we [must write](https://stackoverflow.com/a/7077425) the files in binary mode. We change all our output functions accordingly, anywhere there is a `f.write`; for example:

```py
def generate_json(...) -> None:
    with open(..., "wb") as f:
        f.write(json_preamble(offset).encode())
        ...
        f.write(api_calls_json.encode())
        f.write(json_postamble.encode())
```

With this, the remaining failing test that needs to be updated is `src-tauri\api\sample-calls\get_api_calls-offset-empty.yaml`:

```yaml
request:
  - get_api_calls
  - >
    {
      "offset": 100
    }
...

```

Upon manual testing, we find that the API calls list is not loading new API calls when we scroll to the bottom. We realize that this is because promises, unlike timeouts, don't reset to be undefined after they are resolved. We edit `src-svelte\src\routes\api-calls\ApiCalls.svelte` accordingly, remember to update the page size that we had forgotten to edit after changing the backend earlier:

```ts
...

const PAGE_SIZE = 50;

...

function loadApiCalls() {
    ...

    llmCallsPromise = getApiCalls(llmCalls.length)
      .then((newCalls) => {
        ...
        llmCallsPromise = undefined;
      })
      ...
  }
```

We then add the second API call to `src-svelte\src\routes\api-calls\ApiCalls.stories.ts`:

```ts
export const Full: StoryObj = Template.bind({}) as any;
Full.parameters = {
  ...,
  sampleCallFiles: [
    "/api/sample-calls/get_api_calls-full.yaml",
    "/api/sample-calls/get_api_calls-offset.yaml",
  ],
};
```

Now it works as expected. We should turn this manual test into an automated one. We do so by creating `src-svelte\src\routes\api-calls\ApiCalls.playwright.test.ts` based on the Switch and Slider playwright tests:

```ts
import {
  type Browser,
  type BrowserContext,
  chromium,
  expect,
  type Page,
  type Frame,
  type Locator,
} from "@playwright/test";
import { afterAll, beforeAll, describe, test } from "vitest";

const DEBUG_LOGGING = false;

describe("Api Calls endless scroll test", () => {
  let page: Page;
  let frame: Frame;
  let browser: Browser;
  let context: BrowserContext;

  beforeAll(async () => {
    browser = await chromium.launch({ headless: true });
    context = await browser.newContext();
    page = await context.newPage();

    if (DEBUG_LOGGING) {
      page.on("console", (msg) => {
        console.log(msg);
      });
    }
  });

  afterAll(async () => {
    await browser.close();
  });

  const getScrollElement = async () => {
    const url = `http://localhost:6006/?path=/story/screens-llm-call-list--full`;
    await page.goto(url);

    const maybeFrame = page.frame({ name: "storybook-preview-iframe" });
    if (!maybeFrame) {
      throw new Error("Could not find Storybook iframe");
    }
    frame = maybeFrame;

    const apiCallsScrollElement = frame.locator(".scroll-contents");
    return { apiCallsScrollElement };
  };

  const expectLastMessage = async (
    apiCallsScrollElement: Locator,
    expectedValue: string,
  ) => {
    // the actual last child is the bottom indicator that triggers the shadow and the
    // auto-load
    const lastMessageContainer = apiCallsScrollElement.locator("a:nth-last-child(2) .message.instance .text-container");
    await expect(lastMessageContainer).toHaveText(expectedValue);
  };

  test(
    "loads more messages when scrolled to end",
    async () => {
      const { apiCallsScrollElement } = await getScrollElement();
      await expectLastMessage(apiCallsScrollElement, "Mocking number 10.");

      await apiCallsScrollElement.evaluate((el) => {
        el.scrollTop = el.scrollHeight;
      });
      await expectLastMessage(apiCallsScrollElement, "Mocking number 0.");
    },
    { retry: 2 },
  );
});

```

Unfortunately, for some reason, this auto-wait ends up being flaky. We try a manual wait, which fares no better:

```ts
  const expectLastMessage = async (
    apiCallsScrollElement: Locator,
    expectedValue: string,
  ) => {
    ...
    const lastMessageContainer = ...;
    const lastMessageStr = await lastMessageContainer.evaluate((el) =>
      el.textContent,
    );
    expect(lastMessageStr).not.toBeNull();
    expect(lastMessageStr).toEqual(expectedValue.toString());
  };

  test(
    "loads more messages when scrolled to end",
    async () => {
      ...

      await apiCallsScrollElement.evaluate((el) => {
        ...
      });
      await page.waitForTimeout(2000);
      await expectLastMessage(apiCallsScrollElement, "Mocking number 0.");
    },
    ...
  );
```

Upon further testing in headed mode, it turns out that the test passes as soon as the bottom drawer is closed, allowing the elements to be visible. We revert to the auto-wait and close the drawer automatically as we do in the Storybook tests:

```ts
const getScrollElement = async () => {
    ...
    await page.goto(url);
    await page.locator("button[title='Hide addons [A]']").click();

    ...
  };
```

Finally, the timestamps shown on the frontend are identical because the frontend list page does not include the seconds. As such, we edit `src-tauri\api\sample-database-writes\many-api-calls\generate.py` to change all `2024-01-16 08:50:{0:02d}.738093890` into `2024-01-16 08:{0:02d}:50.738093890`.

To make sure that this is referenced elsewhere and not forgotten, we edit `src-tauri\Makefile`:

```Makefile
test-files:
	python3 api/sample-database-writes/many-api-calls/generate.py

```

We finally link to these new pages in the app itself. We just have to edit `src-svelte\src\routes\SidebarUI.svelte` to link to the API calls list page, because that page already links to the individual API call pages:

```ts
  ...
  import IconDatabase from "~icons/fa6-solid/database";
  ...

  const routes: App.Route[] = [
    ...
    {
      name: "API Calls",
      path: "/api-calls",
      icon: IconDatabase,
    },
    ...
  ];
```

However, we notice that the sidebar indicator is now off. We do a little bit more refactoring:

```svelte
<script lang="ts">
  ...

  function routeMatches(sidebarRoute: string, pageRoute: string) {
    if (sidebarRoute === "/") {
        return pageRoute === sidebarRoute;
      }
      return pageRoute.includes(sidebarRoute);
  }

  function setIndicatorPosition(newRoute: string) {
    const routeIndex = routes.findIndex((r) => routeMatches(r.path, newRoute));
    ...
  }

  ...
</script>

<header>
  ...
  <nav>
    ...
    {#each routes as route (route.path)}
      <a
        aria-current={routeMatches(route.path, currentRoute) ? "page" : undefined}
        ...
      >
        ...
      </a>
    {/each}
  </nav>
</header>
```

We edit `src-svelte\src\routes\SidebarUI.test.ts` to rename the existing tests and add new ones. We can't add them to the existing test because the existing ones assume the same starting URL, and rendering it twice will result in multiple sidebars with the same links existing.

```ts
describe("Sidebar", () => {
  test("requires exact match for homepage", () => {
    render(SidebarUI, {
      currentRoute: "/",
      dummyLinks: true,
    });
    const homeLink = screen.getByTitle("Dashboard");
    const apiCallsLink = screen.getByTitle("API Calls");
    expect(homeLink).toHaveAttribute("aria-current", "page");
    expect(apiCallsLink).not.toHaveAttribute("aria-current", "page");
  });

  test("highlights right icon for sub-paths", () => {
    render(SidebarUI, {
      currentRoute: "/api-calls/1234",
      dummyLinks: true,
    });
    const homeLink = screen.getByTitle("Dashboard");
    const apiCallsLink = screen.getByTitle("API Calls");
    expect(homeLink).not.toHaveAttribute("aria-current", "page");
    expect(apiCallsLink).toHaveAttribute("aria-current", "page");
  });
});

describe("Sidebar interactions", () => {
  ...
});
```

Now we notice various layout problems. We edit `src-svelte\src\routes\styles.css` for a class that will be repeated frequently:

```css
.full-height {
  flex: 1;
  display: flex;
  flex-direction: column;
}
```

We use this in `src-svelte\src\routes\PageTransition.svelte` so that the bottom padding for the individual API call page is properly rendered:

```svelte
{#key currentRoute}
  {#if ready}
    <div
      class="transition-container full-height"
      ...
    >
      ...
    </div>
  {/if}
{/key}

<style>
  .transition-container {
    ...
    min-height: 100vh;
    ...
  }
</style>

```

We continue onto `src-svelte\src\lib\InfoBox.svelte`, where we remove `flex: 1;` from the `.container` itself unless it is also `full-height`. We also remove the former custom styles for `.container.full-height .info-box`:

```css
  .container {
    ...
  }

  .container.full-height,
  .container.full-height .border-container,
  .container.full-height .info-box,
  .container.full-height .info-content {
    flex: 1;
    display: flex;
    flex-direction: column;
  }
```

While we have repeated the `full-height` styles from `styles.css`, we do this so that we don't have to add code to set the `full-height` class on every relevant child class. Instead, the parent container being `full-height` will automatically apply the styles to all children.

We make use of this in `src-svelte/src/routes/api-calls/ApiCalls.svelte`, and remove the now-duplicated styles from the divs. We also find that the time column needs a little more width to not run into a second line. We could use JavaScript to find the maximum width and adjust to that, but that seems to be a bit too much work for now.

```svelte
<InfoBox title="LLM API Calls" fullHeight>
  <div class="container full-height" ...>
    ...
    <div class="scrollable-container full-height">
      ...
    </div>
  </div>
</InfoBox>

<style>
  .container {
    ...
  }

  .scrollable-container {
    ...
  }

  ...

  .message .time {
    width: 12.5rem;
    ...
  }
</style>
```

We edit `src-svelte\src\routes\chat\Chat.svelte` to the same effect:

```svelte
<InfoBox title="Chat" fullHeight>
  <div class="chat-container ... full-height">
    ...
  </div>
</div>
```

Finally, we do the same for `src-svelte/src/routes/credits/Credits.svelte`, removing the `.credits-container` CSS block entirely and removing duplicate lines from `.container`:

```svelte
<div class="container full-height">
  <div class="credits-container full-height">
    ...
  </div>

  ...
</div>

<style>
  .container {
    ...
  }

  ...
</style>
```

We add a test to `webdriver\test\specs\e2e.test.js`:

```js
describe("App", function () {
  ...

  it("should allow navigation to the API calls page", async function () {
    this.retries(2);
    await findAndClick('a[title="API Calls"]');
    await findAndClick('a[title="Dashboard"]');
    await findAndClick('a[title="API Calls"]');
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("api-calls", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  ...
});
```

However, when trying to actually run the end-to-end test, we end up with the error

```
node:internal/event_target:1054
  process.nextTick(() => { throw err; });
                           ^
Error: The following routes were marked as prerenderable, but were not prerendered because they were not found while crawling your app:
  - /api-calls/[slug]
```

Trying to disable prerendering gives us

```
> Using @sveltejs/adapter-static
  @sveltejs/adapter-static: all routes must be fully prerenderable, but found the following routes that are dynamic:
    - src/routes/api-calls/[slug]

  You have the following options:
    - set the `fallback` option — see https://kit.svelte.dev/docs/single-page-apps#usage for more info.
    - add `export const prerender = true` to your root `+layout.js/.ts` or `+layout.server.js/.ts` file. This will try to prerender all pages.
    - add `export const prerender = true` to any `+server.js/ts` files that are not fetched by page `load` functions.
    - adjust the `prerender.entries` config option (routes with parameters are not part of entry points by default) — see https://kit.svelte.dev/docs/configuration#prerender for more info.
    - pass `strict: false` to `adapter-static` to ignore this error. Only do this if you are sure you don't need the routes in question in your final app, as they will be unavailable. See https://github.com/sveltejs/kit/tree/main/packages/adapter-static#strict for more info.

  If this doesn't help, you may need to use a different adapter. @sveltejs/adapter-static can only be used for sites that don't need a server for dynamic rendering, and can run on just a static file server.
  See https://kit.svelte.dev/docs/page-options#prerender for more details
error during build:
Error: Encountered dynamic routes
    at adapt (file:///root/zamm/node_modules/@sveltejs/adapter-static/index.js:38:12)
    at adapt (file:///root/zamm/node_modules/@sveltejs/kit/src/core/adapt/index.js:38:8)
    at finalise (file:///root/zamm/node_modules/@sveltejs/kit/src/exports/vite/index.js:904:13)
    ...
```

However, according to the official documentation [here](https://kit.svelte.dev/docs/page-options#prerender-when-not-to-prerender):

> Note that you can still prerender pages that load data based on the page's parameters, such as a `src/routes/blog/[slug]/+page.svelte` route.

Taking note of [the documentation](https://kit.svelte.dev/docs/configuration#prerender) on configuring SvelteKit, we edit `src-svelte/svelte.config.js`:

```js
/** @type {import('@sveltejs/kit').Config} */
const config = {
  ...,
  kit: {
    ...,
    prerender: {
      crawl: false,
      entries: ["*", "/api-calls/[slug]"],
    },
  },
};

```

Now the end-to-end screenshot can be taken, but the Storybook ones need some touch ups. We edit `src-svelte/src/routes/AnimationControl.svelte` to make use of the new `full-height` class:

```svelte
<div class="container full-height" ...>
  ...
</div>
```

and do the same for `src-svelte/src/lib/__mocks__/MockFullPageLayout.svelte`:

```svelte
<div class="storybook-wrapper full-height">
  ...
</div>
```

We edit the new tests in `src-svelte/src/routes/storybook.test.ts` to be a full body screenshot (not a full page one because we don't want to include Storybook elements):

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "individual"],
    ...,
    screenshotEntireBody: true,
  },
  {
    path: ["screens", "llm-call", "list"],
    ...,
    screenshotEntireBody: true,
  },
];
```

Finally, the narrow individual LLM call is producing a screenshot with gray at the bottom due to Playwright clipping out anything that is outside the current viewport. We take inspiration from [this workaround](https://github.com/microsoft/playwright/issues/12962#issuecomment-1077674428), and edit `src-svelte/src/routes/storybook.test.ts`, where the value of 60 was chosen to reduce flakiness for both the screenshots that don't need resizing (resizing too much produces a line at the top of the screenshot) and the one that does need it, but sometimes doesn't get resized enough:

```ts
  const takeScreenshot = async (
    frame: Frame,
    page: Page,
    screenshotEntireBody?: boolean,
  ) => {
    let locatorStr = screenshotEntireBody
      ? "body"
      : "#storybook-root > :first-child";
    const elementClass = await frame.locator(locatorStr).getAttribute("class");
    if (elementClass?.includes("storybook-wrapper")) {
      locatorStr = ".storybook-wrapper > :first-child > :first-child";
    }

    const currentViewport = page.viewportSize();
    if (currentViewport === null) {
      throw new Error("Viewport is null");
    }

    const elementLocator = frame.locator(locatorStr);
    await elementLocator.waitFor({ state: "visible"});
    const elementHeight = await elementLocator.evaluate((el) => el.clientHeight);
    const storybookHeight = 60; // height taken up by Storybook elements
    const effectiveViewportHeight = currentViewport.height - storybookHeight;
    const extraHeightNeeded = elementHeight - effectiveViewportHeight;

    if (extraHeightNeeded > 0) {
      await page.setViewportSize({
        width: currentViewport.width,
        height: currentViewport.height + extraHeightNeeded
      })
    }

    return await elementLocator.screenshot();
  };

  ...

  for (const config of components) {
    ...
          const screenshot = await takeScreenshot(
            ...,
            page,
            ...,
          );
    ...
          if (variantConfig.assertDynamic !== undefined) {
            ...
            const newScreenshot = await takeScreenshot(
              ...,
              page,
              ...
            );
            ...
          }
    ...
  }
```

Our CI tests are failing because the server localizes to a different timezone. We fix this by editing `src-svelte/src/routes/api-calls/ApiCalls.svelte` to allow specific locales and timezones to be passed in:

```ts
  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
  ...

  const formatter = new Intl.DateTimeFormat(dateTimeLocale, {
    ...,
    timeZone,
  });
```

We also edit `src-svelte/src/routes/api-calls/ApiCalls.stories.ts` to make use of these new options:

```ts
export const Small: StoryObj = Template.bind({}) as any;
Small.args = {
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh"
};
...

export const Full: StoryObj = Template.bind({}) as any;
Full.args = {
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh"
};
...
```

We do the same thing for `src-svelte/src/routes/api-calls/[slug]/ApiCall.svelte`:

```ts
  export let id: string;
  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;

  const formatter = new Intl.DateTimeFormat(dateTimeLocale, {
    year: "numeric",
    month: "long",
    day: "numeric",
    hour: "numeric",
    minute: "numeric",
    second: "numeric",
    hour12: true,
    timeZone,
  });
```

and `src-svelte/src/routes/api-calls/[slug]/ApiCall.stories.ts`:

```ts
export const Narrow: StoryObj = Template.bind({}) as any;
Narrow.args = {
  ...,
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
...

export const Wide: StoryObj = Template.bind({}) as any;
Wide.args = {
  ...,
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
...
```

It turns out that the screenshot for the static background is now also flaky on CI. We edit `src-svelte/src/routes/storybook.test.ts` to only do the window resizing when needed:

```ts
interface VariantConfig {
  ...
  resizeWindow?: boolean;
  ...
}

const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      ...,
      {
        name: "full-message-width",
        resizeWindow: true,
        additionalAction: async (frame: Frame, page: Page) => {
          await new Promise((r) => setTimeout(r, 1000));
          // need to do a manual scroll because Storybook resize messes things up on CI
          const scrollContents = frame.locator(".scroll-contents");
          await scrollContents.focus();
          await page.keyboard.press("End");
        },
      },
      ...
    ],
    ...
  },
  ...,
  {
    path: ["screens", "llm-call", "individual"],
    variants: [
      {
        name: "narrow",
        resizeWindow: true,
      },
      "wide",
    ],
    screenshotEntireBody: true,
  },
  ...
];

...

describe.concurrent("Storybook visual tests", () => {
  ...

  const takeScreenshot = async (
    ...,
    resizeWindow: boolean,
    ...
  ) => {
    ...
    if (elementClass?.includes("storybook-wrapper")) {
      ...
    }
    const elementLocator = frame.locator(locatorStr);
    await elementLocator.waitFor({ state: "visible" });

    if (resizeWindow) {
      const currentViewport = page.viewportSize();
      ...

      if (extraHeightNeeded > 0) {
        ...
      }
    }

    return await elementLocator.screenshot();
  };

  ...

  for (const config of components) {
    ...
          const screenshot = await takeScreenshot(
            ...,
            variantConfig.resizeWindow ?? false,
            ...
          );
    ...
            const newScreenshot = await takeScreenshot(
              ...,
              variantConfig.resizeWindow ?? false,
              ...
            );
    ...
  }
});
```

Finally, we get a new failure with "new-message-sent", where the test is timing out because the Snackbar error message close button is no longer being shown. We see the error being logged

```
Uncaught (in promise) TypeError: message.includes is not a function
    warn supress-warnings.js:7
    snackbarError Snackbar.svelte:25
    sendChatMessage Chat.svelte:65
    ...
```

Upon further logging in `Chat.svelte`, we see the error

```
TypeError: playback is undefined
    mockInvokeFn invoke.ts:19
    chat bindings.ts:38
    sendChatMessage Chat.svelte:61
    submitChat Form.svelte:30
    ...
```

We edit `src-svelte/src/routes/chat/Chat.stories.ts` to use the TauriInvokeDecorator. It is unclear how this was ever working before:

```ts
...
import TauriInvokeDecorator from "$lib/__mocks__/invoke";
...

export default {
  component: ChatComponent,
  title: "Screens/Chat/Conversation",
  ...,
  decorators: [
    ...,
    TauriInvokeDecorator,
    ...
  ],
};
```

On CI, we also run into the pre-commit error

```
ESLint: 9.0.0

Error: Could not find config file.
    at locateConfigFileToUse (/home/runner/.cache/pre-commit/repothq_9gxk/node_env-system/lib/node_modules/eslint/lib/eslint/eslint.js:349:21)
    at async calculateConfigArray (/home/runner/.cache/pre-commit/repothq_9gxk/node_env-system/lib/node_modules/eslint/lib/eslint/eslint.js:384:49)
    at async ESLint.lintFiles (/home/runner/.cache/pre-commit/repothq_9gxk/node_env-system/lib/node_modules/eslint/lib/eslint/eslint.js:814:25)
    at async Object.execute (/home/runner/.cache/pre-commit/repothq_9gxk/node_env-system/lib/node_modules/eslint/lib/cli.js:461:23)
    at async main (/home/runner/.cache/pre-commit/repothq_9gxk/node_env-system/lib/node_modules/eslint/bin/eslint.js:165:22)
```

This is likely because we have a beta version of eslint set in the pre-commit config. However, even changing the `rev` doesn't automatically fix things. We edit `.pre-commit-config.yaml` to simply move the hook to the local repo:

```yaml
repos:
  ...
  - repo: local
    hooks:
      ...
      - id: eslint
        name: eslint
        entry: yarn eslint --fix --max-warnings=0
        language: system
        types: [file]
        files: \.(js|ts|svelte)$
        exclude: src-svelte/src/lib/sample-call.ts
      ...
```

##### Column resize adjustment

We realized that the fixed width of 12.5rem is sometimes not enough for some dates in some date formats. We edit `src-svelte\src\routes\api-calls\ApiCalls.svelte` to be more dynamically adjustable:

```svelte
<script lang="ts">
  ...
  const MIN_COLUMN_WIDTH = "5rem";

  ...
  let messageWidth = MIN_COLUMN_WIDTH;
  let timeWidth = MIN_COLUMN_WIDTH;
  let headerMessageWidth = MIN_COLUMN_WIDTH;

  ...

  function getWidths(selector: string) {
    const elements = document.querySelectorAll(selector);
    return Array.from(elements).map((el) => el.getBoundingClientRect().width).filter((width) => width > 0);
  }

  function resizeMessageWidth() {
    messageWidth = MIN_COLUMN_WIDTH;
    // time width doesn't need a reset because it never decreases

    setTimeout(() => {
      const textWidths = getWidths(".text-container");
      const timeWidths = getWidths(".time");
      const minTextWidth = Math.floor(Math.min(...textWidths));
      messageWidth = `${minTextWidth}px`;
      const maxTimeWidth = Math.ceil(Math.max(...timeWidths));
      timeWidth = `${maxTimeWidth}px`;

      headerMessageWidth = messageWidth;
    }, 10);
  }

  function loadApiCalls() {
    ...

    llmCallsPromise = getApiCalls(llmCalls.length)
      .then((newCalls) => {
        ...

        requestAnimationFrame(resizeMessageWidth);
      })
      ...;
  }

  ...

  $: minimumWidths = `--message-width: ${messageWidth}; ` + `--header-message-width: ${headerMessageWidth}; ` + `--time-width: ${timeWidth}`;
</script>

<InfoBox title="LLM API Calls" fullHeight>
  <div class="container full-height" style={minimumWidths}>
    <div class="message header">
      <div class="text-container">
        <div class="text">Message</div>
      </div>
      ...
    </div>
  </div>
</InfoBox>

<style>
  .container {
    gap: 0.25rem;
  }

  ...

  .message.header .text-container .text {
    max-width: var(--header-message-width);
  }

  .message .time {
    min-width: var(--time-width);
    box-sizing: border-box;
    ...
  }
</style>
```

Note that:

- We're using `getBoundingClientRect` instead of `clientWidth` because the latter can be [rounded off](https://stackoverflow.com/a/48309564) in the opposite direction of what we want at any given time
- We also have a separate `--header-message-width` to avoid the header column from flickering much when more content is loaded, because header flicker is much more visible than content flicker.

Finally, as one more nit, we edit

```css
  .message.instance {
    ...
    height: 1.62rem;
    box-sizing: border-box;
  }
```

because we find that on Linux Mint, the rows with Khmer text are taller than the rows without.

During testing, we find that the empty page now has a noticeably shrunken time column due to the lack of any time values. We therefore restore a different minimum width to the time column, and rename `MIN_COLUMN_WIDTH` back to `MIN_MESSAGE_WIDTH`:

```ts
  const MIN_MESSAGE_WIDTH = "5rem";
  const MIN_TIME_WIDTH = "12.5rem";

  ...
  let messageWidth = MIN_MESSAGE_WIDTH;
  let timeWidth = MIN_TIME_WIDTH;
  let headerMessageWidth = MIN_MESSAGE_WIDTH;
```

## Restoring conversation

### Refactoring Chat.svelte

We want to make it possible to restore conversations from previous API calls. To do this, we first refactor `src-svelte\src\routes\chat\Chat.svelte` to use a Svelte store:

```svelte
<script lang="ts" context="module">
  import { writable } from "svelte/store";

  export const conversation = writable<ChatMessage[]>([
    {
      role: "System",
      text: "You are ZAMM, a chat program. Respond in first person.",
    },
  ]);
</script>

<script lang="ts">
  ...

  function appendMessage(message: ChatMessage) {
    conversation.update((messages) => [...messages, message]);
    ...
  }

  async function sendChatMessage(message: string) {
    ...

    try {
      let llmCall = await chat(..., $conversation);
      ...
    } ...
  }
</script>

<InfoBox title="Chat" ...>
  <div ...>
    <Scrollable
      ...
    >
      <div class="composite-reveal" role="list">
        {#if $conversation.length > 1}
          {#each $conversation.slice(1) as message, i (i)}
            ...
          {/each}
          ...
        {/if}
      </div>
    </Scrollable>

    ...
  </div>
</InfoBox>

...
```

We remove `src-svelte/src/routes/chat/PersistentChat.svelte` because it is no longer needed, and update `src-svelte/src/routes/chat/PersistentChatView.svelte` to use the regular `Chat` component instead:

```svelte
<script lang="ts">
  import Chat from "./Chat.svelte";

  ...
</script>

...

{#if visible}
  <Chat />
{/if}

```

We delete `src-svelte/src/routes/chat/PersistentChat.test.ts` and move its code to `src-svelte\src\routes\chat\Chat.test.ts` instead:

```ts
...
import PersistentChatView from "./PersistentChatView.svelte";
...

...

describe("Chat conversation", () => {
  ...

  test("persists a conversation after returning to it", async () => {
    render(PersistentChatView, {});
    await sendChatMessage(
      "Hello, does this work?",
      "../src-tauri/api/sample-calls/chat-start-conversation.yaml",
    );

    await userEvent.click(screen.getByRole("button", { name: "Remount" }));
    await waitFor(() => {
      expect(screen.getByText("Hello, does this work?")).toBeInTheDocument();
    });
  });
});

```

We get multiple failing tests:

```
 FAIL  src/routes/chat/Chat.test.ts > Chat conversation > persists a conversation after returning to it  
TestingLibraryElementError: Found multiple elements with the text: Hello, does this work?

Here are the matching elements:

Ignored nodes: comments, script, style
<p>
  Hello, does this work?
</p>

Ignored nodes: comments, script, style
<p>
  Hello, does this work?
</p>

Ignored nodes: comments, script, style
<p>
  Hello, does this work?
</p>

(If this is intentional, then use the `*AllBy*` variant of the query (like `queryAllByText`, `getAllByText`, or `findAllByText`)).

...

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Errors ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

Vitest caught 3 unhandled errors during the test run.
This might cause false positive tests. Resolve unhandled errors to make sure your tests are not affected.

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Rejection ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
TypeError: message.includes is not a function
 ❯ Console.console.warn ../node_modules/svelte-markdown/src/supress-warnings.js:8:17
 ❯ Module.snackbarError src/lib/snackbar/Snackbar.svelte:25:13
     23|
     24|   export function snackbarError(msg: string) {
     25|     console.warn(msg);
       |             ^
     26|     animateDurationMs = baseAnimationDurationMs;
     27|     const id = nextId++;
 ❯ sendChatMessage src/routes/chat/Chat.svelte:69:7
 ❯ HTMLFormElement.submitChat src/routes/chat/Form.svelte:30:7
 ❯ HTMLFormElement.<anonymous> ../../../../../../Amos%20Ng/Documents/projects/zamm-dev/zamm/src-svelte/node_modules/.vitest/deps/chunk-X2XOW6IX.js:482:15
 ❯ HTMLFormElement.callTheUserObjectsOperation ../node_modules/jsdom/lib/jsdom/living/generated/EventListener.js:26:30
 ❯ innerInvokeEventListeners ../node_modules/jsdom/lib/jsdom/living/events/EventTarget-impl.js:350:25    
 ❯ invokeEventListeners ../node_modules/jsdom/lib/jsdom/living/events/EventTarget-impl.js:286:3
 ❯ HTMLFormElementImpl._dispatch ../node_modules/jsdom/lib/jsdom/living/events/EventTarget-impl.js:233:9
 ❯ fireAnEvent ../node_modules/jsdom/lib/jsdom/living/helpers/events.js:18:36

This error originated in "src/routes/chat/Chat.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
The latest test that might've caused the error is "won't send multiple messages at once". It might mean one of the following:
- The error was thrown, while Vitest was running this test.
- If the error occurred after the test had been completed, this was the last documented test before it was thrown.

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Rejection ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
TypeError: message.includes is not a function
 ❯ Console.console.warn ../node_modules/svelte-markdown/src/supress-warnings.js:8:17
 ❯ Module.snackbarError src/lib/snackbar/Snackbar.svelte:25:13
     23|
     24|   export function snackbarError(msg: string) {
     25|     console.warn(msg);
       |             ^
     26|     animateDurationMs = baseAnimationDurationMs;
     27|     const id = nextId++;
 ❯ sendChatMessage src/routes/chat/Chat.svelte:69:7
 ❯ HTMLFormElement.submitChat src/routes/chat/Form.svelte:30:7
 ❯ HTMLFormElement.<anonymous> ../../../../../../Amos%20Ng/Documents/projects/zamm-dev/zamm/src-svelte/node_modules/.vitest/deps/chunk-X2XOW6IX.js:482:15
 ❯ HTMLFormElement.callTheUserObjectsOperation ../node_modules/jsdom/lib/jsdom/living/generated/EventListener.js:26:30
 ❯ innerInvokeEventListeners ../node_modules/jsdom/lib/jsdom/living/events/EventTarget-impl.js:350:25    
 ❯ invokeEventListeners ../node_modules/jsdom/lib/jsdom/living/events/EventTarget-impl.js:286:3
 ❯ HTMLFormElementImpl._dispatch ../node_modules/jsdom/lib/jsdom/living/events/EventTarget-impl.js:233:9 
 ❯ fireAnEvent ../node_modules/jsdom/lib/jsdom/living/helpers/events.js:18:36

This error originated in "src/routes/chat/Chat.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
The latest test that might've caused the error is "persists a conversation after returning to it". It might mean one of the following:
- The error was thrown, while Vitest was running this test.
- If the error occurred after the test had been completed, this was the last documented test before it was thrown.

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Rejection ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
TypeError: latestMessage?.resizeBubble is not a function
 ❯ Timeout._onTimeout src/routes/chat/Chat.svelte:48:28
     46|     setTimeout(async () => {
     47|       const latestMessage = messageComponents[messageComponents.length - 1];
     48|       await latestMessage?.resizeBubble(conversationWidthPx);
       |                            ^
     49|       growable?.scrollToBottom();
     50|     }, 10);
 ❯ listOnTimeout node:internal/timers:573:17
 ❯ processTimers node:internal/timers:514:7

This error originated in "src/routes/chat/Chat.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 Test Files  1 failed (1)
      Tests  2 failed | 1 passed (3)
     Errors  3 errors
   Start at  15:58:44
   Duration  3.21s
```

We realize that we don't reset the conversation for tests, so we edit `src-svelte\src\routes\chat\Chat.svelte`, changing where `ChatMessage` is imported for consistency:

```svelte
<script lang="ts" context="module">
  ...
  import { type ChatMessage } from "$lib/bindings";

  const initialMessage: ChatMessage = {
    role: "System",
    text: "You are ZAMM, a chat program. Respond in first person.",
  };

  export const conversation = writable<ChatMessage[]>([initialMessage]);

  export function resetConversation() {
    conversation.set([initialMessage]);
  }
</script>

<script lang="ts">
  ...
  import { chat } from "$lib/bindings";
  ...
</script>
```

We edit `src-svelte\src\routes\chat\Chat.test.ts` as well to use this:

```ts
...
import Chat, { resetConversation } from "./Chat.svelte";
...

describe("Chat conversation", () => {
  ...

  afterEach(() => {
    ...
    resetConversation();
  });

  ...
});
```

Now our tests pass. During commit, we get the error

```
c:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-svelte\src\routes\chat\+page.svelte:2:30
Error: Cannot find module './PersistentChat.svelte' or its corresponding type declarations. (ts)
<script lang="ts">
  import PersistentChat from "./PersistentChat.svelte";
</script>
```

We edit `src-svelte\src\routes\chat\+page.svelte` as well to use the regular `Chat` component:

```svelte
<script lang="ts">
  import Chat from "./Chat.svelte";
</script>

<Chat />
```

We try to add a test to `src-svelte\src\routes\chat\Chat.test.ts` to check that the store can be successfully controlled by another component:

```ts
...
import ..., { conversation, ... } from "./Chat.svelte";
...

describe("Chat conversation", () => {
  ...

  test("listens for updates to conversation store", async () => {
    render(Chat, {});
    expect(screen.queryByText("Hello, does this work?")).not.toBeInTheDocument();
    conversation.set([
      {
        "role": "System",
        "text": "You are ZAMM, a chat program. Respond in first person."
      },
      {
        "role": "Human",
        "text": "Hello, does this work?"
      },
      {
        "role": "AI",
        "text": "Yes, it works. How can I assist you today?"
      },
    ]);
    await waitFor(() => {
      expect(screen.getByText("Hello, does this work?")).toBeInTheDocument();
    });
    await sendChatMessage(
      "Tell me something funny.",
      "../src-tauri/api/sample-calls/chat-continue-conversation.yaml",
    );
  });
});

```

It fails with two errors, one being that the last message from the AI is duplicated:

```
stderr | src/routes/chat/Chat.test.ts > Chat conversation > listens for updates to conversation store    
No matching call found for ["chat",{"provider":"OpenAI","llm":"gpt-4","temperature":null,"prompt":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"Hello, does this work?"},{"role":"AI","text":"Yes, it works. How can I assist you today?"},{"role":"AI","text":"Yes, it works. How can I assist you today?"},{"role":"Human","text":"Tell me something funny."}]}].    
Candidates are ["chat",{"provider":"OpenAI","llm":"gpt-4","temperature":null,"prompt":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"Hello, does this work?"},{"role":"AI","text":"Yes, it works. How can I assist you today?"},{"role":"Human","text":"Tell me something funny."}]}]
```

and the other being a problem with the mounting of the scrollable component:

```
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Rejection ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
TypeError: growable?.scrollToBottom is not a function
 ❯ Timeout._onTimeout src/routes/chat/Chat.svelte:55:17
     53|       await latestMessage?.resizeBubble(conversationWidthPx);
     54|       console.log(growable?.scrollToBottom);
     55|       growable?.scrollToBottom();
       |                 ^
     56|     }, 10);
     57|   }

This error originated in "src/routes/chat/Chat.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
```

The first problem only occurs when other tests are being run at the same time. After more debugging, it turns out that this is coming from the API callback from the previous test finally being executed. It is strange that `resetConversation();` in `afterEach` doesn't help to reset the conversation here, but it turns out it is because the previous test finishes before the API callback has had time to complete. We fix this test, which turns out to have never worked the way we expected because 1) it was simply waiting for the same `nextExpectedHumanPrompt` again instead of waiting for the next one, 2) it never even had the continue conversation playback added, and 3) it should've expected 2 successful API calls because the counter never gets reset here:

```ts
  test("won't send multiple messages at once", async () => {
    ...
    expect(screen.getByText(nextExpectedHumanPrompt)).toBeInTheDocument();
    // this part differs from sendChatMessage
    await waitFor(() => {
      expect(screen.getByText("Yes, it works. How can I assist you today?")).toBeInTheDocument();
    });

    playback.addSamples(
      "../src-tauri/api/sample-calls/chat-continue-conversation.yaml",
    );
    await userEvent.type(chatInput, "Tell me something funny.");
    await userEvent.click(screen.getByRole("button", { name: "Send" }));
    expect(tauriInvokeMock).toHaveBeenCalledTimes(2);
    expect(screen.getByText("Tell me something funny.")).toBeInTheDocument();
    await waitFor(() => {
      expect(screen.getByText(/Because they make up everything/)).toBeInTheDocument();
    });
  });
```

This tackles the first problem. As for the second one, since it appears to only exist in the test environment, we add a guard for it in `src-svelte\src\routes\chat\Chat.svelte`:

```ts
  function appendMessage(...) {
    ...
    setTimeout(async () => {
      ...
      if (growable?.scrollToBottom) {
        growable.scrollToBottom();
      }
    }, ...);
  }
```

This demonstrates a danger to having false confidence in tests that don't actually check for what you think they check for.

The Chat screenshot tests are failing now because this component no longer expects conversations to be passed in via arguments anymore. Instead, we update `src-svelte\src\lib\__mocks__\stores.ts`:

```ts
...
import { conversation } from "../../routes/chat/Chat.svelte";
import type { ..., ChatMessage } from "$lib/bindings";
...

interface Stores {
  ...
  conversation?: ChatMessage[];
}

...

const SvelteStoresDecorator: Decorator = (
  ...
) => {
  ...
  conversation.set(stores?.conversation || []);

  return story(...);
};

```

Then we edit `src-svelte/src/routes/chat/Chat.stories.ts`, moving the conversation argument from story `args` to `parameters`:

```ts
...

export const NotEmpty: StoryObj = Template.bind({}) as any;
NotEmpty.parameters = {
  stores: {
    conversation,
  },
  ...
};

...

export const TypingIndicator: StoryObj = Template.bind({}) as any;
TypingIndicator.args = {
  ...
};
TypingIndicator.parameters = {
  stores: {
    conversation: shortConversation,
  },
  ...
};

...
```

### Adding restore button on API call page

We edit `src-svelte\src\lib\controls\Button.svelte` to allow for firing click events. We could also do this by passing in props, but we prefer the event syntax:

```svelte
<script lang="ts">
  import { createEventDispatcher } from "svelte";

  ...
  const dispatchClickEvent = createEventDispatcher();

  function handleClick() {
    dispatchClickEvent("click");
  }
</script>

{#if unwrapped}
  <button ... on:click={handleClick}>
    {text}
  </button>
{:else}
  <button ... on:click={handleClick}>
    <div class="cut-corners inner" class:right-end={rightEnd}>{text}</div>
  </button>
{/if}
```

Then, we edit `src-svelte\src\routes\api-calls\[slug]\ApiCall.svelte` to add a button that updates the conversation store with the contents of the API call:

```svelte
<script lang="ts">
  ...
  import { type LlmCall, getApiCall } from "$lib/bindings";
  ...
  import Button from "$lib/controls/Button.svelte";
  import { conversation } from "../../chat/Chat.svelte";
  import { goto } from "$app/navigation";

  ...
  let apiCall: LlmCall | undefined = undefined;
  ...
  let apiCallPromise = getApiCall(id)
    .then((retrievedApiCall) => {
      apiCall = retrievedApiCall;
      ...
    })
    ...;

  function restoreConversation() {
    if (!apiCall) {
      snackbarError("API call not yet loaded");
      return;
    }

    const restoredConversation = [
      ...apiCall.request.prompt.messages,
      apiCall.response.completion,
    ]
    conversation.set(restoredConversation);

    goto("/chat");
  }
</script>

...

<div class="actions-container">
  <InfoBox title="Actions" childNumber={1}>
    <div class="action-buttons">
      <Button text="Restore conversation" on:click={restoreConversation} />
    </div>
  </InfoBox>
</div>

<style>
  ...

  .actions-container {
    margin-top: 1rem;
  }

  .action-buttons {
    width: fit-content;
    margin: 0 auto;
  }
</style>

```

Note that this strategy of separating the second InfoBox from the first one is different from the one employed in `src-svelte\src\routes\credits\Credits.svelte` to use the `gap` property of the containing div. We like the look of this gap, so we edit that file to also feature a gap of `1rem` instead.

We add a test, but at first we get the test error

```
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Suites 1 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/routes/api-calls/[slug]/ApiCall.test.ts [ src/routes/api-calls/[slug]/ApiCall.test.ts ]
Error: Failed to resolve import "$app/navigation" from "src/routes/api-calls/[slug]/ApiCall.svelte". Does the file exist?
 ❯ formatError ../../../../../../Amos%20Ng/Documents/projects/zamm-dev/zamm/node_modules/vite/dist/node/chunks/dep-stQc5rCc.js:50500:46
```

We figure out that our `src-svelte\vitest.config.ts` says

```ts
export default defineConfig({
  ...,
  resolve: {
    alias: {
      ...,
      $app: path.resolve("src/vitest-mocks"),
    },
  },
  ...
});
```

So we create a file at `src-svelte\src\vitest-mocks\navigation.ts`:

```ts
import { mockStores } from "./stores";

export function goto(url: string) {
  mockStores.page.set({
    url: new URL(url, window.location.href),
    params: {},
  });
}

```

and we modify `src-svelte\src\vitest-mocks\stores.ts`, which we originally created in the "Sound" section of [`sidebar.md`](/zamm-notes/sidebar.md), to be able to mock updates to the page store:

```ts
...

export const mockStores = {
  ...,
  page: writable({ url: new URL("http://localhost"), params: {} }),
  ...
};

export const page = {
  subscribe(...) {
    return mockStores.page.subscribe(fn);
  },
};

```

Now our test at `src-svelte\src\routes\api-calls\[slug]\ApiCall.test.ts` works completely:

```ts
import { expect, test, vi, type Mock } from "vitest";
import "@testing-library/jest-dom";

import { render, screen, waitFor } from "@testing-library/svelte";
import userEvent from "@testing-library/user-event";
import { resetConversation } from "../../chat/Chat.svelte";
import { TauriInvokePlayback } from "$lib/sample-call-testing";
import { conversation } from "../../chat/Chat.svelte";
import ApiCall from "./ApiCall.svelte";
import { get } from "svelte/store";
import { mockStores } from "../../../vitest-mocks/stores";

describe("Individual API call", () => {
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
  });

  afterEach(() => {
    vi.unstubAllGlobals();
    resetConversation();
  });

  test("can restore chat conversation", async () => {
    playback.addSamples("../src-tauri/api/sample-calls/get_api_call-continue-conversation.yaml");
    render(ApiCall, { id: "c13c1e67-2de3-48de-a34c-a32079c03316" });
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    await waitFor(() => {
      screen.getByText("Hello, does this work?");
    });
    expect(get(conversation)).toEqual([
      { role: "System", text: "You are ZAMM, a chat program. Respond in first person." },
    ]);
    expect(get(mockStores.page).url.pathname).toEqual("/");

    const restoreButton = await waitFor(() => screen.getByText("Restore conversation"));
    userEvent.click(restoreButton);
    await waitFor(() => {
      expect(get(conversation)).toEqual([
        { role: "System", text: "You are ZAMM, a chat program. Respond in first person." },
        {
          "role": "Human",
          "text": "Hello, does this work?"
        },
        {
          "role": "AI",
          "text": "Yes, it works. How can I assist you today?"
        },
        {
          "role": "Human",
          "text": "Tell me something funny."
        },
        {
          "role": "AI",
          "text": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
        },
      ]);
    });
    expect(get(mockStores.page).url.pathname).toEqual("/chat");
  });
});

```

#### Debugging overly short column widths

We notice that sometimes the API calls page sets the column width for messages to be extraordinarily short, but that this gets fixed when you scroll down and more API calls load. Upon more debugging, it turns out that this happens when you already have some chats in the conversation view (perhaps by loading one from the list of API calls), and then switch over from that page to the API list page. Because both pages will exist at the same time during the Svelte transition period, the `.message` bubbles from `src-svelte\src\routes\chat\MessageUI.svelte` get included in the JS selector for `src-svelte\src\routes\api-calls\ApiCalls.svelte`. As such, we edit `src-svelte\src\routes\api-calls\ApiCalls.svelte` to get the JS selector to be specific to the items on that page:

```svelte
<script lang="ts">
  ...

  function getWidths(selector: string) {
    ...
    const results = Array.from(elements)
      ...;
    return results;
  }

  function resizeMessageWidth() {
    ...

    setTimeout(() => {
      const textWidths = getWidths(".api-calls-page .text-container");
      const timeWidths = getWidths(".api-calls-page .time");
      ...
    }, ...);
  }

  ...
</script>

<InfoBox title="LLM API Calls" ...>
  <div class="container api-calls-page ..." ...>
    ...
  </div>
</InfoBox>
```

Note that we save the results of `getWidths` to an intermediate variable first for easier debug logging.

We manually check that the bug no longer appears. Reproducing this bug requires selecting a conversation that contains a short message, most likely from the human since ChatGPT is usually verbose in its responses. Testing this automatically would require writing an end-to-end test, as this is only triggered by navigation between pages. It would also require the API calls list and conversation view to be populated. We can leave this as a TODO for the future, but for now this would be too much effort for the payoff.

#### Refactoring out actions InfoBox

To be consistent with visual tests for other pages, we make it so that `src-svelte\src\routes\api-calls\[slug]\ApiCall.svelte` consists of components imported from two separate files instead:

```svelte
<script lang="ts">
  import ApiCallDisplay from "./ApiCallDisplay.svelte";
  import { type LlmCall, getApiCall } from "$lib/bindings";
  import Actions from "./Actions.svelte";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";

  export let id: string;
  let apiCall: LlmCall | undefined = undefined;

  getApiCall(id)
    .then((retrievedApiCall) => {
      apiCall = retrievedApiCall;
    })
    .catch((error) => {
      snackbarError(error);
    });
</script>

<div class="container">
  <ApiCallDisplay {...$$restProps} bind:apiCall={apiCall} />
  <Actions {apiCall} />
</div>

<style>
  .container {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }
</style>

```

where now `src-svelte\src\routes\api-calls\[slug]\Actions.svelte` looks like

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import { type LlmCall } from "$lib/bindings";
  import { conversation } from "../../chat/Chat.svelte";
  import { goto } from "$app/navigation";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";
  import Button from "$lib/controls/Button.svelte";

  export let apiCall: LlmCall | undefined = undefined;

  function restoreConversation() {
    if (!apiCall) {
      snackbarError("API call not yet loaded");
      return;
    }

    const restoredConversation = [
      ...apiCall.request.prompt.messages,
      apiCall.response.completion,
    ];
    conversation.set(restoredConversation);

    goto("/chat");
  }
</script>

<InfoBox title="Actions" childNumber={1}>
  <div class="action-buttons">
    <Button text="Restore conversation" on:click={restoreConversation} />
  </div>
</InfoBox>

<style>
  .action-buttons {
    width: fit-content;
    margin: 0 auto;
  }
</style>

```

and `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.svelte` looks like

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import SubInfoBox from "$lib/SubInfoBox.svelte";
  import { type LlmCall } from "$lib/bindings";
  import Loading from "$lib/Loading.svelte";

  export let dateTimeLocale: string | undefined = undefined;
  export let timeZone: string | undefined = undefined;
  export let apiCall: LlmCall | undefined = undefined;

  const formatter = new Intl.DateTimeFormat(dateTimeLocale, {
    year: "numeric",
    month: "long",
    day: "numeric",
    hour: "numeric",
    minute: "numeric",
    second: "numeric",
    hour12: true,
    timeZone,
  });

  let humanTime: string | undefined = undefined;
  let temperature: string | undefined = undefined;

  function updateDisplayStrings(apiCall: LlmCall | undefined) {
    if (!apiCall) {
      return;
    }

    const timestamp = apiCall.timestamp + "Z";
    const date = new Date(timestamp);
    humanTime = formatter.format(date);

    temperature = apiCall.request.temperature.toFixed(2);
  }

  $: updateDisplayStrings(apiCall);
</script>

<InfoBox title="API Call">
  {#if apiCall}
    <table>
      <tr>
        <td>ID</td>
        <td>{apiCall?.id ?? "Unknown"}</td>
      </tr>
      <tr>
        <td>Time</td>
        <td>{humanTime ?? "Unknown"}</td>
      </tr>
      <tr>
        <td>LLM</td>
        <td>
          {apiCall?.llm.requested ?? "Unknown"}
          {#if apiCall?.llm.requested !== apiCall?.llm.name}
            → {apiCall?.llm.name}
          {/if}
        </td>
      </tr>
      <tr>
        <td>Temperature</td>
        <td>
          {temperature ?? "Unknown"}
        </td>
      </tr>
      <tr>
        <td>Tokens</td>
        <td>
          {apiCall?.tokens.prompt ?? "Unknown"} prompt +
          {apiCall?.tokens.response ?? "Unknown"} response =
          {apiCall?.tokens.total ?? "Unknown"} total tokens
        </td>
      </tr>
    </table>

    <SubInfoBox subheading="Prompt">
      <div class="prompt">
        {#each apiCall?.request.prompt.messages ?? [] as message}
          <div class={"message " + message.role.toLowerCase()}>
            <span class="role">{message.role}</span>
            <pre>{message.text}</pre>
          </div>
        {/each}
      </div>
    </SubInfoBox>

    <SubInfoBox subheading="Response">
      <pre class="response">{apiCall?.response.completion.text ??
          "Unknown"}</pre>
    </SubInfoBox>
  {:else}
    <Loading />
  {/if}
</InfoBox>

<style>
  td:first-child {
    color: var(--color-faded);
    padding-right: 1rem;
  }

  .prompt {
    margin-bottom: 1rem;
  }

  .message {
    margin: 0.5rem 0;
    padding: 0.5rem;
    border-radius: var(--corner-roundness);
    display: flex;
    gap: 1rem;
    align-items: center;
  }

  .message.system {
    background-color: var(--color-system);
  }

  .message.human {
    background-color: var(--color-human);
  }

  .message.ai {
    background-color: var(--color-ai);
  }

  .message .role {
    color: var(--color-faded);
    width: 4rem;
    min-width: 4rem;
    text-align: center;
  }

  pre {
    white-space: pre-wrap;
    font-family: var(--font-mono);
    margin: 0;
    text-align: left;
  }
</style>

```

We rename `src-svelte/src/routes/api-calls/[slug]/ApiCall.stories.ts` to `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.stories.ts` and insert the API calls directly:

```ts
...

const CONTINUE_CONVERSATION_CALL = {
  "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
  "timestamp": "2024-01-16T09:50:19.738093890",
  ...
};

const KHMER_CALL = {
  "id": "92665f19-be8c-48f2-b483-07f1d9b97370",
  "timestamp": "2024-04-10T07:22:12.752276900",
  ...
};

...

export const Narrow: StoryObj = Template.bind({}) as any;
Narrow.args = {
  apiCall: CONTINUE_CONVERSATION_CALL,
  ...
};
...

export const Wide: StoryObj = Template.bind({}) as any;
Wide.args = {
  apiCall: CONTINUE_CONVERSATION_CALL,
  ...
};
...

export const Khmer: StoryObj = Template.bind({}) as any;
Khmer.args = {
  apiCall: KHMER_CALL,
  ...
};
...
```

This means we can now get rid of `src-tauri/api/sample-calls/get_api_call-khmer.yaml`, which was created only for visual test purposes anyway. We were using the visual tests as a proxy test that the API call logic works on mount, which we have now lost, so we instead edit `src-svelte/src/routes/api-calls/[slug]/ApiCall.test.ts` to test for that explicitly:

```ts
...
import { ..., within } from "@testing-library/svelte";
...

describe("Individual API call", () => {
  ...

  function expectRowValue(key: string, expectedValue: string) {
    const keyRegex = new RegExp(key);
    const row = screen.getByRole("row", { name: keyRegex });
    const rowValueCell = within(row).getAllByRole("cell")[1];
    expect(rowValueCell).toHaveTextContent(expectedValue);
  }

  test("can load API call with correct details", async () => {
    playback.addSamples(
      "../src-tauri/api/sample-calls/get_api_call-continue-conversation.yaml",
    );
    render(ApiCall, { id: "c13c1e67-2de3-48de-a34c-a32079c03316" });
    expect(tauriInvokeMock).toHaveReturnedTimes(1);

    // check that system message is displayed
    await waitFor(() => {
      expect(screen.getByText("You are ZAMM, a chat program. Respond in first person.")).toBeInTheDocument();
    });
    // check that human message is displayed
    expect(screen.getByText("Hello, does this work?")).toBeInTheDocument();
    // check that AI message is displayed
    expect(screen.getByText(
      "Sure, here's a joke for you: " + 
      "Why don't scientists trust atoms? " +
      "Because they make up everything!"
    )).toBeInTheDocument();
    // check that metadata is displayed
    expectRowValue("LLM", "gpt-4 → gpt-4-0613");
    expectRowValue("Temperature", "1.00");
    expectRowValue("Tokens", "57 prompt + 22 response = 79 total tokens");
  });

  ...
});
```

where the table trick is something we copied from `src-svelte\src\routes\components\Metadata.test.ts`.

We add new stories for the actions component specifically at `src-svelte/src/routes/api-calls/[slug]/Actions.stories.ts`:

```ts
import ActionsComponent from "./Actions.svelte";
import type { StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";

export default {
  component: ActionsComponent,
  title: "Screens/LLM Call/Individual/Actions",
  argTypes: {},
  decorators: [TauriInvokeDecorator],
};

const Template = ({ ...args }) => ({
  Component: ActionsComponent,
  props: args,
});

export const Wide: StoryObj = Template.bind({}) as any;
Wide.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};

export const Narrow: StoryObj = Template.bind({}) as any;
Narrow.parameters = {
  viewport: {
    defaultViewport: "mobile2",
  },
};

```

We register these new stories in `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "individual", "actions"],
    variants: ["narrow", "wide"],
    screenshotEntireBody: true,
  },
  ...
];
```

## InfoBox reveal animation

We add some classes to help the messages be revealed one at a time in `src-svelte/src/lib/SubInfoBox.svelte`:

```svelte
<section class="sub-info-box composite-reveal" ...>
  ...
  <div class="content composite-reveal">
    ...
  </div>
</section>
```

We then edit `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.svelte`:

```svelte
<InfoBox title="API Call">
  {#if apiCall}
    <table class="composite-reveal">
      ...
    </table>

    <SubInfoBox subheading="Prompt">
      <div class="prompt composite-reveal">
        {#each ... as message}
          <div class={"message atomic-reveal " + ...}>
            ...
          </div>
        {/each}
      </div>
    </SubInfoBox>

    ...
  {/if}
</InfoBox>
```

## Editing API calls

We want to allow the user to edit existing API calls to produce new ones.

### Frontend refactoring

#### Prompt

We start by refactoring the Prompt display, along with all associated styles, out of `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.svelte`:

```svelte
<script lang="ts">
  ...
  import Prompt from "./Prompt.svelte";
  ...
</script>

<InfoBox title="API Call">
  {#if apiCall}
    <table class="composite-reveal">
      ...
    </table>

    <Prompt prompt={apiCall.request.prompt} />

    <SubInfoBox subheading="Response">
      ...
    </SubInfoBox>
    
    ...
  {/if}
</InfoBox>

<style>
  ...
</style>
```

where `src-svelte\src\routes\api-calls\[slug]\Prompt.svelte` now looks like

```svelte
<script lang="ts">
  import autosize from "autosize";
  import SubInfoBox from "$lib/SubInfoBox.svelte";
  import { type Prompt } from "$lib/bindings";
  import { onMount } from "svelte";

  export let prompt: Prompt;
  export let editable = false;
  let textareas: HTMLTextAreaElement[] = [];

  function toggleRole(i: number) {
    switch (prompt.messages[i].role) {
      case "System":
        prompt.messages[i].role = "Human";
        break;
      case "Human":
        prompt.messages[i].role = "AI";
        break;
      case "AI":
        prompt.messages[i].role = "System";
        break;
    }
  }

  function editText(
    i: number,
    e: Event & { currentTarget: EventTarget & HTMLTextAreaElement },
  ) {
    prompt.messages[i].text = e.currentTarget.value;
  }

  function addMessage() {
    prompt.messages = [...prompt.messages, { role: "System", text: "" }];
  }

  onMount(() => {
    textareas.forEach((textarea) => autosize(textarea));
    return () => {
      textareas.forEach((textarea) => autosize.destroy(textarea));
    };
  });
</script>

<SubInfoBox subheading="Prompt">
  <div class="prompt composite-reveal">
    {#each prompt.messages ?? [] as message, i}
      <div class={"message atomic-reveal " + message.role.toLowerCase()}>
        <span
          class="role"
          class:editable
          role="button"
          aria-label="Toggle message type"
          tabindex="0"
          on:click={() => toggleRole(i)}
          on:keypress={() => toggleRole(i)}>{message.role}</span
        >
        {#if editable}
          <textarea
            rows="1"
            value={message.text}
            on:input={(e) => editText(i, e)}
            bind:this={textareas[i]}
          />
        {:else}
          <pre>{message.text}</pre>
        {/if}
      </div>
    {/each}
    {#if editable}
      <button title="Add a new message to the chat" on:click={addMessage}
        >+</button
      >
    {/if}
  </div>
</SubInfoBox>

<style>
  .prompt {
    margin-bottom: 1rem;
  }

  .role.editable {
    cursor: pointer;
  }

  .message {
    transition-property: background-color;
    transition: var(--standard-duration) ease-out;
    margin: 0.5rem 0;
    padding: 0.5rem;
    border-radius: var(--corner-roundness);
    display: flex;
    gap: 1rem;
    align-items: center;
  }

  .message.system {
    background-color: var(--color-system);
  }

  .message.human {
    background-color: var(--color-human);
  }

  .message.ai {
    background-color: var(--color-ai);
  }

  .message .role {
    color: var(--color-faded);
    width: 4rem;
    min-width: 4rem;
    text-align: center;
  }

  textarea {
    font-family: var(--font-mono);
    font-size: 1rem;
    background-color: transparent;
    width: 100%;
    resize: none;
  }

  button {
    width: 100%;
    padding: 0.2rem;
    box-sizing: border-box;
    background: transparent;
    border: 2px dashed var(--color-border);
    border-radius: var(--corner-roundness);
    font-size: 1rem;
    color: var(--color-faded);
    cursor: pointer;
    transition: calc(0.5 * var(--standard-duration)) ease-out;
  }

  button:hover {
    background: var(--color-border);
    color: #fff;
  }
</style>

```

because we made it possible to edit each message in the prompt, as well as add new messages and toggle message types.

Because the `pre` elements are now split apart, we move the styling for them to `src-svelte\src\routes\styles.css`:

```css
pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  word-break: break-word;
  font-family: var(--font-mono);
  margin: 0;
  text-align: left;
}
```

We refactor the sample calls out of `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.stories.ts`:

```ts
...
import {
  CONTINUE_CONVERSATION_CALL,
  KHMER_CALL,
  LOTS_OF_CODE_CALL,
} from "./sample-calls";
...

export default {
  component: ApiCallComponent,
  ...
};

...
```

where `src-svelte\src\routes\api-calls\[slug]\sample-calls.ts` looks like

```ts
export const CONTINUE_CONVERSATION_CALL = {
  id: "c13c1e67-2de3-48de-a34c-a32079c03316",
  ...
};

export const KHMER_CALL = {
  id: "92665f19-be8c-48f2-b483-07f1d9b97370",
  ...
};

export const LOTS_OF_CODE_CALL = {
  id: "9857257b-8e17-4203-91eb-c10bef8ff4e6",
  ...
};
```

We introduce new stories to `src-svelte\src\routes\api-calls\[slug]\Prompt.stories.ts` as well:

```ts
import PromptComponent from "./Prompt.svelte";
import type { StoryObj } from "@storybook/svelte";
import { CONTINUE_CONVERSATION_CALL } from "./sample-calls";

export default {
  component: PromptComponent,
  title: "Screens/LLM Call/Individual/Prompt",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: PromptComponent,
  props: args,
});

export const Uneditable: StoryObj = Template.bind({}) as any;
Uneditable.args = {
  prompt: CONTINUE_CONVERSATION_CALL.request.prompt,
};

export const Editable: StoryObj = Template.bind({}) as any;
Editable.args = {
  editable: true,
  prompt: CONTINUE_CONVERSATION_CALL.request.prompt,
};

```

Now that we have the initial UI mocked out, we edit `src-svelte\src\routes\api-calls\[slug]\Prompt.svelte` a little further and introduce a placeholder of "Set text for new prompt message...".

We also realize that we should avoid making the role toggle-able when the prompt is not editable. We edit `src-svelte/src/routes/api-calls/[slug]/Prompt.svelte` again:

```ts
  function toggleRole(i: number) {
    if (!editable) {
      return;
    }

    ...
  }
```

#### New API call submission

We realize that before implementing the ability to edit API calls, it may make sense to add the ability to make arbitrary new API calls. We create a new component at `src-svelte/src/routes/api-calls/new/ApiCallEditor.svelte`:

```svelte
<script lang="ts" context="module">
  import type { Prompt as PromptType } from "$lib/bindings";
  import { writable } from "svelte/store";
  export const prompt = writable<PromptType>({
    type: "Chat",
    messages: [{ role: "System", text: "" }],
  });
</script>

<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import PromptComponent from "../[slug]/Prompt.svelte";
  import Button from "$lib/controls/Button.svelte";
  import { chat } from "$lib/bindings";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";
  import { goto } from "$app/navigation";
  let expectingResponse = false;
  async function submitApiCall() {
    if (expectingResponse) {
      return;
    }
    expectingResponse = true;
    try {
      const createdLlmCall = await chat({
        provider: "OpenAI",
        llm: "gpt-4",
        temperature: null,
        prompt: $prompt.messages,
      });
      goto(`/api-calls/${createdLlmCall.id}`);
    } catch (error) {
      snackbarError(error as string | Error);
    } finally {
      expectingResponse = false;
    }
  }
</script>

<InfoBox title="New API Call">
  <PromptComponent editable bind:prompt={$prompt} />

  <div class="action">
    <Button on:click={submitApiCall}>Submit</Button>
  </div>
</InfoBox>

<style>
  .action :global(button) {
    width: 100%;
  }
</style>
```

We add a corresponding story at `src-svelte/src/routes/api-calls/new/ApiCallEditor.stories.ts`:

```ts
import ApiCallEditorComponent from "./ApiCallEditor.svelte";
import type { StoryObj } from "@storybook/svelte";

export default {
  component: ApiCallEditorComponent,
  title: "Screens/LLM Call/New",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: ApiCallEditorComponent,
  props: args,
});

export const Blank: StoryObj = Template.bind({}) as any;
```

and add this to `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "new"],
    variants: ["blank"],
  },
  ...
];
```

Then, we edit `src-svelte/src/routes/api-calls/new/ApiCallEditor.svelte` again to make it ready for testing by introducing a reset function:

```svelte
<script lang="ts" context="module">
  ...

  export function resetNewApiCall() {
    prompt.set({
      type: "Chat",
      messages: [{ role: "System", text: "" }],
    });
  }
</script>

<script lang="ts">
  ...

  async function submitApiCall() {
    ...

    try {
      ...
      resetNewApiCall();

      ...
    } ...
  }
</script>
```

We can now test this in `src-svelte/src/routes/api-calls/new/ApiCallEditor.test.ts`:

```ts
import { expect, test, vi, type Mock } from "vitest";
import "@testing-library/jest-dom";

import { render, screen } from "@testing-library/svelte";
import ApiCallEditor, { resetNewApiCall } from "./ApiCallEditor.svelte";
import userEvent from "@testing-library/user-event";
import { TauriInvokePlayback } from "$lib/sample-call-testing";
import { get } from "svelte/store";
import { mockStores } from "../../../vitest-mocks/stores";

describe("API call editor", () => {
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
  });

  afterEach(() => {
    vi.unstubAllGlobals();
    resetNewApiCall();
  });

  async function addNewMessage() {
    const numMessages = screen.getAllByRole("listitem").length;
    const newMessageButton = screen.getByRole("button", { name: "+" });
    await userEvent.click(newMessageButton);
    expect(screen.getAllByRole("listitem").length).toBe(numMessages + 1);
  }

  async function setNewMessage(role: string, message: string) {
    const messageDivs = screen.getAllByRole("listitem");
    const lastMessageDiv = messageDivs[messageDivs.length - 1];

    const roleToggle = lastMessageDiv.querySelector(
      'span[aria-label="Toggle message type"]',
    );
    if (roleToggle === null) {
      throw new Error("Role toggle not found");
    }
    for (let i = 0; i < 3; i++) {
      if (roleToggle.textContent === role) {
        break;
      }
      await userEvent.click(roleToggle);
    }
    expect(roleToggle.textContent).toBe(role);

    const messageInput = lastMessageDiv.querySelector("textarea");
    if (messageInput === null) {
      throw new Error("Message input not found");
    }
    await userEvent.type(messageInput, message);
    expect(messageInput).toHaveValue(message);
  }

  test("can manually trigger API call with all roles", async () => {
    render(ApiCallEditor, {});
    expect(tauriInvokeMock).not.toHaveBeenCalled();
    playback.addSamples(
      "../src-tauri/api/sample-calls/chat-manual-conversation-recreation.yaml",
    );

    await setNewMessage(
      "System",
      "You are ZAMM, a chat program. Respond in first person.",
    );
    await addNewMessage();
    await setNewMessage("Human", "Hello, does this work?");
    await addNewMessage();
    await setNewMessage("AI", "Yes, it works. How can I assist you today?");
    await addNewMessage();
    await setNewMessage("Human", "Tell me something funny.");

    await userEvent.click(screen.getByRole("button", { name: "Submit" }));
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    expect(get(mockStores.page).url.pathname).toEqual(
      "/api-calls/c13c1e67-2de3-48de-a34c-a32079c03316",
    );
  });
});

```

To get this test to run, we need to edit `src-svelte/src/routes/api-calls/[slug]/Prompt.svelte` to mark the roles of each div as a "listitem" for use in the test. We also realize we should only set the role to "button" for the label if it is actually editable, and otherwise just disable it.

```svelte
<SubInfoBox subheading="Prompt">
  <div class="prompt composite-reveal" role="list">
    {#each prompt.messages ?? [] as message, i}
      <div
        ...
        role="listitem"
      >
        <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
        <span
          ...
          role={editable ? "button" : "text"}
          aria-label={editable ? "Toggle message type" : undefined}
          tabindex={editable ? 0 : undefined}
          ...>{message.role}</span
        >
        ...
      </div>
    {/each}
    ...
  </div>
</SubInfoBox>
```

However, we also need to ignore the `a11y-no-noninteractive-tabindex` warning, or else we get hit with:

```
/root/zamm/src-svelte/src/routes/api-calls/[slug]/Prompt.svelte
  55:9  error  A11y: noninteractive element cannot have nonnegative tabIndex value(a11y-no-noninteractive-tabindex)  svelte/valid-compile
```

We had meant to reuse the API call `src-tauri/api/sample-calls/chat-continue-conversation.yaml` in the test, but it turns out we need to create a new one at `src-tauri/api/sample-calls/chat-manual-conversation-recreation.yaml`:

```yaml
request:
  - chat
  - >
    {
      "args": {
        "provider": "OpenAI",
        "llm": "gpt-4",
        "temperature": null,
        "prompt": [
          {
            "role": "System",
            "text": "You are ZAMM, a chat program. Respond in first person."
          },
          {
            "role": "Human",
            "text": "Hello, does this work?"
          },
          {
            "role": "AI",
            "text": "Yes, it works. How can I assist you today?"
          },
          {
            "role": "Human",
            "text": "Tell me something funny."
          }
        ]
      }
    }
response:
  message: >
    {
      "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
      "timestamp": "2024-01-16T09:50:19.738093890",
      "response_message": {
        "role": "AI",
        "text": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
      }
    }
sideEffects:
  database:
    startStateDump: conversation-started
    endStateDump: conversation-manually-recreated
  network:
    recordingFile: continue-conversation.json

```

The only difference here is that there is no `previous_call_id` listed in the arguments -- and therefore a different database dump as well. We create a `src-tauri/api/sample-database-writes/conversation-manually-recreated/dump.sql` that's a copy of `src-tauri/api/sample-database-writes/conversation-continued/dump.sql` except for the link between the new API call and the previous one, and we create a `src-tauri/api/sample-database-writes/conversation-manually-recreated/dump.yaml` that is likewise a copy of `src-tauri/api/sample-database-writes/conversation-continued/dump.yaml` but without the `conversation` key.

We make sure that this new API call is consistent with how the backend processes it by editing `src-tauri/src/commands/llms/chat.rs`:

```rs
    #[tokio::test]
    async fn test_manual_conversation_recreation() {
        test_llm_api_call(
            function_name!(),
            "api/sample-calls/chat-manual-conversation-recreation.yaml",
        )
        .await;
    }
```

While committing, we find that we must remove the extraneous `use std::env;` from `src-tauri/src/main.rs`. We remove it, but it is worrying that this lint was not detected on Windows or in the CI environment.

We need to put this in an actual page in the app. We create `src-svelte/src/routes/api-calls/new/+page.svelte`:

```svelte
<script>
  import ApiCallEditor from "./ApiCallEditor.svelte";
</script>

<ApiCallEditor />

```

We then link to it by editing `src-svelte/src/routes/api-calls/ApiCalls.svelte`:

```svelte
<script lang="ts">
  ...
  import IconAdd from "~icons/mingcute/add-fill";
  ...
</script>

<InfoBox title="LLM API Calls" fullHeight>
  <div class="container api-calls-page full-height" style={minimumWidths}>
    <a class="new-api-call" href="/api-calls/new/">
      <IconAdd />
    </a>
    ...
  </div>
</InfoBox>

<style>
  ...

  a.new-api-call {
    position: absolute;
    top: 1rem;
    right: 1rem;
  }

  a.new-api-call :global(svg) {
    transform: scale(1.2);
    color: var(--color-faded);
  }

  ...
</style>
```

We further `src-svelte/src/routes/api-calls/ApiCalls.svelte` to change the placeholder text to refer to the new possibility for populating API calls:

```svelte
              <EmptyPlaceholder>
                Looks like you haven't made any calls to an LLM yet.<br />Get
                started via <a href="/chat">chat</a> or by making one <a href="/api-calls/new/">from scratch</a>.
              </EmptyPlaceholder>
```

We edit `src-svelte/src/lib/EmptyPlaceholder.svelte` to style the placeholder links in the same faded color, because we want all placeholder links to look this way across the app:

```css
  .empty :global(a) {
    color: var(--color-faded);
    text-decoration: underline;
  }
```

We want to make sure that this new button looks right when the info box animation is being loaded, so we create `src-svelte/src/routes/api-calls/ApiCalls.full-page.stories.ts` to test:

```ts
import ApiCallsComponent from "./ApiCalls.svelte";
import MockPageTransitions from "$lib/__mocks__/MockPageTransitions.svelte";
import type { StoryFn, StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";

export default {
  component: ApiCallsComponent,
  title: "Screens/LLM Call/List",
  argTypes: {},
  decorators: [
    TauriInvokeDecorator,
    (story: StoryFn) => {
      return {
        Component: MockPageTransitions,
        slot: story,
      };
    },
  ],
};

const Template = ({ ...args }) => ({
  Component: ApiCallsComponent,
  props: args,
});

export const FullPage: StoryObj = Template.bind({}) as any;
FullPage.args = {
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
FullPage.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
  sampleCallFiles: [
    "/api/sample-calls/get_api_calls-full.yaml",
    "/api/sample-calls/get_api_calls-offset.yaml",
  ],
};

```

Next, we want to add this page to the end-to-end screenshots test. We follow the example [here](https://webdriver.io/fr/docs/selectors/#element-with-certain-text) and add this test to `webdriver/test/specs/e2e.test.js`:

```js
  it("should allow navigation to the new API call page", async function () {
    this.retries(2);
    await findAndClick('a[title="API Calls"]');
    await findAndClick('a=from scratch');
    await findAndClick('a[title="API Calls"]');
    await findAndClick('a=from scratch');
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("new-api-call", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });
```

We finish up with some nits by editing `src-svelte/src/routes/api-calls/ApiCalls.svelte` to add a mouseover title for the add button:

```svelte
    <a class="new-api-call" ... title="New API call">
      <IconAdd />
    </a>
```

and we edit `src-svelte/src/routes/api-calls/new/ApiCallEditor.svelte` to get the submit button to grow, but only up to a fixed size, so that it doesn't look too tiny (if it fits to content) but also doesn't get too unwieldy large (if the window is too big):

```css
  .action {
    width: 100%;
    display:flex;
    justify-content: center;
  }

  .action :global(button) {
    width: 100%;
    max-width: 25rem;
  }
```

and edit `src-svelte/src/routes/storybook.test.ts` to screenshot the entire body:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "new"],
    ...,
    screenshotEntireBody: true,
  },
  ...
];
```

We find that a couple of screenshots, `src-svelte/screenshots/baseline/screens/chat/conversation/new-message-sent.png` and `src-svelte/screenshots/baseline/screens/chat/conversation/full-message-width.png`, are failing now because of our change to the `pre` word break. We move these three lines outside of `src-svelte/src/routes/styles.css` and into `src-svelte/src/routes/api-calls/[slug]/Prompt.svelte`:

```css
  .message pre {
    white-space: pre-wrap;
    word-wrap: break-word;
    word-break: break-word;
  }
```

and `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.svelte`:

```css
  pre {
    white-space: pre-wrap;
    word-wrap: break-word;
    word-break: break-word;
  }
```

It is good that our screenshot tests caught this.

### Backend implementation

We do a new Diesel migration:

```bash
$ diesel migration generate variant_links                                    
Creating migrations\2024-05-29-183428_variant_links\up.sql
Creating migrations\2024-05-29-183428_variant_links\down.sql
```

where `src-tauri\migrations\2024-05-29-183428_variant_links\up.sql` is

```sql
CREATE TABLE llm_call_variants (
  canonical_id VARCHAR NOT NULL,
  variant_id VARCHAR NOT NULL,
  PRIMARY KEY (canonical_id, variant_id),
  FOREIGN KEY (canonical_id) REFERENCES llm_calls (id) ON DELETE CASCADE,
  FOREIGN KEY (variant_id) REFERENCES llm_calls (id) ON DELETE CASCADE
);

CREATE VIEW llm_call_named_variants AS
  SELECT
    llm_call_variants.canonical_id AS canonical_id,
    canonical.completion AS canonical_completion,
    llm_call_variants.variant_id AS variant_id,
    variant.completion AS variant_completion
  FROM
    llm_call_variants
    JOIN llm_calls AS canonical ON llm_call_variants.canonical_id = canonical.id
    JOIN llm_calls AS variant ON llm_call_variants.variant_id = variant.id
  ORDER BY variant.timestamp ASC;

```

and `src-tauri\migrations\2024-05-29-183428_variant_links\down.sql` is:

```sql
DROP TABLE llm_call_variants;
DROP VIEW llm_call_named_variants;

```

This is of course based mainly on the API call conversation links table and view. We run this migration:

```bash
$ diesel migration run --database-url 'C:\Users\Amos Ng\AppData\Roaming\zamm\ZAMM\data\zamm.sqlite3'
Running migration 2024-05-29-183428_variant_links
```

and by doing so `src-tauri\src\schema.rs` gets updated automatically. We'll have to manually edit `src-tauri\src\views.rs` ourselves to introduce our new view:

```rs
...

diesel::table! {
    llm_call_named_variants (canonical_id, variant_id) {
        canonical_id -> Text,
        canonical_completion -> Text,
        variant_id -> Text,
        variant_completion -> Text,
    }
}

diesel::allow_tables_to_appear_in_same_query!(..., llm_call_named_variants, ...);

```

Next, we recreate much of the querying that we did for conversation links, but this time to instead gather data for edited API links. First we start off by editing `src-tauri\src\models\llm_calls\various.rs` to add the new metadata data structure we want:

```rs
#[derive(Debug, Default, Clone, Serialize, Deserialize, specta::Type)]
pub struct VariantMetadata {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub canonical: Option<LlmCallReference>,
    #[serde(skip_serializing_if = "Vec::is_empty", default)]
    pub variants: Vec<LlmCallReference>,
    #[serde(skip_serializing_if = "Vec::is_empty", default)]
    pub sibling_variants: Vec<LlmCallReference>,
}

impl VariantMetadata {
    pub fn is_default(&self) -> bool {
        self.canonical.is_none()
            && self.variants.is_empty()
            && self.sibling_variants.is_empty()
    }
}

```

We define how this data structure is supposed to be assembled in `src-tauri\src\models\llm_calls\llm_call.rs`. Note that a list of LLM call references is returned every time; the only difference is whether this represents a list of variant children that this is the canonical parent of, or whether this represents the list of variant siblings that this is but one member of:

```rs
...
use crate::models::llm_calls::various::{
    ...,
    VariantMetadata,
};
...

#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct LlmCall {
    ...
    #[serde(skip_serializing_if = "VariantMetadata::is_default", default)]
    pub variation: VariantMetadata,
}

...

pub type LlmCallLeftJoinResult = (
    LlmCallRow,
    Option<EntityId>,
    Option<ChatMessage>,
    Option<EntityId>,
    Option<ChatMessage>,
);

pub type LlmCallQueryResults = (
    LlmCallLeftJoinResult,
    Vec<(EntityId, ChatMessage)>,
    Vec<(EntityId, ChatMessage)>,
);

impl From<LlmCallQueryResults> for LlmCall {
    fn from(query_results: LlmCallQueryResults) -> Self {
        let (
            (
                llm_call_row,
                previous_call_id,
                previous_call_completion,
                maybe_canonical_id,
                maybe_canonical_completion,
            ),
            next_calls,
            variants,
        ) = query_results;

        ...
        let variant_references = variants
            .into_iter()
            .map(|(id, completion)| (id, completion).into())
            .collect();
        let variant_metadata = if let (Some(canonical_id), Some(canonical_completion)) =
            (maybe_canonical_id, maybe_canonical_completion)
        {
            VariantMetadata {
                canonical: Some((canonical_id, canonical_completion).into()),
                variants: Vec::new(),
                sibling_variants: variant_references,
            }
        } else {
            VariantMetadata {
                canonical: None,
                variants: variant_references,
                sibling_variants: Vec::new(),
            }
        };

        LlmCall {
            ...,
            variation: variant_metadata,
        }
    }
}

```

We do the actual query in `src-tauri/src/commands/llms/get_api_call.rs`:

```rs
async fn get_api_call_helper(
    ...
) -> ZammResult<LlmCall> {
    ...

    let left_join_result: LlmCallLeftJoinResult = llm_calls::table
        .left_join(
            llm_call_named_follow_ups::...,
        )
        .left_join(
            llm_call_named_variants::dsl::llm_call_named_variants
                .on(llm_calls::id.eq(llm_call_named_variants::variant_id)),
        )
        .select((
            ...,
            llm_call_named_variants::canonical_id.nullable(),
            llm_call_named_variants::canonical_completion.nullable(),
        ))
        ...;
    let next_calls_result = llm_call_named_follow_ups::table
        .select((
            ...
        ))
        .filter(llm_call_named_follow_ups::previous_call_id.eq(&parsed_uuid))
        .load::<(EntityId, ChatMessage)>(conn)?;
    let canonical_id = left_join_result.3.clone().unwrap_or(parsed_uuid);
    let variants_result = llm_call_named_variants::table
        .select((
            llm_call_named_variants::variant_id,
            llm_call_named_variants::variant_completion,
        ))
        .filter(llm_call_named_variants::canonical_id.eq(canonical_id))
        .load::<(EntityId, ChatMessage)>(conn)?;
    Ok((left_join_result, next_calls_result, variants_result).into())
}
```

Finally, we also update the queries in `src-tauri/src/test_helpers/database_contents.rs`:

```rs
...
use crate::views::{..., llm_call_named_variants};
...

pub async fn get_database_contents(
    ...
) -> ZammResult<DatabaseContents> {
    ...
    let llm_call_left_joins = llm_calls::table
        .left_join(
            llm_call_named_follow_ups::...,
        )
        .left_join(
            llm_call_named_variants::dsl::llm_call_named_variants
                .on(llm_calls::id.eq(llm_call_named_variants::variant_id)),
        )
        .select((
            ...,
            llm_call_named_variants::canonical_id.nullable(),
            llm_call_named_variants::canonical_completion.nullable(),
        ))
        ...;
    let llm_calls_result: ZammResult<Vec<LlmCall>> = llm_call_left_joins
        .into_iter()
        .map(|lf| {
            let (
                llm_call_row,
                previous_call_id,
                previous_call_completion,
                maybe_canonical_id,
                maybe_canonical_completion,
            ) = lf;
            let next_calls_result: Vec<(EntityId, ChatMessage)> =
                llm_call_named_follow_ups::...;
            let canonical_id = maybe_canonical_id
                .clone()
                .unwrap_or(llm_call_row.id.clone());
            let variants_result: Vec<(EntityId, ChatMessage)> =
                llm_call_named_variants::table
                    .select((
                        llm_call_named_variants::variant_id,
                        llm_call_named_variants::variant_completion,
                    ))
                    .filter(llm_call_named_variants::canonical_id.eq(canonical_id))
                    .load::<(EntityId, ChatMessage)>(db)?;
            let reconstituted_lf: LlmCallLeftJoinResult = (
                llm_call_row,
                previous_call_id,
                previous_call_completion,
                maybe_canonical_id,
                maybe_canonical_completion,
            );
            Ok((reconstituted_lf, next_calls_result, variants_result).into())
        })
        .collect();
    Ok(DatabaseContents {
        ...
    })
}
```

We destructure and restructure the elements again because we encountered an error with simply pulling one of the elements out. After we get everything in a working state, we simplify things further with:

```rs
    let llm_calls_result: ZammResult<Vec<LlmCall>> = llm_call_left_joins
        .into_iter()
        .map(|lf| {
            let llm_call_id = lf.0.id.clone();
            let next_calls_result: Vec<(EntityId, ChatMessage)> =
                llm_call_named_follow_ups::table
                    .select((
                        ...
                    ))
                    .filter(
                        llm_call_named_follow_ups::previous_call_id
                            .eq(&llm_call_id),
                    )
                    .load::...;
            let canonical_id = lf.3.clone().unwrap_or(llm_call_id);
            let variants_result: Vec<(EntityId, ChatMessage)> =
                ...;
            Ok((lf, next_calls_result, variants_result).into())
        })
        .collect();
```

We check that all our tests pass, as they should because no API calls with actual variants have been added yet.

#### Linking variants in API calls

Now we add code to actually write the variant links out. We modify `src-tauri\src\commands\llms\chat.rs` to take in variant metadata when the frontend sends it:

```rs
...
use crate::models::llm_calls::{
    ..., NewLlmCallVariant, ...
};
use crate::schema::{..., llm_call_variants};
...
use diesel::prelude::*;
...

#[derive(...)]
pub struct ChatArgs {
    ...
    #[serde(skip_serializing_if = "Option::is_none")]
    canonical_id: Option<Uuid>,
}

async fn chat_helper(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...

    if let Some(conn) = db.as_mut() {
        ...

        if let Some(potential_canonical_uuid) = args.canonical_id {
            let potential_canonical_id = EntityId { uuid: potential_canonical_uuid };
            // check if the canonical ID is itself a variant
            let canonical_id = llm_call_variants::table
                .select(llm_call_variants::canonical_id)
                .filter(llm_call_variants::variant_id.eq(&potential_canonical_id))
                .first::<EntityId>(conn)
                .unwrap_or(potential_canonical_id);
            diesel::insert_into(llm_call_variants::table)
                .values(NewLlmCallVariant {
                    canonical_id: &canonical_id,
                    variant_id: &new_id,
                })
                .execute(conn)?;
        }
    } // todo: warn users if DB write unsuccessful

    ...
}

...

#[cfg(test)]
mod tests {
    ...
    
    #[tokio::test]
    async fn test_edit_conversation() {
        test_llm_api_call(
            function_name!(),
            "api/sample-calls/chat-edit-conversation.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_re_edit_conversation() {
        test_llm_api_call(
            function_name!(),
            "api/sample-calls/chat-re-edit-conversation.yaml",
        )
        .await;
    }
}

```

Our first test at `src-tauri/api/sample-calls/chat-edit-conversation.yaml` just modifies the last message:

```yaml
request:
  - chat
  - >
    {
      "args": {
        "provider": "OpenAI",
        "llm": "gpt-4",
        "temperature": null,
        "canonical_id": "c13c1e67-2de3-48de-a34c-a32079c03316",
        "prompt": [
          ...,
          {
            "role": "Human",
            "text": "Tell me a funny joke."
          }
        ]
      }
    }
response:
  message: >
    {
      "id": "f39a5017-89d4-45ec-bcbb-25c2bd43cfc1",
      "timestamp": "2024-06-08T06:20:40.601356700",
      "response_message": {
        "role": "AI",
        "text": "Sure, here is a light-hearted joke for you: \n\nWhy don't scientists trust atoms?\n\nBecause they make up everything!"
      }
    }
sideEffects:
  database:
    startStateDump: conversation-forked-step-2
    endStateDump: conversation-edited
  network:
    recordingFile: edit-conversation.json

```

Our second transcript at `src-tauri/api/sample-calls/chat-re-edit-conversation.yaml` is based on this first edit, and itself further edits an earlier message:

```yaml
request:
  - chat
  - >
    {
      "args": {
        "provider": "OpenAI",
        "llm": "gpt-4",
        "temperature": null,
        "canonical_id": "f39a5017-89d4-45ec-bcbb-25c2bd43cfc1",
        "prompt": [
          ...,
          {
            "role": "Human",
            "text": "Hello, does this really work?"
          },
          ...,
          {
            "role": "Human",
            "text": "Tell me a funny joke."
          }
        ]
      }
    }
response:
  message: >
    {
      "id": "7a35a4cf-f3d9-4388-bca8-2fe6e78c9648",
      "timestamp": "2024-06-08T09:40:22.392223700",
      "response_message": {
        "role": "AI",
        "text": "Sure, here you go: Why don't scientists trust atoms? Because they make up everything!"
      }
    }
sideEffects:
  database:
    startStateDump: conversation-edited
    endStateDump: conversation-edited-2
  network:
    recordingFile: re-edit-conversation.json

```

However, because `7a35a4cf-f3d9-4388-bca8-2fe6e78c9648` is itself a variant call, we want to make sure that the backend actually registers this as a variant of the original `c13c1e67-2de3-48de-a34c-a32079c03316` instead. This does call into question whether the database snapshots contain as much high-level intentions of our tests as a regular database query in the test would. The database query provides stronger guarantees for testing at first, but if we re-record the database, we would lose sight of the original intentions. As such, we edit `src-tauri\src\commands\llms\chat.rs` again to make sure that this nuance is captured in a comment:

```rs
    #[tokio::test]
    async fn test_re_edit_conversation() {
        // this test checks that if we edit a variant, the new variant gets linked to
        // the original canonical call, not to the variant that was edited
        test_llm_api_call(
            function_name!(),
            "api/sample-calls/chat-re-edit-conversation.yaml",
        )
        .await;
    }
```

The new code references a new struct that we must define in `src-tauri\src\models\llm_calls\linkage.rs`:

```rs
...
use crate::schema::{..., llm_call_variants};
...

#[derive(Insertable)]
#[diesel(table_name = llm_call_variants)]
pub struct NewLlmCallVariant<'a> {
    pub canonical_id: &'a EntityId,
    pub variant_id: &'a EntityId,
}

```

We expose this in `src-tauri\src\models\llm_calls\mod.rs`:

```rs
pub use linkage::{..., NewLlmCallVariant};
```

During testing, we find that the second re-edit only introduces a single variant link in, with no trace of the original to be found. We realize that this is because we're not inserting the links when reading them in from the database dump. We edit `src-tauri/src/test_helpers/database_contents.rs`:

```rs
...
use crate::models::llm_calls::{
    ..., NewLlmCallVariant, ...,
};
...
use crate::schema::{..., llm_call_variants, ...};
...

impl DatabaseContents {
    ...

    pub fn insertable_call_variants(&self) -> Vec<NewLlmCallVariant> {
        self.llm_calls
            .iter()
            .flat_map(|k| k.as_variant_rows())
            .collect()
    }
}

...

pub async fn read_database_contents(
    ...
) -> ZammResult<()> {
    ...
    db.transaction::<(), diesel::result::Error, _>(|conn| {
        ...
        diesel::insert_into(llm_call_variants::table)
            .values(&db_contents.insertable_call_variants())
            .execute(conn)?;
        Ok(())
    })?;
    Ok(())
}

pub fn dump_sqlite_database(db_path: &PathBuf, dump_path: &PathBuf) {
    let dump_output = std::process::Command::new("sqlite3")
        ...
        .arg(".dump ... llm_call_variants")
        ...;
    ...
}
```

This necessitates editing `src-tauri\src\models\llm_calls\llm_call.rs` as well:

```rs
...
#[cfg(test)]
use crate::models::llm_calls::{..., NewLlmCallVariant, ...};
...

#[cfg(test)]
impl LlmCall {
    ...

    pub fn as_variant_rows(&self) -> Vec<NewLlmCallVariant> {
        self.variation
            .variants
            .iter()
            .map(|variant| NewLlmCallVariant {
                canonical_id: &self.id,
                variant_id: &variant.id,
            })
            .collect()
    }
}
```

This time around, we edit `src-tauri/src/test_helpers/api_testing.rs` to automatically copy output database files for us if the gold files are missing, just like how we do the same for the recorded network requests. This approach admittedly means that tests will still succeed if we only delete the YAML dump file, but that shouldn't be too much of a concern since the normal mode of operation will still be deleting the entire folder.

```rs
...

fn copy_file_if_missing(
    expected_file_path: impl AsRef<Path>,
    actual_file_path: impl AsRef<Path>,
    fail_on_copy: bool,
) {
    let expected_path_abs = expected_file_path.as_ref().absolutize().unwrap();
    let actual_path_abs = actual_file_path.as_ref().absolutize().unwrap();

    if !expected_path_abs.exists() {
        fs::create_dir_all(expected_path_abs.parent().unwrap()).unwrap();
        fs::copy(&actual_path_abs, &expected_path_abs).unwrap();
        if fail_on_copy {
            panic!(
                "Gold file not found at {}, copied actual file from {}",
                expected_path_abs.as_ref().display(),
                actual_path_abs.as_ref().display(),
            );
        } else {
            eprintln!(
                "Gold file not found at {}, copied actual file from {}",
                expected_path_abs.as_ref().display(),
                actual_path_abs.as_ref().display(),
            );
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

        // check the call against db side-effects
        if let Some(test_db) = &side_effects_helpers.db {
            ...

            copy_file_if_missing(
                sample.db_end_dump("yaml"),
                &actual_db_yaml_dump,
                false,
            );
            copy_file_if_missing(
                sample.db_end_dump("sql"),
                &actual_db_sql_dump,
                true,
            );

            compare_files(
                sample.db_end_dump("yaml"),
                &actual_db_yaml_dump,
                ...
            );
            compare_files(
                sample.db_end_dump("sql"),
                &actual_db_sql_dump,
                ...
            );
        }

        ...
    }
}
```

`src-tauri/api/sample-network-requests/edit-conversation.json` and `src-tauri/api/sample-network-requests/re-edit-conversation.json` automatically get recorded by the VCR test code. Similarly for the database, we now have `src-tauri\api\sample-database-writes\conversation-edited\dump.sql` automatically written out, which we just have to compare to `src-tauri\api\sample-database-writes\conversation-forked-step-2\dump.sql` to confirm that sure enough, the only changes are the addition of these two rows:

```sql
...
INSERT INTO llm_calls VALUES('f39a5017-89d4-45ec-bcbb-25c2bd43cfc1','2024-06-08 06:20:40.601356700','open_ai','gpt-4','gpt-4-0613',1.0,58,25,83,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"Hello, does this work?"},{"role":"AI","text":"Yes, it works. How can I assist you today?"},{"role":"Human","text":"Tell me a funny joke."}]}','{"role":"AI","text":"Sure, here is a light-hearted joke for you: \n\nWhy don''t scientists trust atoms?\n\nBecause they make up everything!"}');
...
INSERT INTO llm_call_variants VALUES('c13c1e67-2de3-48de-a34c-a32079c03316','f39a5017-89d4-45ec-bcbb-25c2bd43cfc1');
```

We do the same comparison for `src-tauri\api\sample-database-writes\conversation-edited-2\dump.sql`, checking that the only differences with `src-tauri\api\sample-database-writes\conversation-edited\dump.sql` are once again the *addition* of these two rows:

```sql
...
INSERT INTO llm_calls VALUES('7a35a4cf-f3d9-4388-bca8-2fe6e78c9648','2024-06-08 09:40:22.392223700','open_ai','gpt-4','gpt-4-0613',1.0,59,19,78,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"Hello, does this really work?"},{"role":"AI","text":"Yes, it works. How can I assist you today?"},{"role":"Human","text":"Tell me a funny joke."}]}','{"role":"AI","text":"Sure, here you go: Why don''t scientists trust atoms? Because they make up everything!"}');
...
INSERT INTO llm_call_variants VALUES('c13c1e67-2de3-48de-a34c-a32079c03316','7a35a4cf-f3d9-4388-bca8-2fe6e78c9648');
```

as opposed to the *replacement* of the previous `llm_call_variants` line, as had happened when the test code was not inserting the rows of the variants table for the dump it was reading in.

### Frontend feature development

Now we make use of this on the frontend. We update the Specta bindings, and then edit `src-svelte\src\routes\api-calls\new\ApiCallEditor.svelte` after discovering that [this](https://css-tricks.com/flexbox-truncated-text/) is how you get text ellipsis to show up for flex children:

```svelte
<script lang="ts" context="module">
  import type { ..., LlmCallReference } from "$lib/bindings";
  ...

  export const canonicalRef = writable<LlmCallReference | undefined>(undefined);
  ...

  export function getDefaultApiCall(): PromptType {
    return {
      type: "Chat",
      messages: [{ role: "System", text: "" }],
    };
  }

  export function resetNewApiCall() {
    canonicalRef.set(undefined);
    prompt.set(getDefaultApiCall());
  }
</script>

<script lang="ts">
  ...

  async function submitApiCall() {
    ...

    try {
      const createdLlmCall = await chat({
        ...
        temperature: ...,
        canonical_id: $canonicalRef?.id,
        prompt: ...,
      });
      ...
    }
    ...
  }
</script>

<InfoBox title="New API Call">
  {#if $canonicalRef}
    <div class="canonical-display">
      <span class="label">Original API call:</span>
      <a href="/api-calls/{$canonicalRef.id}">{$canonicalRef.snippet}</a>
    </div>
  {/if}

  ...
</InfoBox>

<style>
  .canonical-display {
    background-color: var(--color-offwhite);
    padding: 0.75rem;
    border-radius: var(--corner-roundness);
    display: flex;
    flex-direction: row;
    gap: 0.5rem;
  }

  .canonical-display .label {
    white-space: nowrap;
  }

  .canonical-display a {
    min-width: 0;
    flex: 1;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  ...
</style>
```

We wrap the default API call in a `getDefaultApiCall` function so that we can reuse it in tests, and also in case of any update shenanigans when `src-svelte\src\routes\api-calls\[slug]\Prompt.svelte` edits the message directly, because Svelte [does not appear](https://stackoverflow.com/a/70614916) to make copies of objects that are assigned to stores.

We update `src-svelte/src/routes/api-calls/new/ApiCallEditor.stories.ts` to demonstrate how it might look when we're editing an API call instead of creating one from scratch:

```ts
...
import SvelteStoresDecorator from "$lib/__mocks__/stores";
import { CONTINUE_CONVERSATION_CALL } from "../[slug]/sample-calls";

export default {
  ...,
  decorators: [SvelteStoresDecorator],
};

...

export const EditContinuedConversation: StoryObj = Template.bind({}) as any;
EditContinuedConversation.parameters = {
  stores: {
    apiCallEditing: {
      canonicalRef: {
        id: "c13c1e67-2de3-48de-a34c-a32079c03316",
        snippet:
          "Sure, here's a joke for you: Why don't scientists trust atoms? " +
          "Because they make up everything!",
      },
      prompt: CONTINUE_CONVERSATION_CALL.request.prompt,
    },
  },
};

```

This in turn requires that we update `src-svelte/src/lib/__mocks__/stores.ts`:

```ts
...
import {
  canonicalRef,
  getDefaultApiCall,
  prompt,
} from "../../routes/api-calls/new/ApiCallEditor.svelte";
import type {
  ...,
  Prompt,
  LlmCallReference,
} from "$lib/bindings";
...

interface ApiCallEditing {
  canonicalRef: LlmCallReference;
  prompt: Prompt;
}

interface Stores {
  ...
  apiCallEditing?: ApiCallEditing;
}

...

const SvelteStoresDecorator: Decorator = (
  ...
) => {
  ...
  canonicalRef.set(stores?.apiCallEditing?.canonicalRef);
  prompt.set(stores?.apiCallEditing?.prompt || getDefaultApiCall());
  ...
}
```

We should also add this new story to `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "new"],
    variants: [..., "edit-continued-conversation"],
    ...
  },
  ...
];
```

Now we edit `src-svelte\src\routes\api-calls\new\ApiCallEditor.test.ts`. Because this time around, we're editing an actual existing API call instead of creating one from scratch, we need to clear the existing input before typing a new one in:

```ts
...
import ApiCallEditor, {
  canonicalRef,
  prompt,
  ...,
} from "./ApiCallEditor.svelte";
...

describe("API call editor", () => {
  ...

  async function setNewMessage(role: string, message: string) {
    ...
    if (messageInput.value !== "") {
      await userEvent.clear(messageInput);
    }
    await userEvent.type(...);
    ...
  }

  ...

  test("can edit an existing API call", async () => {
    const snippet =
      "Sure, here's a joke for you: Why don't scientists trust atoms? " +
      "Because they make up everything!";
    canonicalRef.set({
      id: "c13c1e67-2de3-48de-a34c-a32079c03316",
      snippet,
    });
    prompt.set({
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
    });
    render(ApiCallEditor, {});
    const originalApiCallLabel = screen.getByText("Original API call:");
    const originalApiCallLink = originalApiCallLabel.nextElementSibling;
    if (originalApiCallLink === null) {
      throw new Error("Original API call link not found");
    }
    expect(originalApiCallLink).toHaveTextContent(snippet);
    expect(originalApiCallLink).toHaveAttribute(
      "href",
      "/api-calls/c13c1e67-2de3-48de-a34c-a32079c03316",
    );

    expect(tauriInvokeMock).not.toHaveBeenCalled();
    playback.addSamples(
      "../src-tauri/api/sample-calls/chat-edit-conversation.yaml",
    );
    await setNewMessage("Human", "Tell me a funny joke.");

    await userEvent.click(screen.getByRole("button", { name: "Submit" }));
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    expect(get(mockStores.page).url.pathname).toEqual(
      "/api-calls/f39a5017-89d4-45ec-bcbb-25c2bd43cfc1",
    );
  });
});

```

#### Setting the canonical reference stores

We want the individual API call page to have a button that lets us edit that API call. As such, we edit `src-svelte/src/routes/api-calls/[slug]/Actions.svelte` to add this button:

```svelte
<script lang="ts">
  ...
  import { canonicalRef, prompt } from "../new/ApiCallEditor.svelte";
  ...

  function editApiCall() {
    if (!apiCall) {
      snackbarError("API call not yet loaded");
      return;
    }

    canonicalRef.set({
      id: apiCall.id,
      snippet: apiCall.response.completion.text,
    });
    prompt.set(apiCall.request.prompt);

    goto("/api-calls/new/");
  }

  ...
</script>

<InfoBox title="Actions" ...>
  <div ...>
    <div class="button-container cut-corners outer">
      <Button unwrapped leftEnd ...>Edit API call</Button>
      <Button unwrapped rightEnd on:click={restoreConversation}>Restore conversation</Button>
    </div>
  </div>
</InfoBox>

<style>
  ...

  .button-container {
    display: flex;
  }

  .button-container :global(.left-end) {
    --cut-top-left: 8px;
  }

  .button-container :global(.right-end) {
    --cut-bottom-right: 8px;
  }

  @media (max-width: 35rem) {
    .button-container {
      flex-direction: column;
    }
  }
</style>

```

We empirically discover that 35rem is a good point to switch over from a horizontal to vertical layout for buttons on the app. Our styling changes call for some edits to `src-svelte/src/lib/controls/Button.svelte`:

```svelte
<script lang="ts">
  ...
  export let leftEnd = false;
  export let rightEnd = ...;
  ...
</script>

{#if unwrapped}
  <button
    ...
    class:left-end={leftEnd}
    ...
  >
    ...
  </button>
{:else}
  <button
    ...
    class:left-end={leftEnd}
    ...
  >
    <div ... class:left-end={leftEnd} ...>
      ...
    </div>
  </button>
{/if}

<style>
  ...

  .outer.left-end,
  .inner.left-end {
    --cut-bottom-right: 0.01rem;
  }

  ...
</style>
```

Next, we test our new button. But first, we refactor some test data out into a new file `src-svelte/src/routes/api-calls/new/test.data.ts`:

```ts
import type { Prompt } from "$lib/bindings";

export const EDIT_CANONICAL_REF = {
  id: "c13c1e67-2de3-48de-a34c-a32079c03316",
  snippet: "Sure, here's a joke for you: Why don't scientists trust atoms? " +
  "Because they make up everything!",
};

export const EDIT_PROMPT: Prompt = {
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

We edit `src-svelte\src\routes\api-calls\new\ApiCallEditor.test.ts` accordingly, to import these variables that were just moved out of that file:

```ts
...
import { EDIT_CANONICAL_REF, EDIT_PROMPT } from "./test.data";

describe("API call editor", () => {
  ...

  test("can edit an existing API call", async () => {
    canonicalRef.set(EDIT_CANONICAL_REF);
    prompt.set(EDIT_PROMPT);
    ...
    expect(originalApiCallLink).toHaveTextContent(EDIT_CANONICAL_REF.snippet);
    expect(originalApiCallLink).toHaveAttribute(
      "href",
      ...
    );
    ...
  });
});
```

Note that we don't define and export the test data from this file directly, because if we did that, calling `vitest` on just `src-svelte\src\routes\api-calls\[slug]\ApiCall.test.ts` would cause Vitest to also run the tests on this imported test file.

Naturally, our next step is to also import this data and use it in `src-svelte/src/routes/api-calls/[slug]/ApiCall.test.ts`:

```ts
...
import { canonicalRef, prompt, getDefaultApiCall, resetNewApiCall } from "../new/ApiCallEditor.svelte";
import { EDIT_PROMPT } from "../new/test.data";
...

describe("Individual API call", () => {
  ...

  beforeEach(() => {
    ...
    mockStores.page.set({
      url: new URL("http://localhost/"),
      params: {},
    });
    resetConversation();
    resetNewApiCall();
  });

  afterEach(() => {
    vi.unstubAllGlobals();
  });

  test("can edit API call", async() => {
    playback.addSamples(
      "../src-tauri/api/sample-calls/get_api_call-continue-conversation.yaml",
    );
    render(ApiCall, { id: "c13c1e67-2de3-48de-a34c-a32079c03316" });
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    await waitFor(() => {
      screen.getByText("Hello, does this work?");
    });
    expect(get(canonicalRef)).toBeUndefined();
    expect(get(prompt)).toEqual(getDefaultApiCall());
    expect(get(mockStores.page).url.pathname).toEqual("/");

    const editButton = await waitFor(() =>
      screen.getByText("Edit API call"),
    );
    userEvent.click(editButton);
    await waitFor(() => {
      expect(get(prompt)).toEqual(EDIT_PROMPT);
    });
    expect(get(canonicalRef)).toEqual(EDIT_CANONICAL_REF);
    expect(get(mockStores.page).url.pathname).toEqual("/api-calls/new/");
  });
});

```

It is good to use the same exact data across these two tests so that we can be sure the handoff takes place correctly.

We notice that on the development Tauri app on Windows, the new API call editing page is incorrectly rendered by the individual API svelte component. This is because SvelteKit routing appears to get confused by the slug, such that it thinks `/api-calls/new/` should be handled by the `/api-calls/[slug]` route. It turns out we can fix this by editing the prerender entries in `src-svelte/svelte.config.js` to explicitly include the `/api-calls/new/` page:

```js
const config = {
  ...

  kit: {
    ...
    prerender: {
      ...,
      entries: ["*", "/api-calls/new/", "/api-calls/[slug]"],
    },
  },
};
```

#### Showing variant data

We first edit the backend tests to return API calls that show the variant data. We edit `src-tauri\api\sample-calls\get_api_call-continue-conversation.yaml` to return from a version of the database that has variant data on the given API call:

```yaml
...
response:
  message: >
    {
      "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
      ...,
      "variation": {
        "variants": [
          {
            "id": "f39a5017-89d4-45ec-bcbb-25c2bd43cfc1",
            "snippet": "Sure, here is a light-hearted joke for you: Why don't scientists trust atoms? Because they make up everything!..."
          },
          {
            "id": "7a35a4cf-f3d9-4388-bca8-2fe6e78c9648",
            "snippet": "Sure, here you go: Why don't scientists trust atoms? Because they make up everything!"
          }
        ]
      }
    }
sideEffects:
  database:
    startStateDump: conversation-edited-2
    endStateDump: conversation-edited-2

```

We then also create `src-tauri\api\sample-calls\get_api_call-edit.yaml` to show the return value when the API call in question is a variant and not the canonical one:

```yaml
request:
  - get_api_call
  - >
    {
      "id": "7a35a4cf-f3d9-4388-bca8-2fe6e78c9648"
    }
response:
  message: >
    {
      "id": "7a35a4cf-f3d9-4388-bca8-2fe6e78c9648",
      "timestamp": "2024-06-08T09:40:22.392223700",
      "llm": {
        "name": "gpt-4-0613",
        "requested": "gpt-4",
        "provider": "OpenAI"
      },
      "request": {
        "prompt": {
          "type": "Chat",
          "messages": [
            {
              "role": "System",
              "text": "You are ZAMM, a chat program. Respond in first person."
            },
            {
              "role": "Human",
              "text": "Hello, does this really work?"
            },
            {
              "role": "AI",
              "text": "Yes, it works. How can I assist you today?"
            },
            {
              "role": "Human",
              "text": "Tell me a funny joke."
            }
          ]
        },
        "temperature": 1.0
      },
      "response": {
        "completion": {
          "role": "AI",
          "text": "Sure, here you go: Why don't scientists trust atoms? Because they make up everything!"
        }
      },
      "tokens": {
        "prompt": 59,
        "response": 19,
        "total": 78
      },
      "variation": {
        "canonical": {
          "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
          "snippet": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
        },
        "sibling_variants": [
          {
            "id": "f39a5017-89d4-45ec-bcbb-25c2bd43cfc1",
            "snippet": "Sure, here is a light-hearted joke for you: Why don't scientists trust atoms? Because they make up everything!..."
          },
          {
            "id": "7a35a4cf-f3d9-4388-bca8-2fe6e78c9648",
            "snippet": "Sure, here you go: Why don't scientists trust atoms? Because they make up everything!"
          }
        ]
      }
    }
sideEffects:
  database:
    startStateDump: conversation-edited-2
    endStateDump: conversation-edited-2

```

We edit `src-tauri/src/commands/llms/get_api_call.rs` to include this test:

```rs
    #[tokio::test]
    async fn test_get_api_call_edit() {
        check_get_api_call_sample(
            function_name!(),
            "./api/sample-calls/get_api_call-edit.yaml",
        )
        .await;
    }
```

Then, we edit `src-svelte\src\routes\api-calls\[slug]\sample-calls.ts` to update `CONTINUE_CONVERSATION_CALL` and include `VARIANT_CALL` with the contents of the response returned in `src-tauri\api\sample-calls\get_api_call-edit.yaml`.

We refactor out LLM call reference link logic into `src-svelte/src/lib/ApiCallReferenceLink.svelte`:

```svelte
<script lang="ts">
  import type { LlmCallReference } from "$lib/bindings";

  export let apiCall: LlmCallReference;
  export let nolink = false;
</script>

{#if nolink}
  <span>{apiCall.snippet}</span>
{:else}
  <a href="/api-calls/{apiCall.id}">{apiCall.snippet}</a>
{/if}

<style>
  span, a {
    min-width: 0;
    flex: 1;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  span {
    font-style: italic;
  }
</style>

```

and `src-svelte/src/lib/ApiCallReference.svelte`, which will trigger ellipsis in case the parent container does not have `display: flex;`:

```svelte
<script lang="ts">
  import ApiCallReferenceLink from "./ApiCallReferenceLink.svelte";
  import type { LlmCallReference } from "./bindings";

  export let apiCall: LlmCallReference;
  export let selfContained = false;
</script>

{#if selfContained}
  <div class="ellipsis-container">
    <ApiCallReferenceLink {apiCall} {...$$restProps} />
  </div>
{:else}
  <ApiCallReferenceLink {apiCall} {...$$restProps} />
{/if}

<style>
  .ellipsis-container {
    display: flex;
    width: 100%;
  }
</style>

```

We define the `apiCall` argument explicitly instead of including it in `restProps`, because otherwise Svelte complains about that mandatory attribute not being included. [This](https://github.com/sveltejs/language-tools/issues/1377) may be a potentially related issue.

With this, we can now edit `src-svelte/src/routes/api-calls/new/ApiCallEditor.svelte` to use this new component and remove the styling for `.canonical-display a`:

```svelte
<script lang="ts">
  import ApiCallReference from "$lib/ApiCallReference.svelte";
  ...
</script>

<InfoBox title="New API Call">
  {#if $canonicalRef}
    <div class="canonical-display">
      ...
      <ApiCallReference apiCall={$canonicalRef} />
    </div>
  {/if}
  ...
</InfoBox>

<style>
  ...
</style>
```

We can now use this in `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.svelte` as well:

```svelte
<script lang="ts">
  ...
  import ApiCallReference from "$lib/ApiCallReference.svelte";
  ...

  function getThisAsRef(apiCall: LlmCall | undefined) {
    if (!apiCall) {
      return;
    }

    return {
      id: apiCall.id,
      snippet: apiCall.response.completion.text,
    };
  }

  $: thisAsRef = getThisAsRef(apiCall);
  $: variants = apiCall?.variation?.variants ?? apiCall?.variation?.sibling_variants ?? [];
</script>

<InfoBox title="API Call">
  {#if apiCall}
    ...
    {#if apiCall?.variation !== undefined}
      <SubInfoBox subheading="Variants">
        <div class="variation-links composite-reveal">
          {#if apiCall.variation.canonical}
            <ApiCallReference selfContained apiCall={apiCall.variation.canonical} />
          {:else if thisAsRef}
            <ApiCallReference selfContained nolink apiCall={thisAsRef} />
          {/if}

          <ul>
            {#each variants as variant}
              <li>
                <ApiCallReference selfContained nolink={variant.id === apiCall.id} apiCall={variant} />
              </li>
            {/each}
          </ul>
        </div>
      </SubInfoBox>
    {/if}
    ...
  {/if}
</InfoBox>

<style>
  ...

  .variation-links ul {
    margin: 0;
    padding: 0;
    list-style-type: circle;
  }

  .variation-links li {
    display: list-item;
    margin-left: 1rem;
    width: calc(100% - 1rem);
  }

  ...
</style>
```

where it turns out we have to set [these CSS properties](https://stackoverflow.com/a/3444102) for the list item icons to actually show up.

We add a story to `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.stories.ts` that showcases how the page gets rendered for a variant API call:

```ts
...
import {
  ...,
  VARIANT_CALL,
} from "./sample-calls";

...

export const Variant: StoryObj = Template.bind({}) as any;
Variant.args = {
  apiCall: VARIANT_CALL,
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};

```

We add this story to `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "individual"],
    variants: [
      ...,
      "variant",
    ],
    ...
  },
  ...
];
```

##### Rounded tree list

We take inspiration from [this answer](https://stackoverflow.com/a/67702407). [This repo](https://github.com/xotonic/flex-tree) appears interesting too, but we will go with the first style for now. We edit `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.svelte` as such:

```css
  .variation-links ul {
    ...
    list-style-type: none;
  }

  .variation-links li {
    --indent: 2rem;
    --line-thickness: 2px;
    margin-left: var(--indent);
    width: calc(100% - var(--indent));
    position: relative;
  }

  .variation-links li:before {
    content: "";
    position: absolute;
    bottom: 50%;
    left: calc(-1 * var(--indent) + 0.5rem);
    width: 1rem;
    height: 150%;
    border-bottom: var(--line-thickness) solid var(--color-border);
    border-left: var(--line-thickness) solid var(--color-border);
    border-bottom-left-radius: var(--corner-roundness);
  }

  .variation-links li:first-child:before {
    height: 50%;
  }
```

Next, we find out that `src-svelte\src\routes\api-calls\[slug]\ApiCall.test.ts` is now failing because the response text appears twice in the document. We edit the test to fix this, and to also test for the new conversation and variant links we've added:

```ts
  test("can load API call with correct details", async () => {
    ...

    // check that AI message is displayed
    const responseSection = await screen.findByLabelText("Response");
    const response = responseSection.querySelector("pre");
    expect(response).toHaveTextContent("Sure, here's a joke for you: Why don't scientists trust atoms? " +
    "Because they make up everything!");

    // check that metadata is displayed
    ...

    // check that links are displayed
    const conversationSection = await screen.findByLabelText("Conversation");
    const conversationLinks = Array.from(conversationSection.querySelectorAll("a")).map((a) => a.href);
    expect(conversationLinks).toEqual([
      // previous
      "http://localhost:3000/api-calls/d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
      // next
      "http://localhost:3000/api-calls/0e6bcadf-2b41-43d9-b4cf-81008d4f4771",
      "http://localhost:3000/api-calls/63b5c02e-b864-4efe-a286-fbef48b152ef",
    ]);

    const variantSection = await screen.findByLabelText("Variants");
    const variantLinks = Array.from(variantSection.querySelectorAll("a")).map((a) => a.href);
    expect(variantLinks).toEqual([
      "http://localhost:3000/api-calls/f39a5017-89d4-45ec-bcbb-25c2bd43cfc1",
      "http://localhost:3000/api-calls/7a35a4cf-f3d9-4388-bca8-2fe6e78c9648",
    ]);
  });
```
