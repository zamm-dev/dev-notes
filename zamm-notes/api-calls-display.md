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
