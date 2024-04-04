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

Note that we make use of the `--color-hover` and the `EmptyPlaceholder` that we just refactored above. We also make use of the resize trick from the Chat component to size the message quote appropriately.

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

We bubble this up in `src-svelte\src\lib\Scrollable.svelte`. There are perhaps more idiomatic ways of bubbling events up, but since this is our first time doing this and we are just getting the component set up, we minimize the number of failure points for easier debugging in case something doesn't work:

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

We notice that 10 items in the list is actually not enough to fill the whole thing up. The bottom indicator never gets triggered again because it stays visible after the new API calls are loaded. We should change the default page size. However, to do so, we'd also need to update our tests. We should write and commit a proper script to generate the test files for us.
