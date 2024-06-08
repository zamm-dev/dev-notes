# Chat feature

## Initial implementation

### Python

First we set up our dependencies by editing `src-python/pyproject.toml`:

```toml
...

[tool.poetry.dependencies]
...
openai = "^1.6.1"
langchain = "^0.1.0"
langchain-openai = "^0.0.2"

[tool.poetry.group.dev.dependencies]
...
vcr-langchain = "^0.0.31"

...
```

Then we create a sample call file at `src-python/api/sample-calls/chat.yaml` to work off of:

```bash
$ request:
  - chat
  - >
    {
      "provider": "OpenAI",
      "api_key": "dummy-key",
      "llm": "gpt-3.5-turbo-1106",
      "prompt": [
        {
          "role": "System",
          "message": "You are the brains of ZAMM, a chat program."
        },
        {
          "role": "Human",
          "message": "Hello, does this work?"
        }
      ]
    }
response:
  message: >
    {
      "response": "LLM says it does work."
    }

```

We don't actually know how the response will look yet, but at least we'll start off with enough information to send the request. We then extract the request JSON part into a `chat.json` and create a schema using:

```bash
$ yarn quicktype src-python/api/sample-calls/chat.json -o src-python/api/schemas/chat_args.json -l schema
```

We review the generated `src-python/api/schemas/chat_args.json` to check that it looks reasonable:

```json
{
    "$schema": "http://json-schema.org/draft-06/schema#",
    "$ref": "#/definitions/ChatArgs",
    "definitions": {
        "ChatArgs": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
                "provider": {
                    "type": "string"
                },
                "api_key": {
                    "type": "string"
                },
                "llm": {
                    "type": "string"
                },
                "prompt": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/Prompt"
                    }
                }
            },
            "required": [
                "api_key",
                "llm",
                "prompt",
                "provider"
            ],
            "title": "ChatArgs"
        },
        "Prompt": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
                "role": {
                    "type": "string"
                },
                "message": {
                    "type": "string"
                }
            },
            "required": [
                "message",
                "role"
            ],
            "title": "Prompt"
        }
    }
}

```

We rename `Prompt` to `ChatMessage` because it's a list of chat messages:

```json
{
    ...
    "definitions": {
        "ChatArgs": {
            ...,
            "properties": {
                ...,
                "prompt": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/ChatMessage"
                    }
                }
            },
            ...
        },
        "ChatMessage": {
            ...
            "title": "ChatMessage"
        }
    }
}
```

Next, we generate the Python classes using:

```bash
$ make quicktype
```

To make it easier to experiment with quicktype generation from these JSON files, we can edit the main `Makefile` to only consider JSON files in that directory, instead of a total `*` glob:

```Makefile
quicktype:
	yarn quicktype src-python/api/schemas/*.json ...
	yarn quicktype src-python/api/schemas/*.json ...
	...
```

With a bit more experimentation, we find that we want our chat response to similarly make use of the `ChatMessage` schema, but quicktype will not realize it's the same object and name it a different class in the generated Python model file. As such, we:

- put both chat arg and response types in the same file
- make one single top-level class called `ChatMethod`, because quicktype ignores any definitions that are not part of the top-level class
- add a `temperature` setting to the arguments
- rename `src-python/api/schemas/chat_args.json` to `src-python/api/schemas/chat_method.json`

With these changes, the file looks like:

```json
"$schema": "http://json-schema.org/draft-06/schema#",
    "type": "object",
    "additionalProperties": false,
    "title": "ChatMethod",
    "properties": {
        "args": {
            "$ref": "#/definitions/ChatArgs"
        },
        "response": {
            "$ref": "#/definitions/ChatResponse"
        }
    },
    "required": [
        "args",
        "response"
    ],
    "definitions": {
        "ChatArgs": {
            ...,
            "properties": {
                ...
                "temperature": {
                    "type": "number"
                },
                ...
            },
            ...
        },
        "ChatResponse": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
                "response": {
                    "$ref": "#/definitions/ChatMessage"
                },
                "tokens": {
                    "$ref": "#/definitions/TokenMetadata"
                }
            },
            "required": [
                "response",
                "tokens"
            ],
            "title": "ChatResponse"
        },
        ...
        "TokenMetadata": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
                "total": {
                    "type": "integer"
                },
                "prompt": {
                    "type": "integer"
                },
                "completion": {
                    "type": "integer"
                },
                "cost": {
                    "type": "number"
                }
            },
            "required": [
                "completion",
                "cost",
                "prompt",
                "total"
            ],
            "title": "TokenMetadata"
        }
    }
}

```

The only new model in `src-python/zamm/api/models.py` that we're not going to make use of is `ChatMethod`. As for the rest of the fields, we make use of them in the final `src-python/api/sample-calls/chat.yaml`, specifying a different temperature than the regular langchain default of 0.7 to make sure that it's being passed through correctly:

```yaml
request:
  - chat
  - >
    {
      ...
      "temperature": 0.65,
      ...
    }
response:
  message: >
    {
      "response": {
        "role": "AI",
        "message": "Hello! Yes, it works. How can I assist you today?"
      },
      "tokens": {
        "total": 44,
        "prompt": 30,
        "completion": 14,
        "cost": 0.000058
      }
    }

```

We check out `src-python/tests/api/sample_call.py` in Git after the `make quicktype` command because the definition of sample calls haven't changed, and we need that custom type definition that we manually added in for mypy.

Next, we actually implement the new API method in `src-python/zamm/api/chat.py`:

```py
"""API for sending LLMs a single prompt."""

from langchain_openai import ChatOpenAI
from zamm.api.methods import ApiMethod
from zamm.api.models import ChatArgs, ChatMessage, ChatResponse, TokenMetadata
from langchain_community.callbacks import get_openai_callback
from langchain_core.messages import (
    BaseMessage as LCBaseMessage,
    HumanMessage,
    SystemMessage,
    AIMessage
)
from langchain_core.outputs import ChatGeneration
from langchain_core.prompt_values import ChatPromptValue
from typing import cast
import os


def to_langchain_prompt(messages: ChatMessage) -> ChatPromptValue:
    """Convert a list of our ChatMessage to a Langchain prompt."""
    lc_messages: list[LCBaseMessage] = []
    for message in messages:
        if message.role == "Human":
            lc_messages.append(HumanMessage(content=message.message))
        elif message.role == "AI":
            lc_messages.append(AIMessage(content=message.message))
        elif message.role == "System":
            lc_messages.append(SystemMessage(content=message.message))
        else:
            raise ValueError(f"Unknown role {message.role}")
    return ChatPromptValue(messages=lc_messages)


def chat(args: ChatArgs) -> ChatResponse:
    """Send a chat message to the LLM."""
    if args.provider == "OpenAI":
        if "ZAMM_DUMMY_API_KEYS" in os.environ:
            api_key = os.environ["OPENAI_API_KEY"]
        else:
            api_key = args.api_key
        llm = ChatOpenAI(api_key=api_key, model=args.llm, temperature=args.temperature)
    else:
        raise NotImplementedError(f"Provider {args.provider} not yet supported")
    
    prompt = to_langchain_prompt(args.prompt)
    with get_openai_callback() as cb:
        result = cast(ChatGeneration, llm.generate_prompt([prompt]).generations[0][0])
        return ChatResponse(
            response=ChatMessage(message=result.text, role="AI"),
            tokens=TokenMetadata(
                completion=cb.completion_tokens,
                prompt=cb.prompt_tokens,
                total=cb.total_tokens,
                cost=cb.total_cost,
            )
        )


chat_method = ApiMethod(ChatArgs, str, chat)

```

We edit `src-python/Makefile` next to make sure that `ZAMM_DUMMY_API_KEYS` is being set before the test:

```Makefile
tests: export ZAMM_DUMMY_API_KEYS = true
tests:
	poetry run pytest -v

```

We need to refer to this new method in the execution declarations file, and to make that convenient we re-export it and the greet method in `src-python/zamm/api/__init__.py`:

```py
"""API methods for the Python CLI."""

from zamm.api.greet import greet_method
from zamm.api.chat import chat_method

__all__ = ["greet_method", "chat_method"]

```

Now we can import both in one line in `src-python/zamm/execution.py`:

```py
...

from zamm.api import chat_method, greet_method

METHODS = {
    "chat": chat_method,
    ...
}

...
```

We test this in `src-python/tests/api/test_chat.py`:

```py
"""Test that chat invocations work."""

from tests.api.test_helpers import compare_io
import vcr_langchain as vcr


@vcr.use_cassette()
def test_openai_chat() -> None:
    """Make sure a regular chat message works."""
    compare_io("api/sample-calls/chat.yaml")

```

We run the tests and manually check the newly recorded API call to OpenAI at `src-python/tests/api/test_openai_chat.yaml` to ensure that it's calling with the correct content:

```yaml
interactions:
- request:
    body: '{"messages": [{"role": "system", "content": "You are the brains of ZAMM,
      a chat program."}, {"role": "user", "content": "Hello, does this work?"}], "model":
      "gpt-3.5-turbo-1106", ..., "temperature": 0.65}'
    headers:
      host:
      - api.openai.com
      x-stainless-async:
      - 'false'
    method: POST
    uri: https://api.openai.com/v1/chat/completions
  response:
    content: "{...\"model\": \"gpt-3.5-turbo-1106\",\n
      ... \"message\": {\n        \"role\":
      \"assistant\",\n        \"content\": \"Hello! Yes, it works. How can I assist
      you today?\"\n      },... \"usage\": {\n    \"prompt_tokens\": 30,\n    \"completion_tokens\":
      14,\n    \"total_tokens\": 44\n  },..."
    headers:
      ...
```

We also make sure that the test runs successfully the next time with `ZAMM_DUMMY_API_KEYS` unset, to ensure that it is actually replaying cached API calls to OpenAI.

The tests all pass, but annoying warnings are generated. We edit `src-python/pyproject.toml` again to make warnings a failure, and to ignore these warnings:

```toml
...

[tool.pytest.ini_options]
filterwarnings = [
    "error",
    'ignore::pydantic.warnings.PydanticDeprecatedSince20',
]

...
```

#### Commit errors

We run into a bunch of issues with the quicktype-generated file and mypy. We go through them:

```
src-python/zamm/api/models.py:27: error: Function is missing a type annotation  [no-untyped-def]
```

This refers to

```py
def from_union(fs, x):
    ...
```

We change it again as we did in [`api-boundary-type-safety.md`](/general-notes/setup/tauri/api-boundary-type-safety.md):

```py
def from_union(fs: List[Callable], x: Any) -> Any:
    ...
```

Next:

```
src-python/zamm/api/chat.py:20: error: "ChatMessage" has no attribute "__iter__" (not iterable)  [attr-defined]
```

It is:

```py
def to_langchain_prompt(messages: ChatMessage) -> ChatPromptValue:
    ...
    for message in messages:
        ...
```

We fix with:

```py
def to_langchain_prompt(messages: list[ChatMessage]) -> ChatPromptValue:
    ...
    for mesage in messages:
        ...
```

This also fixes the later error

```
src-python/zamm/api/chat.py:43: error: Argument 1 to "to_langchain_prompt" has incompatible type "list[ChatMessage]"; expected "ChatMessage"  [arg-type]
```

at

```py
def chat(args: ChatArgs) -> ChatResponse:
    ...
    prompt = to_langchain_prompt(args.prompt)
    ...
```

Next:

```
src-python/zamm/api/chat.py:57: error: Argument 3 to "ApiMethod" has incompatible type "Callable[[ChatArgs], ChatResponse]"; expected "Callable[[ChatArgs], str]"  [arg-type]
```

caused by

```py
chat_method = ApiMethod(ChatArgs, str, chat)
```

We fix with:

```py
chat_method = ApiMethod(ChatArgs, ChatResponse, chat)
```

Next:

```
src-python/zamm/execution.py:17: error: "object" has no attribute "args_type"  [attr-defined]
src-python/zamm/execution.py:18: error: "object" has no attribute "invoke"  [attr-defined]
```

caused by:

```py
...

METHODS = {
    "chat": chat_method,
    "greet": greet_method,
}


def handle_commandline_args(method_name: str, args_dict_str: str) -> str:
    """Handle commandline arguments."""
    method = METHODS[method_name]
    args_dict = json.loads(args_dict_str)
    args = method.args_type.from_dict(args_dict)
    response = method.invoke(args)
    ...

```

We fix by editing `src-python/zamm/api/__init__.py` again to export `ApiMethod`:

```py
...
from zamm.api.methods import ApiMethod

__all__ = [..., "ApiMethod"]

```

and then adding better type hinting to `src-python/zamm/execution.py`:

```py
...

from zamm.api import ApiMethod, ...

METHODS: dict[str, ApiMethod] = {
    ...
}

...
```

#### Saving model name

We realize that OpenAI returns the specific model used in the response, which may be different from the more general model requested. We should save this information as well. We edit `src-python/api/schemas/chat_method.json` in preparation:

```json
{
  "$schema": "http://json-schema.org/draft-06/schema#",
  ...
  "definitions": {
    ...
    "ChatResponse": {
      ...
      "properties": {
        ...
        "llm": {
          "type": "string"
        }
      },
      "required": [..., "llm"],
      ...
    },
    ...
  }
}
```

We edit the existing file `src-python/api/sample-calls/chat.yaml`:

```yaml
request:
  - chat
  - >
    {
      ...,
      "llm": "gpt-3.5-turbo-1106",
      ...
    }
response:
  message: >
    {
      ...,
      "llm": "gpt-3.5-turbo-1106"
    }
```

and create a new one at `src-python/api/sample-calls/chat_reply.yaml` where we demonstrate this in action:

```yaml
request:
  - chat
  - >
    {
      "provider": "OpenAI",
      "api_key": "dummy-key",
      "llm": "gpt-3.5-turbo",
      "temperature": 0.3,
      "prompt": [
        {
          "role": "System",
          "message": "You are the brains of ZAMM, a chat program."
        },
        {
          "role": "Human",
          "message": "Hello, does this work?"
        },
        {
          "role": "AI",
          "message": "Hello! Yes, it works. How can I assist you today?"
        },
        {
          "role": "Human",
          "message": "How do you start a new Tauri project?"
        }
      ]
    }
response:
  message: >
    {
      "response": {
        "role": "AI",
        "message": "TBD"
      },
      ...
    }

```

We edit `src-python/tests/api/test_chat.py` to make this new sample call:

```py
@vcr.use_cassette()
def test_openai_chat_reply() -> None:
    """Make sure a response to an AI message works."""
    compare_io("api/sample-calls/chat_reply.yaml")

```

The file `src-python/tests/api/test_openai_chat_reply.yaml` gets recorded:

```yaml
interactions:
- request:
    body: '{"messages": [{"role": "system", "content": "You are the brains of ZAMM,
      a chat program."}, {"role": "user", "content": "Hello, does this work?"}, {"role":
      "assistant", "content": "Hello! Yes, it works. How can I assist you today?"},
      {"role": "user", "content": "How do you start a new Tauri project?"}], "model":
      "gpt-3.5-turbo", "n": 1, "stream": false, "temperature": 0.3}'
    headers:
      host:
      - api.openai.com
      x-stainless-async:
      - 'false'
    method: POST
    uri: https://api.openai.com/v1/chat/completions
  response:
    content: "{\n  \"id\": \"chatcmpl-8fbMpBNgyxD4RF9e2CcdsPWtgJ3qq\",\n  \"object\":
      \"chat.completion\",\n  \"created\": 1704925779,\n  \"model\": \"gpt-3.5-turbo-0613\",\n
      \ \"choices\": [\n    {\n      \"index\": 0,\n      \"message\": {\n        \"role\":
      \"assistant\",\n        \"content\": \"To start a new Tauri project, you can
      follow these steps:\\n\\n1. Install Node.js: Make sure you have Node.js installed
      on your machine. You can download it from the official Node.js website (https://nodejs.org).\\n\\n2.
      Install Tauri: Open your terminal or command prompt and run the following command
      to install Tauri globally:\\n   ```\\n   npm install -g tauri\\n   ```\\n\\n3.
      Create a new Tauri project: Navigate to the directory where you want to create
      your new Tauri project. Then, run the following command to create a new Tauri
      project:\\n   ```\\n   tauri init\\n   ```\\n\\n4. Configure your project: During
      the initialization process, you'll be prompted to answer a few questions to
      configure your project. You can choose the desired options based on your requirements.\\n\\n5.
      Build your project: Once the project is initialized, navigate into the project
      directory and run the following command to build your Tauri project:\\n   ```\\n
      \  tauri build\\n   ```\\n\\n6. Run your project: After the build process is
      complete, you can run your Tauri project using the following command:\\n   ```\\n
      \  tauri dev\\n   ```\\n\\nThese steps will help you get started with a new
      Tauri project. Remember to consult the official Tauri documentation (https://tauri.studio/)
      for more detailed information and additional configuration options.\"\n      },\n
      \     \"logprobs\": null,\n      \"finish_reason\": \"stop\"\n    }\n  ],\n
      \ \"usage\": {\n    \"prompt_tokens\": 63,\n    \"completion_tokens\": 295,\n
      \   \"total_tokens\": 358\n  },\n  \"system_fingerprint\": null\n}\n"
    headers:
      CF-Cache-Status:
      - DYNAMIC
      CF-RAY:
      - 84385c27eb03ef90-PDX
      Cache-Control:
      - no-cache, must-revalidate
      Connection:
      - keep-alive
      Content-Encoding:
      - gzip
      Content-Type:
      - application/json
      Date:
      - Wed, 10 Jan 2024 22:29:49 GMT
      Transfer-Encoding:
      - chunked
      openai-model:
      - gpt-3.5-turbo-0613
      openai-processing-ms:
      - '10154'
    http_version: HTTP/1.1
    status_code: 200
version: 1

```

Note that the request is for `gpt-3.5-turbo`, but the response comes back as `gpt-3.5-turbo-0613`. This is what we're looking to save. We go back and update the sample call file to reflect the expected output.

We do `make quicktype` again, and `src-python/zamm/api/models.py` gets updated.

We find that we have to create a custom OpenAI callback handler ourselves, because the default one does not save model information. We do so in `src-python/zamm/api/chat.py`:

```py
...
from typing import cast, Any

from langchain_community.callbacks import OpenAICallbackHandler
...


class CustomOpenAICallbackHandler(OpenAICallbackHandler):
    """Custom OpenAI callback handler."""

    model_name: str | None

    def __init__(self) -> None:
        """Initialize the OpenAI callback handler."""
        super().__init__()
        self.model_name = None

    def on_llm_end(self, response: LLMResult, **kwargs: Any) -> None:
        """Collect remaining metadata."""
        super().on_llm_end(response, **kwargs)
        self.model_name = response.llm_output["model_name"]


...

def chat(args: ChatArgs) -> ChatResponse:
    ...
    cb = CustomOpenAICallbackHandler()
    result = cast(ChatGeneration, llm.generate_prompt([prompt], callbacks=[cb]).generations[0][0])
    return ChatResponse(
        llm=cb.model_name or args.llm,
        response=ChatMessage(message=result.text, role="AI"),
        tokens=TokenMetadata(
            completion=cb.completion_tokens,
            prompt=cb.prompt_tokens,
            total=cb.total_tokens,
            cost=cb.total_cost,
        ),
    )

...
```

Unfortunately the tests fail. After digging in deeper to debug for a bit, we edit `langchain_openai/chat_models/base.py` of `langchain-openai` version 0.0.2 to print out the response:

```py
class ChatOpenAI(BaseChatModel):
    ...

    def _create_chat_result(self, response: Union[dict, BaseModel]) -> ChatResult:
        generations = []
        if not isinstance(response, dict):
            response = response.dict()
        print(response["model"])
        for res in response["choices"]:
            ...
        token_usage = response.get("usage", {})
        llm_output = {
            "token_usage": token_usage,
            "model_name": self.model_name,
            "system_fingerprint": response.get("system_fingerprint", ""),
        }
        return ChatResult(generations=generations, llm_output=llm_output)
```

`gpt-3.5-turbo-0613` gets printed correctly, but for some reason the returned model is actually being discarded completely in this code, with the original request model name being used instead.

At this point, we'll have to fork or patch `langchain-openai` ourselves, and the package doesn't even appear to be open-source. The whole point of using langchain was to enable us to quickly add new LLMs later on, and having to customize LLM-specific code to this level means that we might as well just use the OpenAI package directly. But if we're using the Python OpenAI package directly, we might as well just use the community-provided Rust package directly.

We commit everything just in case we want to revisit this path later, and go back to where we were before we started working on supporting the chat API in Python.

### Rust

The `async_openai` crate appears to be the most complete. We add that and `rVCR` for testing:

```bash
$ cargo add reqwest
$ cargo add async_openai
$ cargo add --dev rVCR
$ cargo add --dev reqwest_middleware
```

#### Capturing network calls for testing

Our goal is to capture a request as soon as possible so that we can start iterating on tests without waiting for slow network requests.

It turns out there is an [outstanding issue](https://github.com/ChorusOne/rvcr/issues/5) where reqwest-middleware version `0.2.x` breaks rVCR with a message such as

```
error[E0277]: expected a `Fn<(Request, &'a mut task_local_extensions::extensions::Extensions, Next<'a>)>` closure, found `VCRMiddleware`
  --> framework/tests/acceptance.rs:19:15
   |
19 |         .with(middleware)
   |          ---- ^^^^^^^^^^ expected an `Fn<(Request, &'a mut task_local_extensions::extensions::Extensions, Next<'a>)>` closure, found `VCRMiddleware`
   |          |
   |          required by a bound introduced by this call
   |
   = help: the trait `for<'a> Fn<(Request, &'a mut task_local_extensions::extensions::Extensions, Next<'a>)>` is not implemented for `VCRMiddleware`
   = note: required for `VCRMiddleware` to implement `reqwest_middleware::Middleware`
```

We therefore install the latest `0.1.x` version instead, which is `0.1.6` according to [this page](https://lib.rs/crates/reqwest-middleware/versions).

```
[dev-dependencies]
...
reqwest-middleware = "0.1.6"
...
```

and

```bash
$ cargo generate-lockfile
```

During implementation, we realize that we need `async-openai` to make use of `reqwest_middleware` clients instead of `reqwest` clients so that we can swap in test clients. As such, we move `reqwest-middleware` to the regular dependencies section instead of the dev dependencies section.

We edit `src-tauri/src/commands/errors.rs` to introduce new errors:

```rs
use std::{fmt, sync::PoisonError};
use crate::setup::api_keys::Service;

...

#[derive(thiserror::Error, Debug)]
pub enum Error {
    ...
    #[error("Missing API key for {service}")]
    MissingApiKey {
        service: Service,
    },
    #[error("Lock poisoned")]
    Poison {},
    ...
    #[error(transparent)]
    Reqwest {
        #[from]
        source: reqwest::Error,
    },
    #[error(transparent)]
    OpenAI {
        #[from]
        source: async_openai::error::OpenAIError,
    },
    ...
}

impl<T> From<PoisonError<T>> for Error {
    fn from(_: PoisonError<T>) -> Self {
        Self::Poison {}
    }
}

...
```

where the Poison error is something we copied from [this answer](https://stackoverflow.com/a/68804367) due to the presence of lifetime and type parameters that would make it impossible to derive automatically without unnecessarily complicating the `Error` enum to have lifetimes and the such.

We now create `src-tauri/src/commands/llms/chat.rs`:

```rs
use crate::commands::errors::ZammResult;
use crate::setup::api_keys::Service;
use crate::commands::Error;
use crate::ZammApiKeys;
use specta::specta;
use tauri::State;
use async_openai::config::OpenAIConfig;
use async_openai::{
    types::{
        ChatCompletionRequestAssistantMessageArgs, ChatCompletionRequestSystemMessageArgs,
        ChatCompletionRequestUserMessageArgs, CreateChatCompletionRequestArgs,
    },
};


async fn chat_helper(zamm_api_keys: &ZammApiKeys, http_client: reqwest_middleware::ClientWithMiddleware) -> ZammResult<()> {
    let api_keys = zamm_api_keys.0.lock()?;
    if api_keys.openai.is_none() {
        return Err(Error::MissingApiKey {
            service: Service::OpenAI,
        });
    }

    let config = OpenAIConfig::new()
      .with_api_key(api_keys.openai.as_ref().unwrap());

    
    let openai_client = async_openai::Client::with_config(config).with_http_client(http_client);
    let request = CreateChatCompletionRequestArgs::default()
        .model("gpt-3.5-turbo")
        .messages([
            ChatCompletionRequestSystemMessageArgs::default()
                .content("You are ZAMM, a chat program. Respond in first person.")
                .build()?
                .into(),
            ChatCompletionRequestUserMessageArgs::default()
                .content("Hello, does this work?")
                .build()?
                .into(),
        ])
        .build()?;
    let response = openai_client.chat().create(request).await?;
    println!("{:#?}", response);
    Ok(())
}

#[tauri::command(async)]
#[specta]
pub async fn chat(api_keys: State<'_, ZammApiKeys>) -> ZammResult<()> {
    let http_client = reqwest::ClientBuilder::new().build()?;
    let client_with_middleware = reqwest_middleware::ClientBuilder::new(http_client)
        .build();
    chat_helper(&api_keys, client_with_middleware).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::env;
    use crate::setup::api_keys::ApiKeys;
    use std::sync::Mutex;
    use reqwest_middleware::{ClientBuilder, ClientWithMiddleware};
    use rvcr::{VCRMiddleware, VCRMode};
    use std::path::PathBuf;

    #[tokio::test]
    async fn test_start_conversation() {
        let api_keys = ZammApiKeys(Mutex::new(ApiKeys {
            openai: env::var("OPENAI_API_KEY").ok(),
        }));

        let recording_path = PathBuf::from("api/sample-call-requests/start-conversation.json");
        let is_recording = !recording_path.exists();
        let middleware: VCRMiddleware = if is_recording {
            VCRMiddleware::try_from(recording_path)
            .unwrap()
            .with_mode(VCRMode::Record)
            .with_modify_request(|req| {
                let header_blacklist = ["authorization"];

                // Overwrite query params with filtered ones
                req.headers = req.headers.clone().iter().filter_map(
                    |(k, v)| {
                        if header_blacklist.contains(&k.as_str()) {
                            None
                        } else {
                            Some((k.clone(), v.clone()))
                        }
                    }
                ).collect();
            }).with_modify_response(|resp| {
                let header_blacklist = ["set-cookie", "openai-organization"];

                // Overwrite query params with filtered ones
                resp.headers = resp.headers.clone().iter().filter_map(
                    |(k, v)| {
                        if header_blacklist.contains(&k.as_str()) {
                            None
                        } else {
                            Some((k.clone(), v.clone()))
                        }
                    }
                ).collect();
            })
        } else {
            VCRMiddleware::try_from(recording_path)
            .unwrap()
            .with_mode(VCRMode::Replay)
        };

        let vcr_client: ClientWithMiddleware = ClientBuilder::new(reqwest::Client::new())
            .with(middleware)
            .build();

        let result = chat_helper(&api_keys, vcr_client).await;
        assert!(!result.is_ok());
    }
}

```

with just a debug print statement to see what output we're getting. We make the test fail so that the stdout output will show up. According to [this answer](https://stackoverflow.com/a/67156994), we have to use Tokio to get async tests to run in Rust:

```bash
$ cargo add --dev tokio --features macros
```

We pipe this command through to `src-tauri/src/commands/llms/mod.rs`:

```rs
mod chat;

pub use chat::chat;

```

and `src-tauri/src/commands/mod.rs`:

```rs
...
mod llms;

pub use errors::Error;
...
pub use llms::chat;
```

where we also re-export `errors::Error` for convenience. Now that the chat code file is accessible from the top-level of the project, `cargo build` will surface any compilation errors and make it easier for us to develop.

We need `async-openai` to make use of `request-middleware`, but the types cannot be converted into one another because `reqwest`'s Client is a struct, not a trait. As such, we clone `async-openai` into `forks/async-openai` and make changes on our own branch:

```bash
$ mkdir -p forks
$ cd forks
$ git clone git@github.com:amosjyng/async-openai.git
$ git checkout -b rvcr
```

We edit their `forks/async-openai/async-openai/Cargo.toml` to add our new package in:

```toml
...

[dependencies]
...
reqwest-middleware = "0.1.6"
...
```

We edit `forks/async-openai/async-openai/src/client.rs` to use the middleware client where possible:

```rs
...

#[derive(Debug, Clone)]
/// Client is a container for config, backoff and http_client
/// used to make API calls.
pub struct Client<C: Config> {
    http_client: reqwest_middleware::ClientWithMiddleware,
    streaming_http_client: reqwest::Client,
    ...
}

impl Client<OpenAIConfig> {
    /// Client with default [OpenAIConfig]
    pub fn new() -> Self {
        Self {
            http_client: reqwest_middleware::ClientBuilder::new(
                reqwest::Client::new(),
            ).build(),
            streaming_http_client: reqwest::Client::new(),
            ...
        }
    }
}

impl<C: Config> Client<C> {
    /// Create client with [OpenAIConfig] or [crate::config::AzureConfig]
    pub fn with_config(config: C) -> Self {
        Self {
            http_client: reqwest_middleware::ClientBuilder::new(
                reqwest::Client::new(),
            ).build(),
            streaming_http_client: reqwest::Client::new(),
            ...
        }
    }

    /// Provide your own [client] to make HTTP requests with.
    ///
    /// [client]: reqwest_middleware::ClientWithMiddleware
    pub fn with_http_client(mut self, http_client: reqwest_middleware::ClientWithMiddleware) -> Self {
        self.http_client = http_client;
        self
    }

    ...

    /// Execute a HTTP request and retry on rate limit
    ///
    /// ...
    async fn execute_raw<M, Fut>(&self, request_maker: M) -> Result<Bytes, OpenAIError>
    where
        M: Fn() -> Fut,
        Fut: core::future::Future<Output = Result<reqwest::Request, OpenAIError>>,
    {
            ...
            let response = client
                .execute(request)
                .await
                .map_err(OpenAIError::ReqwestMiddleware)
                ...;
            ...
    }

    ...

    /// Make HTTP POST request to receive SSE
    pub(crate) async fn post_stream<I, O>(
        &self,
        ...
    ) -> Pin<Box<dyn Stream<Item = Result<O, OpenAIError>> + Send>>
    where
        ...
    {
        let event_source = self
            .streaming_http_client
            ...
            .eventsource()
            .unwrap();

        stream(event_source).await
    }

    /// Make HTTP GET request to receive SSE
    pub(crate) async fn get_stream<Q, O>(
        &self,
        ...
    ) -> Pin<Box<dyn Stream<Item = Result<O, OpenAIError>> + Send>>
    where
        ...
    {
        let event_source = self
            .streaming_http_client
            ...
            .eventsource()
            .unwrap();

        stream(event_source).await
    }
}

...
```

We retain the `reqwest::Client` for streaming requests because `reqwest_middleware` does not support streaming requests.

We edit `forks/async-openai/async-openai/src/error.rs` to add a new error type from the new middleware crate:

```rs
...

#[derive(Debug, thiserror::Error)]
pub enum OpenAIError {
    /// Underlying error from reqwest library after an API call was made
    #[error("http error: {0}")]
    Reqwest(#[from] reqwest::Error),
    #[error("http error: {0}")]
    ReqwestMiddleware(#[from] reqwest_middleware::Error),
    ...
}

...
```

We commit, push, and tag this commit for easy future reference from our project:

```bash
$ git tag zamm/v0.0.0
$ git push --tags
```

We edit our own `src-tauri/Cargo.toml` to feature our custom fork of async-openai:

```toml
...

[patch.crates-io]
async-openai = { path = "../forks/async-openai/async-openai" }
```

We finally get things running, and after filtering out forbidden headers from both the request and the response, we end up with `src-tauri/api/sample-call-requests/start-conversation.json`:

```json
{
  "http_interactions": [
    {
      "response": {
        "body": {
          "encoding": null,
          "string": "{\n  \"id\": \"chatcmpl-8fsDA6VQfreZpjatOXPffkmyC6HO6\",\n  \"object\": \"chat.completion\",\n  \"created\": 1704990528,\n  \"model\": \"gpt-3.5-turbo-0613\",\n  \"choices\": [\n    {\n      \"index\": 0,\n      \"message\": {\n        \"role\": \"assistant\",\n        \"content\": \"Hello! I'm ZAMM, a chat program. I'm here to help and provide information. How can I assist you today?\"\n      },\n      \"logprobs\": null,\n      \"finish_reason\": \"stop\"\n    }\n  ],\n  \"usage\": {\n    \"prompt_tokens\": 32,\n    \"completion_tokens\": 28,\n    \"total_tokens\": 60\n  },\n  \"system_fingerprint\": null\n}\n"
        },
        "http_version": "1.1",
        "status": {
          "code": 200,
          "message": "OK"
        },
        "headers": {
          "x-request-id": [
            "f77666796e1630fbf7197f9ff5c960ab"
          ],
          "x-ratelimit-limit-tokens": [
            "60000"
          ],
          "content-type": [
            "application/json"
          ],
          "openai-processing-ms": [
            "1326"
          ],
          "access-control-allow-origin": [
            "*"
          ],
          "openai-version": [
            "2020-10-01"
          ],
          "cf-ray": [
            "843e88f25dc4ef20-PDX"
          ],
          "x-ratelimit-reset-tokens": [
            "37ms"
          ],
          "openai-model": [
            "gpt-3.5-turbo-0613"
          ],
          "x-ratelimit-reset-requests": [
            "8.64s"
          ],
          "connection": [
            "keep-alive"
          ],
          "x-ratelimit-limit-requests": [
            "10000"
          ],
          "cf-cache-status": [
            "DYNAMIC"
          ],
          "alt-svc": [
            "h3=\":443\"; ma=86400"
          ],
          "x-ratelimit-remaining-tokens": [
            "59963"
          ],
          "date": [
            "Thu, 11 Jan 2024 16:28:49 GMT"
          ],
          "x-ratelimit-remaining-requests": [
            "9999"
          ],
          "strict-transport-security": [
            "max-age=15724800; includeSubDomains"
          ],
          "cache-control": [
            "no-cache, must-revalidate"
          ],
          "server": [
            "cloudflare"
          ],
          "content-length": [
            "552"
          ]
        }
      },
      "request": {
        "uri": "https://api.openai.com/v1/chat/completions",
        "body": {
          "encoding": null,
          "string": "{\"messages\":[{\"content\":\"You are ZAMM, a chat program. Respond in first person.\",\"role\":\"system\"},{\"content\":\"Hello, does this work?\",\"role\":\"user\"}],\"model\":\"gpt-3.5-turbo\"}"
        },
        "method": "post",
        "headers": {
          "openai-beta": [
            "assistants=v1"
          ],
          "content-type": [
            "application/json"
          ]
        }
      },
      "recorded_at": "Thu, 11 Jan 2024 16:28:49 +0000"
    }
  ],
  "recorded_with": "rVCR 0.1.5"
}
```

The test fails with the output

```
CreateChatCompletionResponse {
    id: "chatcmpl-8fsDA6VQfreZpjatOXPffkmyC6HO6",
    choices: [
        ChatChoice {
            index: 0,
            message: ChatCompletionResponseMessage {
                content: Some(
                    "Hello! I'm ZAMM, a chat program. I'm here to help and provide information. How can I assist you today?",
                ),
                tool_calls: None,
                role: Assistant,
                function_call: None,
            },
            finish_reason: Some(
                Stop,
            ),
            logprobs: None,
        },
    ],
    created: 1704990528,
    model: "gpt-3.5-turbo-0613",
    system_fingerprint: None,
    object: "chat.completion",
    usage: Some(
        CompletionUsage {
            prompt_tokens: 32,
            completion_tokens: 28,
            total_tokens: 60,
        },
    ),
}
thread 'commands::llms::chat::tests::test_start_conversation' panicked at 'assertion failed: !result.is_ok()', src/commands/llms/chat.rs:119:9
```

Now that we can see clearly what data we're working with, we should create the database models for it. We try to commit some of our work first, but see this message:

```bash
$ gi add forks/                                                    
warning: adding embedded git repository: forks/async-openai
hint: You've added another git repository inside your current repository.
hint: Clones of the outer repository will not contain the contents of
hint: the embedded repository and will not know how to obtain it.
hint: If you meant to add a submodule, use:
hint: 
hint:   git submodule add <url> forks/async-openai
hint: 
hint: If you added this path by mistake, you can remove it from the
hint: index with:
hint: 
hint:   git rm --cached forks/async-openai
hint: 
hint: See "git help submodule" for more information.
```

We follow the instructions:

```bash
$ git rm --cached forks/async-openai
error: the following file has staged content different from both the
file and the HEAD:
    forks/async-openai
(use -f to force removal)
$ git rm -f --cached forks/async-openai
rm 'forks/async-openai'
$ git submodule add git@github.com:amosjyng/async-openai.git forks/async-openai
Adding existing repo at 'forks/async-openai' to the index
```

To avoid the unused warnings, we pipe our new command all the way through to `src-tauri/src/main.rs`:

```rs
...
use commands::{
    chat, ...
};

...

fn main() {
    #[cfg(debug_assertions)]
    ts::export(
        collect_types![
            ...,
            chat
        ],
        "../src-svelte/src/lib/bindings.ts",
    )
    .unwrap();

    ...

    tauri::Builder::default()
        ...
        .invoke_handler(tauri::generate_handler![
            ...,
            chat
        ])
        ...;
}
```

We are faced with a compilation error next:

```
    Checking zamm v0.0.0 (/root/zamm/src-tauri)
error: future cannot be sent between threads safely
   --> src/commands/llms/chat.rs:46:1
    |
46  |   #[tauri::command(async)]
    |   ^^^^^^^^^^^^^^^^^^^^^^^^ future returned by `chat` is not `Send`
    |
   ::: src/main.rs:56:25
    |
56  |           .invoke_handler(tauri::generate_handler![
    |  _________________________-
57  | |             greet,
58  | |             get_api_keys,
59  | |             set_api_key,
...   |
64  | |             chat
65  | |         ])
    | |_________- in this macro invocation
    |
    = help: within `impl futures::Future<Output = std::result::Result<(), commands::errors::Error>>`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, setup::api_keys::ApiKeys>`
note: future is not `Send` as this value is used across an await
   --> src/commands/llms/chat.rs:41:57
    |
17  |     let api_keys = zamm_api_keys.0.lock()?;
    |         -------- has type `std::sync::MutexGuard<'_, setup::api_keys::ApiKeys>` which is not `Send`
...
41  |     let response = openai_client.chat().create(request).await?;
    |                                                         ^^^^^ await occurs here, with `api_keys` maybe used later
...
44  | }
    | - `api_keys` is later dropped here
note: required by a bound in `tauri::command::private::ResultFutureTag::future`
   --> /root/.asdf/installs/rust/1.71.1/registry/src/index.crates.io-6f17d22bba15001f/tauri-1.5.4/src/command.rs:293:42
    |
289 |     pub fn future<T, E, F>(self, value: F) -> impl Future<Output = Result<Value, InvokeError>>
    |            ------ required by a bound in this associated function
...
293 |       F: Future<Output = Result<T, E>> + Send,
    |                                          ^^^^ required by this bound in `ResultFutureTag::future`
    = note: this error originates in the macro `__cmd__chat` which comes from the expansion of the macro `tauri::generate_handler` (in Nightly builds, run with -Z macro-backtrace for more info)

```

We see from [this answer](https://stackoverflow.com/a/67277503) that it appears we may need to use a different mutex after all with a different executor. We try to use `tokio::sync::Mutex` instead of `std::sync::Mutex`, and start by editing `src-tauri/Cargo.toml` to move `tokio` from the dev dependencies section into the regular dependencies:

```toml
...

[dependencies]
...
tokio = { version = "1.35.1", features = ["macros"] }

...
```

We use the new mutex in `src-tauri/src/main.rs`; the name is the same, so we only need to change the import:

```rs
...
use tokio::sync::Mutex;
...
```

We edit `src-tauri/src/commands/llms/chat.rs` to use `await` instead of `lock()?`:

```rs
...

async fn chat_helper(
    zamm_api_keys: &ZammApiKeys,
    ...
) -> ZammResult<()> {
    let api_keys = zamm_api_keys.0.lock().await;
    ...
}

...

#[cfg(test)]
mod tests {
    ...
    use tokio::sync::Mutex;

    ...
}
```

We propagate these changes to the rest of the project. As we see from [this piece](https://tauri.app/v1/guides/features/command/#async-commands) of Tauri documentation, we need to make some changes such as using a result. As such, some files such as `src-tauri/src/commands/keys/get.rs` will see greater changes:

```rs
use crate::commands::errors::ZammResult;
...

async fn get_api_keys_helper(zamm_api_keys: &ZammApiKeys) -> ApiKeys {
    zamm_api_keys.0.lock().await.clone()
}

#[tauri::command(async)]
#[specta]
pub async fn get_api_keys(api_keys: State<'_, ZammApiKeys>) -> ZammResult<ApiKeys> {
    Ok(get_api_keys_helper(&api_keys).await)
}

#[cfg(test)]
pub mod tests {
    ...
    use tokio::sync::Mutex;

    pub async fn check_get_api_keys_sample(file_prefix: &str, rust_input: &ZammApiKeys) {
        ...

        let actual_result = get_api_keys_helper(rust_input).await;
        ...
    }

    #[tokio::test]
    async fn test_get_empty_keys() {
        ...

        check_get_api_keys_sample(
            ...
        ).await;
    }

    #[tokio::test]
    async fn test_get_openai_key() {
        ...

        check_get_api_keys_sample(
            ...
        ).await;
    }
}
```

Now we encounter various errors with the `async-openai` crate we forked:

```
warning: unused import: `crate::types::ChatCompletionFunctions`
 --> /root/zamm/forks/async-openai/async-openai/src/types/assistant_impls.rs:1:5
  |
1 | use crate::types::ChatCompletionFunctions;
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default
...
```

From [this question](https://stackoverflow.com/a/63593012), which links to various issues such as [this one](https://github.com/rust-lang/cargo/issues/8546) that's still open, this is an outstanding problem with cargo that we'll have to work around. We edit `src-tauri/clippy.sh` to only consider the bit after the `Checking zamm v0.0.0 (/root/zamm/src-tauri)` part of the output:

```sh
...
zamm_output=$(echo "$clippy_output" | awk '/Checking zamm /{flag=1; next} flag')

if [[ $zamm_output == *"warning"* ]]; then
  exit 1;
fi

```

Now our tests are failing, perhaps because `rvcr` is not applying the transformations we defined to the recording. We try to fork that library as well, cloning our own version of it rather than the original repo.

```bash
$ git submodule add git@github.com:ChorusOne/rvcr.git forks/rvcr
$ <reset repo...>
$ git submodule add git@github.com:amosjyng/rvcr.git forks/rvcr 
fatal: A git directory for 'forks/rvcr' is found locally with remote(s):
  origin        git@github.com:ChorusOne/rvcr.git
If you want to reuse this local git directory instead of cloning again from
  git@github.com:amosjyng/rvcr.git
use the '--force' option. If the local git directory is not the correct repo
or you are unsure what this means choose another name with the '--name' option.
$ git submodule add --force git@github.com:amosjyng/rvcr.git forks/rvcr
$ git checkout -b zamm-fixes
```

We then edit `forks/rvcr/src/lib.rs` to provide a better debugging experience:

```rs
...

impl VCRMiddleware {
    ...

    fn header_values_to_string(&self, header_values: Option<&Vec<String>>) -> String {
        match header_values {
            Some(values) => values.join(", "),
            None => "<MISSING>".to_string(),
        }
    }

    fn find_response_in_vcr(&self, req: vcr_cassette::Request) -> Option<vcr_cassette::Response> {
        ...
        // save diff in a string for debugging purposes
        let mut diff = String::new();
        for interaction in iteractions {
            if interaction.request == req {
                return Some(interaction.response.clone());
            } else {
                diff.push_str(&format!(
                    "Unmatched {method:?} to {uri}:\n",
                    method = interaction.request.method,
                    uri = interaction.request.uri.as_str()
                ));
                if interaction.request.method != req.method {
                    diff.push_str(&format!(
                        "  Method differs: recorded {expected:?}, got {got:?}\n",
                        expected = interaction.request.method,
                        got = req.method
                    ));
                }
                if interaction.request.uri != req.uri {
                    diff.push_str("  URI differs:\n");
                    diff.push_str(&format!(
                        "    recorded: \"{}\"\n",
                        interaction.request.uri.as_str()
                    ));
                    diff.push_str(&format!("    got:      \"{}\"\n", req.uri.as_str()));
                }
                if interaction.request.headers != req.headers {
                    diff.push_str("  Headers differ:\n");
                    for (recorded_header_name, recorded_header_values) in
                        &interaction.request.headers
                    {
                        let expected = self.header_values_to_string(Some(recorded_header_values));
                        let got =
                            self.header_values_to_string(req.headers.get(recorded_header_name));
                        if expected != got {
                            diff.push_str(&format!("    {}:\n", recorded_header_name));
                            diff.push_str(&format!("      recorded: \"{}\"\n", expected));
                            diff.push_str(&format!("      got:      \"{}\"\n", got));
                        }
                    }
                    for (got_header_name, got_header_values) in &req.headers {
                        if !interaction.request.headers.contains_key(got_header_name) {
                            let got = self.header_values_to_string(Some(got_header_values));
                            diff.push_str(&format!("    {}:\n", got_header_name));
                            diff.push_str(&format!("      recorded: <MISSING>\n"));
                            diff.push_str(&format!("      got:      \"{}\"\n", got));
                        }
                    }
                }
                if interaction.request.body != req.body {
                    diff.push_str("  Body differs:\n");
                    diff.push_str(&format!(
                        "    recorded: \"{}\"\n",
                        interaction.request.body.string
                    ));
                    diff.push_str(&format!("    got:      \"{}\"\n", req.body.string));
                }
            }
        }
        eprintln!("{}", diff);
        None
    }

    ...
}

#[async_trait::async_trait]
impl Middleware for VCRMiddleware {
    async fn handle(
        &self,
        ...
    ) -> reqwest_middleware::Result<reqwest::Response> {
        ...

        match self.mode {
            VCRMode::Record => {
                ...
            }
            VCRMode::Replay => {
                let vcr_response = self.find_response_in_vcr(vcr_request).unwrap_or_else(|| {
                    panic!(
                        "Cannot find corresponding request in cassette {:?}",
                        self.path
                    )
                });
                ...
            }
        }
    }
}

...
```

With this, we are able to see that the problem is indeed that the request being compared against does not have the header stripped out:

```
Unmatched Post to https://api.openai.com/v1/chat/completions:
  Headers differ:
    authorization:
      recorded: <MISSING>
      got:      "Bearer sk-..."
```

As such, we go and fix our logic in `src-tauri/src/commands/llms/chat.rs` to make sure that the same middleware setup is always used, just with different recording modes:

```rs
    #[tokio::test]
    async fn test_start_conversation() {
        ...
        let vcr_mode = if is_recording {
            VCRMode::Record
        } else {
            VCRMode::Replay
        };
        let middleware = VCRMiddleware::try_from(recording_path)
                .unwrap()
                .with_mode(vcr_mode)
                ...;
        ...
    }
```

Now we decide that we want to say `<CENSORED>` instead of simply leaving the headers out, so that it's still clear what's actually being sent or received. We do some refactoring and work directly with the headers hashmap:

```bash
$ cargo add --dev vcr-cassette
```

and edit `src-tauri/src/commands/llms/chat.rs` while deciding that it's okay to set cookies after all as they are not login cookies:

```rs
...

#[cfg(test)]
mod tests {
    ...
    use vcr_cassette::Headers;

    const CENSORED: &str = "<CENSORED>";

    fn censor_headers(headers: &Headers, blacklisted_keys: &[&str]) -> Headers {
        return headers.clone()
            .iter()
            .map(|(k, v)| {
                if blacklisted_keys.contains(&k.as_str()) {
                    (k.clone(), vec![CENSORED.to_string()])
                } else {
                    (k.clone(), v.clone())
                }
            })
            .collect()
    }

    #[tokio::test]
    async fn test_start_conversation() {
        ...
        let middleware = VCRMiddleware::try_from(recording_path)
            ...
            .with_modify_request(|req| {
                req.headers = censor_headers(&req.headers, &["authorization"]);
            })
            .with_modify_response(|resp| {
                resp.headers = censor_headers(&resp.headers, &["openai-organization"]);
            });
        ...
    }
}
```

#### Database persistence

Now we create new database tables to store our chat API calls in:

```bash
$ diesel migration generate create_llm_calls
Creating migrations/2024-01-13-023144_create_llm_calls/up.sql
Creating migrations/2024-01-13-023144_create_llm_calls/down.sql
```

We edit `src-tauri/migrations/2024-01-13-023144_create_llm_calls/up.sql`:

```sql
CREATE TABLE llm_calls (
  id VARCHAR PRIMARY KEY NOT NULL,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
  provider VARCHAR NOT NULL,
  llm_requested VARCHAR NOT NULL,
  llm VARCHAR NOT NULL,
  temperature REAL NOT NULL,
  prompt_tokens INTEGER,
  response_tokens INTEGER,
  prompt TEXT NOT NULL,
  completion TEXT NOT NULL
)

```

and `src-tauri/migrations/2024-01-13-023144_create_llm_calls/down.sql`:

```sql
DROP TABLE llm_calls

```

and run

```bash
$ diesel migration run --database-url /root/.local/share/zamm/zamm.sqlite3
Running migration 2024-01-13-023144_create_llm_calls
```

We can see that `src-tauri/src/schema.rs` gets updated automatically:

```rs
...

diesel::table! {
    llm_calls (id) {
        id -> Text,
        timestamp -> Timestamp,
        provider -> Text,
        llm_requested -> Text,
        llm -> Text,
        temperature -> Float,
        prompt_tokens -> Nullable<Integer>,
        response_tokens -> Nullable<Integer>,
        prompt -> Text,
        completion -> Text,
    }
}

diesel::allow_tables_to_appear_in_same_query!(..., llm_calls,);

```

Next we have to define the models, but we prepare for this by first refactoring our model module to allow for multiple submodules. We rename `src-tauri/src/models.rs` to `src-tauri/src/models/api_keys.rs` and create `src-tauri/src/models/mod.rs`:

```rs
mod api_keys;

pub use api_keys::{ApiKey, NewApiKey};

```

We commit the specific above changes, and then continue. We initially create a wrapper around the OpenAI types for SQL expressions, because we can't directly implement that for types we don't own:

```rs
#[derive(AsExpression, FromSqlRow, Debug, Clone, Serialize, Deserialize, PartialEq)]
#[diesel(sql_type = Text)]
pub struct OpenAiPrompt {
    #[serde(flatten)]
    pub prompt: Vec<ChatCompletionRequestMessage>,
}

impl Deref for OpenAiPrompt {
    type Target = Vec<ChatCompletionRequestMessage>;

    fn deref(&self) -> &Self::Target {
        &self.prompt
    }
}

impl ToSql<Text, Sqlite> for OpenAiPrompt
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let json_str = serde_json::to_string(&self.prompt)?;
        out.set_value(json_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for OpenAiPrompt
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let json_str = String::from_sql(bytes)?;
        let parsed_json: Vec<ChatCompletionRequestMessage> = serde_json::from_str(&json_str)?;
        Ok(OpenAiPrompt { prompt: parsed_json })
    }
}

#[derive(AsExpression, FromSqlRow, Debug, Clone, Serialize, Deserialize, PartialEq)]
#[diesel(sql_type = Text)]
pub struct OpenAiCompletion {
    #[serde(flatten)]
    pub completion: ChatCompletionResponseMessage,
}

impl Deref for OpenAiCompletion {
    type Target = ChatCompletionResponseMessage;

    fn deref(&self) -> &Self::Target {
        &self.completion
    }
}

impl ToSql<Text, Sqlite> for OpenAiCompletion
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let json_str = serde_json::to_string(&self.completion)?;
        out.set_value(json_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for OpenAiCompletion
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let json_str = String::from_sql(bytes)?;
        let parsed_json: ChatCompletionResponseMessage = serde_json::from_str(&json_str)?;
        Ok(OpenAiCompletion { completion: parsed_json })
    }
}
```

However, because `async_openai` disables Serde tagging, the tests come up with the wrong type when results are deserialized from JSON. Additionally, given that the database is the one thing that will have implications for backwards compatibility, it seems best for us to retain full control over the format we use. Finally, using more generic types will make it easier to add in other LLMs from other providers.

As such, we create our own types in `src-tauri/src/models/llm_calls.rs`, along with convenience conversion functions:

```rs
use crate::commands::Error;
use crate::schema::llm_calls;
use crate::setup::api_keys::Service;
use async_openai::types::{
    ChatCompletionRequestMessage, ChatCompletionRequestUserMessageContent,
    ChatCompletionResponseMessage, Role,
};
use chrono::naive::NaiveDateTime;
use diesel::backend::Backend;
use diesel::deserialize::FromSqlRow;
use diesel::deserialize::{self, FromSql};
use diesel::expression::AsExpression;
use diesel::prelude::*;
use diesel::serialize::{self, IsNull, Output, ToSql};
use diesel::sql_types::Text;
use diesel::sqlite::Sqlite;
use serde::{Deserialize, Serialize};
use serde_json;
use std::ops::Deref;
use uuid::Uuid;

#[derive(
    Debug,
    Clone,
    Serialize,
    Deserialize,
    PartialEq,
    AsExpression,
    FromSqlRow,
    specta::Type,
)]
#[diesel(sql_type = Text)]
#[serde(tag = "role")]
pub enum ChatMessage {
    System { text: String },
    Human { text: String },
    AI { text: String },
}

impl TryFrom<ChatCompletionRequestMessage> for ChatMessage {
    type Error = Error;

    fn try_from(message: ChatCompletionRequestMessage) -> Result<Self, Self::Error> {
        match message {
            ChatCompletionRequestMessage::System(system_message) => {
                Ok(ChatMessage::System {
                    text: system_message.content,
                })
            }
            ChatCompletionRequestMessage::User(user_message) => {
                match user_message.content {
                    ChatCompletionRequestUserMessageContent::Text(text) => {
                        Ok(ChatMessage::Human { text })
                    }
                    ChatCompletionRequestUserMessageContent::Array(_) => {
                        Err(Error::UnexpectedOpenAiResponse {
                            reason: "Image chat not supported yet".to_string(),
                        })
                    }
                }
            }
            ChatCompletionRequestMessage::Assistant(assistant_message) => {
                match assistant_message.content {
                    Some(content) => Ok(ChatMessage::AI { text: content }),
                    None => Err(Error::UnexpectedOpenAiResponse {
                        reason: "AI function calls not supported yet".to_string(),
                    }),
                }
            }
            _ => Err(Error::UnexpectedOpenAiResponse {
                reason: "Only AI text chat is supported".to_string(),
            }),
        }
    }
}

impl TryFrom<ChatCompletionResponseMessage> for ChatMessage {
    type Error = Error;

    fn try_from(message: ChatCompletionResponseMessage) -> Result<Self, Self::Error> {
        let text = message.content.ok_or(Error::UnexpectedOpenAiResponse {
            reason: "No content in response".to_string(),
        })?;
        match message.role {
            Role::System => Ok(ChatMessage::System { text }),
            Role::User => Ok(ChatMessage::Human { text }),
            Role::Assistant => Ok(ChatMessage::AI { text }),
            _ => Err(Error::UnexpectedOpenAiResponse {
                reason: "Only AI text chat is supported".to_string(),
            }),
        }
    }
}

impl ToSql<Text, Sqlite> for ChatMessage
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let json_str = serde_json::to_string(&self)?;
        out.set_value(json_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for ChatMessage
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let json_str = String::from_sql(bytes)?;
        let parsed_json: Self = serde_json::from_str(&json_str)?;
        Ok(parsed_json)
    }
}

#[derive(
    Debug,
    Clone,
    Serialize,
    Deserialize,
    PartialEq,
    AsExpression,
    FromSqlRow,
    specta::Type,
)]
#[diesel(sql_type = Text)]
pub struct ChatPrompt {
    messages: Vec<ChatMessage>,
}

impl Deref for ChatPrompt {
    type Target = Vec<ChatMessage>;

    fn deref(&self) -> &Self::Target {
        &self.messages
    }
}

impl TryFrom<Vec<ChatCompletionRequestMessage>> for ChatPrompt {
    type Error = Error;

    fn try_from(
        messages: Vec<ChatCompletionRequestMessage>,
    ) -> Result<Self, Self::Error> {
        let messages = messages
            .into_iter()
            .map(|message| message.try_into())
            .collect::<Result<Vec<ChatMessage>, Self::Error>>()?;
        Ok(ChatPrompt { messages })
    }
}

impl ToSql<Text, Sqlite> for ChatPrompt
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let json_str = serde_json::to_string(&self)?;
        out.set_value(json_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for ChatPrompt
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let json_str = String::from_sql(bytes)?;
        let parsed_json: Self = serde_json::from_str(&json_str)?;
        Ok(parsed_json)
    }
}
```

As shown in [`diesel.md`](/general-notes/libraries/rust/diesel.md), we use `EntityId` as a wrapper around UUID, except that we name the `uuid` field `id` for JSON serialization and implement `Deref` for it because we want to pretend as if the wrapper didn't actually exist:

```rs
#[derive(
    AsExpression, FromSqlRow, Debug, Serialize, Deserialize, Clone, specta::Type,
)]
#[diesel(sql_type = Text)]
pub struct EntityId {
    #[serde(rename = "id")]
    pub uuid: Uuid,
}

impl Deref for EntityId {
    type Target = Uuid;

    fn deref(&self) -> &Self::Target {
        &self.uuid
    }
}

impl ToSql<Text, Sqlite> for EntityId
where
    String: ToSql<Text, Sqlite>,
{
    fn to_sql<'b>(&'b self, out: &mut Output<'b, '_, Sqlite>) -> serialize::Result {
        let uuid_str = self.uuid.to_string();
        out.set_value(uuid_str);
        Ok(IsNull::No)
    }
}

impl<DB> FromSql<Text, DB> for EntityId
where
    DB: Backend,
    String: FromSql<Text, DB>,
{
    fn from_sql(bytes: DB::RawValue<'_>) -> deserialize::Result<Self> {
        let uuid_str = String::from_sql(bytes)?;
        let parsed_uuid = Uuid::parse_str(&uuid_str)?;
        Ok(EntityId { uuid: parsed_uuid })
    }
}
```

We define the rest of the groupings and the data structure we want to be represented in the database:

```rs
#[derive(Debug, Serialize, Deserialize, Clone, specta::Type)]
pub struct Llm {
    pub name: String,
    pub requested: String,
    pub provider: Service,
}

#[derive(Debug, Serialize, Deserialize, specta::Type)]
pub struct Request {
    pub prompt: ChatPrompt,
    pub temperature: f32,
}

#[derive(Debug, Serialize, Deserialize, specta::Type)]
pub struct Response {
    pub completion: ChatMessage,
}

#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct TokenMetadata {
    pub prompt_tokens: Option<i32>,
    pub response_tokens: Option<i32>,
}

#[derive(Debug, Serialize, Deserialize, specta::Type)]
pub struct LlmCall {
    #[serde(flatten)]
    pub id: EntityId,
    pub timestamp: NaiveDateTime,
    pub llm: Llm,
    pub request: Request,
    pub response: Response,
    pub token_metadata: TokenMetadata,
}
```

It is not clear how to represent this as a Disel row, so we create separate data structures for Diesel instead:

```rs
#[derive(Debug, Queryable, Selectable, Clone)]
#[diesel(table_name = llm_calls)]
pub struct LlmCallRow {
    pub id: EntityId,
    pub timestamp: NaiveDateTime,
    pub provider: Service,
    pub llm_requested: String,
    pub llm: String,
    pub temperature: f32,
    pub prompt_tokens: Option<i32>,
    pub response_tokens: Option<i32>,
    pub prompt: ChatPrompt,
    pub completion: ChatMessage,
}

#[derive(Insertable)]
#[diesel(table_name = llm_calls)]
pub struct NewLlmCallRow<'a> {
    pub id: &'a EntityId,
    pub timestamp: &'a NaiveDateTime,
    pub provider: &'a Service,
    pub llm_requested: &'a str,
    pub llm: &'a str,
    pub temperature: &'a f32,
    pub prompt_tokens: Option<&'a i32>,
    pub response_tokens: Option<&'a i32>,
    pub prompt: &'a ChatPrompt,
    pub completion: &'a ChatMessage,
}
```

and add ways to convert between the two:

```rs
impl LlmCall {
    pub fn as_sql_row(&self) -> NewLlmCallRow {
        NewLlmCallRow {
            id: &self.id,
            timestamp: &self.timestamp,
            provider: &self.llm.provider,
            llm_requested: &self.llm.requested,
            llm: &self.llm.name,
            temperature: &self.request.temperature,
            prompt_tokens: self.token_metadata.prompt_tokens.as_ref(),
            response_tokens: self.token_metadata.response_tokens.as_ref(),
            prompt: &self.request.prompt,
            completion: &self.response.completion,
        }
    }
}

impl From<LlmCallRow> for LlmCall {
    fn from(row: LlmCallRow) -> Self {
        let id = row.id;
        let timestamp = row.timestamp;
        let llm = Llm {
            name: row.llm,
            requested: row.llm_requested,
            provider: row.provider,
        };
        let request = Request {
            prompt: row.prompt,
            temperature: row.temperature,
        };
        let response = Response {
            completion: row.completion,
        };
        let token_metadata = TokenMetadata {
            prompt_tokens: row.prompt_tokens,
            response_tokens: row.response_tokens,
        };
        LlmCall {
            id,
            timestamp,
            llm,
            request,
            response,
            token_metadata,
        }
    }
}

```

Note that we have to make use of a number of new dependencies or new features to existing dependencies in `src-tauri/Cargo.toml`:

```toml
...

[dependencies]
...
diesel = { ..., features = [..., "chrono"] }
...
uuid = { ..., features = [..., "serde"] }
specta = { ..., features = ["uuid", "chrono"] }
...
chrono = "0.4.31"

...
```

Note that we also need to add this new error in `src-tauri/src/commands/errors.rs`:

```rs
#[derive(thiserror::Error, Debug)]
pub enum Error {
    ...
    #[error("Unexpected JSON: {reason}")]
    UnexpectedOpenAiResponse { reason: String },
    ...
}
```

We export the new modules in `src-tauri/src/models/mod.rs`:

```rs
pub mod api_keys;
pub mod llm_calls;

...
```

We want to save the chat request afterwards, so we edit our fork of `forks/async-openai/async-openai/src/chat.rs` to make sure that the request is only borrowed and not moved into the function call, so that it is still available for use afterwards:

```rs
impl<'c, C: Config> Chat<'c, C> {
    ...

    /// Creates a model response for the given chat conversation.
    pub async fn create(
        &self,
        request: &CreateChatCompletionRequest,
    ) -> ... {
        ...
    }

    ...
}
```

We commit, cherry-pick and create the PR [here](https://github.com/64bit/async-openai/pull/181) in case the upstream maintainers are interested. We go back to the parent directory and edit `src-tauri/src/commands/llms/chat.rs` to save the chat interaction to the database:

```rs
...
use crate::models::llm_calls::{
    EntityId, Llm, LlmCall, Request, Response, TokenMetadata,
};
use crate::schema::llm_calls;
...
use crate::{ZammApiKeys, ZammDatabase};
...
use diesel::RunQueryDsl;
...
use uuid::Uuid;

async fn chat_helper(
    ...,
    zamm_db: &ZammDatabase,
    ...
) -> ZammResult<LlmCall> {
    ...

    let db = &mut zamm_db.0.lock().await;

    let config = OpenAIConfig::new().with_api_key(api_keys.openai.as_ref().unwrap());
    let requested_model = "gpt-3.5-turbo";
    let temperature = 1.0;

    ...
    let request = CreateChatCompletionRequestArgs::default()
        .model(requested_model)
        .temperature(temperature)
        ...;
    let response = openai_client.chat().create(&request).await?;

    let token_metadata = TokenMetadata {
        prompt_tokens: response
            .usage
            .as_ref()
            .map(|usage| usage.prompt_tokens as i32),
        response_tokens: response
            .usage
            .as_ref()
            .map(|usage| usage.completion_tokens as i32),
    };
    let sole_choice = response
        .choices
        .first()
        .ok_or(Error::UnexpectedOpenAiResponse {
            reason: "Zero choices".to_owned(),
        })?
        .message
        .to_owned();
    let llm_call = LlmCall {
        id: EntityId {
            uuid: Uuid::new_v4(),
        },
        timestamp: chrono::Utc::now().naive_utc(),
        llm: Llm {
            provider: Service::OpenAI,
            name: response.model.clone(),
            requested: requested_model.to_owned(),
        },
        request: Request {
            temperature,
            prompt: request.messages.try_into()?,
        },
        response: Response {
            completion: sole_choice.try_into()?,
        },
        token_metadata,
    };

    if let Some(conn) = db.as_mut() {
        diesel::insert_into(llm_calls::table)
            .values(llm_call.as_sql_row())
            .execute(conn)?;
    } // todo: warn users if DB write unsuccessful

    Ok(llm_call)
}

#[tauri::command(async)]
#[specta]
pub async fn chat(
    api_keys: State<'_, ZammApiKeys>,
    database: State<'_, ZammDatabase>,
) -> ZammResult<LlmCall> {
    ...
    chat_helper(&api_keys, &database, client_with_middleware).await
}

#[cfg(test)]
mod tests {
    ...
    use crate::models::llm_calls::{ChatMessage, LlmCallRow};
    ...
    use crate::setup::db::MIGRATIONS;
    use diesel::prelude::*;
    use diesel_migrations::MigrationHarness;
    ...

    fn setup_database() -> SqliteConnection {
        let mut conn = SqliteConnection::establish(":memory:").unwrap();
        conn.run_pending_migrations(MIGRATIONS).unwrap();
        conn
    }

    pub fn setup_zamm_db() -> ZammDatabase {
        ZammDatabase(Mutex::new(Some(setup_database())))
    }

    async fn get_llm_call(db: &ZammDatabase, call_id: &EntityId) -> LlmCall {
        use crate::schema::llm_calls::dsl::*;
        let mut conn_mutex = db.0.lock().await;
        let conn = conn_mutex.as_mut().unwrap();
        llm_calls
            .filter(id.eq(call_id))
            .first::<LlmCallRow>(conn)
            .unwrap()
            .into()
    }

    #[tokio::test]
    async fn test_start_conversation() {
        ...

        let db = setup_zamm_db();
        let result = chat_helper(&api_keys, &db, vcr_client).await;
        assert!(result.is_ok(), "Error: {:?}", result.err());
        let ok_result = result.unwrap();
        assert!(ok_result.request.prompt.len() > 0);
        match &ok_result.response.completion {
            ChatMessage::AI { text } => assert!(!text.is_empty()),
            _ => panic!("Unexpected response type"),
        }

        // check that it made it into the database
        let stored_llm_call = get_llm_call(&db, &ok_result.id).await;
        assert_eq!(stored_llm_call.request.prompt, ok_result.request.prompt);
        assert_eq!(
            stored_llm_call.response.completion,
            ok_result.response.completion
        );
    }
}

```

We re-record `src-tauri/api/sample-call-requests/start-conversation.json` to use the new temperature specification.

Now that the tests pass, we can be reasonably confident that database persistence is working.

#### API boundary

Now we can iterate on creating the API boundary. We create `src-tauri/api/sample-calls/chat-start-conversation.yaml` to define how we want the API to look:

```yaml
request:
  - chat
  - >
    {
      "provider": "OpenAI",
      "llm": "gpt-3.5-turbo",
      "temperature": null,
      "prompt": {
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
      }
    }
response:
  message: >
    {
      "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
      "timestamp": "2024-01-16T08:50:19.738093890",
      "llm": {
        "name": "gpt-3.5-turbo-0613",
        "requested": "gpt-3.5-turbo",
        "provider": "OpenAI"
      },
      "request": {
        "prompt": {
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
          "text": "Hello! I'm ZAMM, a chat program. I'm here to assist you. What can I help you with today?"
        }
      },
      "token_metadata": {
        "prompt_tokens": 32,
        "response_tokens": 27
      }
    }

```

The `request` part in the response is technically redundant as it doesn't need to refer to anything, but we don't need to optimize that away right now.

We need to define some additional conversion functions, so we edit `src-tauri/src/models/llm_calls.rs`:

```rs
...
use async_openai::types::{
    ..., ChatCompletionRequestSystemMessage, ChatCompletionRequestUserMessage,
    ChatCompletionRequestAssistantMessage
};
...

impl Into<ChatCompletionRequestMessage> for ChatMessage {
    fn into(self) -> ChatCompletionRequestMessage {
        match self {
            ChatMessage::System { text } => {
                ChatCompletionRequestMessage::System(
                    ChatCompletionRequestSystemMessage {
                        content: text,
                        role: Role::System,
                        ..Default::default()
                    }
                )
            }
            ChatMessage::Human { text } => {
                ChatCompletionRequestMessage::User(ChatCompletionRequestUserMessage {
                    content: ChatCompletionRequestUserMessageContent::Text(
text
                    ),
                    role: Role::User,
                    ..Default::default()
                })
            }
            ChatMessage::AI { text } => {
                ChatCompletionRequestMessage::Assistant(
                    ChatCompletionRequestAssistantMessage {
                        content: Some(text),
                        role: Role::Assistant,
                        ..Default::default()
                    }
                )
            }
        }
    }
}

...

impl Into<Vec<ChatCompletionRequestMessage>> for ChatPrompt {
    fn into(self) -> Vec<ChatCompletionRequestMessage> {
        self.messages
            .into_iter()
            .map(|message| message.into())
            .collect()
    }
}

...

#[derive(..., Clone, ...)]
pub struct Request {
    ...
}

#[derive(..., Clone, ...)]
pub struct Response {
    ...
}

#[derive(..., Clone, ...)]
pub struct LlmCall {
    ...
}
```

Finally, we edit `src-tauri/src/commands/llms/chat.rs` to actually handle a real API call instead of the fake one we've hard-coded into it:

```rs
...
use crate::models::llm_calls::{
    ..., ChatPrompt
};
...
use async_openai::types::{
    CreateChatCompletionRequestArgs, ChatCompletionRequestMessage
};
...

async fn chat_helper(
    ...,
    provider: Service,
    llm: String,
    temperature: Option<f32>,
    prompt: ChatPrompt,
    ...,
) -> ZammResult<LlmCall> {
    ...

    let config = match provider {
        Service::OpenAI => {
            let openai_api_key = api_keys.openai.as_ref().ok_or(Error::MissingApiKey { service: Service::OpenAI })?;
            OpenAIConfig::new().with_api_key(openai_api_key)
        },
    };

    let requested_model = llm;
    let requested_temperature = temperature.unwrap_or(1.0);

    let openai_client =
        async_openai::Client::with_config(config).with_http_client(http_client);
    let messages: Vec<ChatCompletionRequestMessage> = prompt.into();
    let request = CreateChatCompletionRequestArgs::default()
        .model(&requested_model)
        .temperature(requested_temperature)
        .messages(messages)
        .build()?;

    ...

    let llm_call = LlmCall {
        ...
        request: Request {
            temperature: requested_temperature,
            ...
        },
        ...
    }

    ...
}

#[tauri::command(async)]
#[specta]
pub async fn chat(
    ...,
    provider: Service,
    llm: String,
    temperature: Option<f32>,
    prompt: ChatPrompt,
) -> ZammResult<LlmCall> {
    ...
    chat_helper(..., provider, llm, temperature, prompt, ...).await
}

#[cfg(test)]
mod tests {
    ...
    use serde::{Deserialize, Serialize};
    use crate::sample_call::SampleCall;
    use std::fs;
    ...

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct ChatRequest {
        provider: Service,
        llm: String,
        temperature: Option<f32>,
        prompt: ChatPrompt,
    }

    fn parse_request(request_str: &str) -> ChatRequest {
        serde_json::from_str(request_str).unwrap()
    }

    fn parse_response(response_str: &str) -> LlmCall {
        serde_json::from_str(response_str).unwrap()
    }

    fn read_sample(filename: &str) -> SampleCall {
        let sample_str = fs::read_to_string(filename)
            .unwrap_or_else(|_| panic!("No file found at {filename}"));
        serde_yaml::from_str(&sample_str).unwrap()
    }

    #[tokio::test]
    async fn test_start_conversation() {
        ...
        let api_keys = if is_recording {
            ZammApiKeys(Mutex::new(ApiKeys {
                openai: env::var("OPENAI_API_KEY").ok(),
            }))
        } else {
            ZammApiKeys(Mutex::new(ApiKeys {
                openai: Some("dummy".to_string()),
            }))
        };

        ...
        // end dependencies setup

        let sample = read_sample("api/sample-calls/chat-start-conversation.yaml");
        assert_eq!(sample.request.len(), 2);
        assert_eq!(sample.request[0], "chat");

        let request = parse_request(&sample.request[1]);

        let result = chat_helper(&api_keys, &db, request.provider, request.llm, request.temperature, request.prompt, vcr_client).await;
        ...

        // check that the API call returns the expected JSON
        let expected_llm_call = parse_response(&sample.response.message);
        // swap out non-deterministic parts before JSON comparison
        let deterministic_llm_call = LlmCall {
            id: expected_llm_call.id,
            timestamp: expected_llm_call.timestamp,
            ..ok_result.clone()
        };
        let actual_json = serde_json::to_string_pretty(&deterministic_llm_call).unwrap();
        let expected_json = sample.response.message.trim();
        assert_eq!(actual_json, expected_json);

        ...
    }
}

```

We make sure the tests pass now, and commit.

##### Flattening the JSON request/response

Since the nested nature of the `ChatPrompt` struct is just an artifact of the Rust type system, we can flatten it out in the JSON request/response. We can also make the `token_metadata` key less verbose. We edit `src-tauri/api/sample-calls/chat-start-conversation.yaml` to reflect what we want:

```yaml
request:
  - chat
  - >
    {
      ...,
      "prompt": [
        {
          "role": "System",
          "text": "You are ZAMM, a chat program. Respond in first person."
        },
        {
          "role": "Human",
          "text": "Hello, does this work?"
        }
      ]
    }
response:
  message: >
    {
      ...,
      "request": {
        "prompt": [
          {
            "role": "System",
            "text": "You are ZAMM, a chat program. Respond in first person."
          },
          {
            "role": "Human",
            "text": "Hello, does this work?"
          }
        ],
        ...
      },
      ...,
      "tokens": {
        "prompt": 32,
        "response": 27
      }
    }

```

We then edit `src-tauri/src/models/llm_calls.rs`:

```rs
...

#[derive(
    ...
)]
#[diesel(sql_type = Text)]
pub struct ChatPrompt {
    pub prompt: Vec<ChatMessage>,
}

impl Deref for ChatPrompt {
    ...

    fn deref(&self) -> &Self::Target {
        &self.prompt
    }
}

impl TryFrom<Vec<ChatCompletionRequestMessage>> for ChatPrompt {
    type Error = Error;

    fn try_from(
        ...
    ) -> Result<Self, Self::Error> {
        ...
        Ok(ChatPrompt { prompt: messages })
    }
}

impl From<ChatPrompt> for Vec<ChatCompletionRequestMessage> {
    fn from(val: ChatPrompt) -> Self {
        val.prompt
            ...
    }
}

...

#[derive(...)]
pub struct Request {
    #[serde(flatten)]
    pub prompt: ChatPrompt,
    ...
}

...

#[derive(...)]
pub struct TokenMetadata {
    pub prompt: Option<i32>,
    ...
}

...

#[derive(...)]
pub struct LlmCall {
    ...
    pub tokens: TokenMetadata,
}

impl LlmCall {
    pub fn as_sql_row(&self) -> NewLlmCallRow {
        NewLlmCallRow {
            ...
            prompt_tokens: self.tokens.prompt.as_ref(),
            response_tokens: self.tokens.response.as_ref(),
            ...
        }
    }
}

impl From<LlmCallRow> for LlmCall {
    fn from(row: LlmCallRow) -> Self {
        ...
        let token_metadata = TokenMetadata {
            prompt: row.prompt_tokens,
            response: row.response_tokens,
        };
        LlmCall {
            ...,
            tokens: token_metadata,
        }
    }
}
```

and `src-tauri/src/commands/llms/chat.rs`:

```rs
...
use crate::models::llm_calls::{
    ..., ChatMessage, ...
};
...

async fn chat_helper(
    ...,
    prompt: Vec<ChatMessage>,
    ...
) -> ZammResult<LlmCall> {
    ...
    let messages: Vec<ChatCompletionRequestMessage> = prompt.clone().into_iter().map(|m| m.into()).collect();
    ...
    let token_metadata = TokenMetadata {
        prompt: response
            ...,
        response: response
            ...,
    };
    ...
    let llm_call = LlmCall {
        ...
        request: Request {
            ...
            prompt: ChatPrompt { prompt },
        },
        ...
        tokens: token_metadata,
    };
    ...
}

#[tauri::command(async)]
#[specta]
pub async fn chat(
    ...,
    prompt: Vec<ChatMessage>,
) -> ZammResult<LlmCall> {
    ...
}

#[cfg(test)]
mod tests {
    ...

    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    struct ChatRequest {
        ...,
        prompt: Vec<ChatMessage>,
    }

    ...
}
```

##### Fixing the build

While `cargo test` passes, `cargo build` does not. We get errors such as

```
error[E0277]: the trait bound `NaiveDateTime: Deserialize<'_>` is not satisfied
    --> src/models/llm_calls.rs:315:20
     |
315  |     pub timestamp: NaiveDateTime,
     |                    ^^^^^^^^^^^^^ the trait `Deserialize<'_>` is not implemented for `NaiveDateTime`
     |
     = help: the following other types implement trait `Deserialize<'de>`:
               <&'a [u8] as Deserialize<'de>>
               <&'a serde_json::value::RawValue as Deserialize<'de>>
               <&'a std::path::Path as Deserialize<'de>>
               <&'a str as Deserialize<'de>>
               <() as Deserialize<'de>>
               <(T0, T1) as Deserialize<'de>>
               <(T0, T1, T2) as Deserialize<'de>>
               <(T0, T1, T2, T3) as Deserialize<'de>>
             and 444 others
note: required by a bound in `next_value`
    --> /root/.asdf/installs/rust/1.71.1/registry/src/index.crates.io-6f17d22bba15001f/serde-1.0.195/src/de/mod.rs:1865:12
     |
1863 |     fn next_value<V>(&mut self) -> Result<V, Self::Error>
     |        ---------- required by a bound in this associated function
1864 |     where
1865 |         V: Deserialize<'de>,
     |            ^^^^^^^^^^^^^^^^ required by this bound in `MapAccess::next_value`

error[E0277]: the trait bound `NaiveDateTime: Deserialize<'_>` is not satisfied
   --> src/models/llm_calls.rs:315:5
    |
315 |     pub timestamp: NaiveDateTime,
    |     ^^^ the trait `Deserialize<'_>` is not implemented for `NaiveDateTime`
    |
    = help: the following other types implement trait `Deserialize<'de>`:
              <&'a [u8] as Deserialize<'de>>
              <&'a serde_json::value::RawValue as Deserialize<'de>>
              <&'a std::path::Path as Deserialize<'de>>
              <&'a str as Deserialize<'de>>
              <() as Deserialize<'de>>
              <(T0, T1) as Deserialize<'de>>
              <(T0, T1, T2) as Deserialize<'de>>
              <(T0, T1, T2, T3) as Deserialize<'de>>
            and 444 others
note: required by a bound in `python_api::_::_serde::__private::de::missing_field`
   --> /root/.asdf/installs/rust/1.71.1/registry/src/index.crates.io-6f17d22bba15001f/serde-1.0.195/src/private/de.rs:25:8
    |
23  | pub fn missing_field<'de, V, E>(field: &'static str) -> Result<V, E>
    |        ------------- required by a bound in this function
24  | where
25  |     V: Deserialize<'de>,
    |        ^^^^^^^^^^^^^^^^ required by this bound in `missing_field`

For more information about this error, try `rustc --explain E0277`.
```

As we see from [this answer](https://stackoverflow.com/a/65379272), we need to enable the `serde` feature in the `chrono` crate. For some reason this doesn't fail on test builds. We edit `src-tauri/Cargo.toml` accordingly:

```toml
...
chrono = { ..., features = ["serde"] }

...
```

and now our build completes successfully.

#### Storing total tokens

We decide to store the total tokens in the response as well, for convenience querying the database. We edit `src-tauri/migrations/2024-01-13-023144_create_llm_calls/up.sql` directly since we haven't done a commit yet, so there's no need for a new migration:

```sql
CREATE TABLE llm_calls (
  ...
  total_tokens INTEGER,
  ...
)

```

Redoing the diesel migration updates `src-tauri/src/schema.rs` for us. We update `src-tauri/src/models/llm_calls.rs` to match:

```rs
#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct TokenMetadata {
    ...,
    pub total: Option<i32>,
}

#[derive(Debug, Queryable, Selectable, Clone)]
#[diesel(table_name = llm_calls)]
pub struct LlmCallRow {
    ...,
    pub total_tokens: Option<i32>,
    ...
}

#[derive(Insertable)]
#[diesel(table_name = llm_calls)]
pub struct NewLlmCallRow<'a> {
    ...,
    pub total_tokens: Option<&'a i32>,
    ...
}

...

impl LlmCall {
    pub fn as_sql_row(&self) -> NewLlmCallRow {
        NewLlmCallRow {
            ...,
            total_tokens: self.tokens.total.as_ref(),
            ...
        }
    }
}

impl From<LlmCallRow> for LlmCall {
    fn from(row: LlmCallRow) -> Self {
        ...
        let token_metadata = TokenMetadata {
            ...,
            total: row.total_tokens,
        };
        ...
    }
}
```

We edit `src-tauri/src/commands/llms/chat.rs` next to build our response:

```rs
async fn chat_helper(
    zamm_api_keys: &ZammApiKeys,
    zamm_db: &ZammDatabase,
    provider: Service,
    llm: String,
    temperature: Option<f32>,
    prompt: Vec<ChatMessage>,
    http_client: reqwest_middleware::ClientWithMiddleware,
) -> ZammResult<LlmCall> {
    ...
    let token_metadata = TokenMetadata {
        ...,
        total: response
            .usage
            .as_ref()
            .map(|usage| usage.total_tokens as i32),
    };
    ...
}
```

We update our sample calls, such as `src-tauri/api/sample-calls/chat-start-conversation.yaml`:

```yaml
...
response:
  message: >
    {
      ...
      "tokens": {
        "prompt": 32,
        "response": 27,
        "total": 59
      }
    }

```

Finally, `src-svelte/src/lib/bindings.ts` is automatically updated by Specta.

### Frontend API call

Now that we have the backend API call working, we move over to the frontend side of the API boundary. We have Specta automatically regenerate `src-svelte/src/lib/bindings.ts` as usual. We then create a new page at `src-svelte/src/routes/chat/+page.svelte`:

```svelte
<script lang="ts">
  import Chat from "./Chat.svelte";
</script>

<Chat />

```

and define a skeleton version of the `Chat` component at `src-svelte/src/routes/chat/Chat.svelte`, just enough for us to trigger the API call:

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import TextInput from "$lib/controls/TextInput.svelte";
  import Button from "$lib/controls/Button.svelte";
  import { type ChatMessage, chat } from "$lib/bindings";
  import { snackbarError } from "$lib/snackbar/Snackbar.svelte";

  let currentMessage: string;
  let conversation: ChatMessage[] = [
    {
      role: "System",
      text: "You are ZAMM, a chat program. Respond in first person.",
    },
  ];

  async function sendChat() {
    const message = currentMessage.trim();
    if (message) {
      const chatMessage: ChatMessage = {
        role: "Human",
        text: message,
      };
      conversation.push(chatMessage);
      currentMessage = "";

      try {
        let llmCall = await chat("OpenAI", "gpt-3.5-turbo", null, conversation);
        conversation.push(llmCall.response.completion);
      } catch (err) {
        snackbarError(err as string);
      }
    }
  }
</script>

<InfoBox title="Chat">
  <form on:submit|preventDefault={sendChat}>
    <label for="message" class="accessibility-only">Chat with the AI:</label>
    <TextInput name="message" placeholder="Type your message here..." bind:value={currentMessage} />
    <Button text="Send" />
  </form>
</InfoBox>

<style>
  form {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
  }
</style>
```

We define a new Storybook story at `src-svelte/src/routes/chat/Chat.stories.ts`:

```ts
import Chatcomponent from "./Chat.svelte";
import type { StoryObj } from "@storybook/svelte";

export default {
  component: Chatcomponent,
  title: "Screens/Chat/Conversation",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: Chatcomponent,
  props: args,
});

export const Tablet: StoryObj = Template.bind({}) as any;
Tablet.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

```

and then define a new test at `src-svelte/src/routes/chat/Chat.test.ts`:

```ts
import { expect, test, vi, type Mock } from "vitest";
import "@testing-library/jest-dom";

import { render, screen } from "@testing-library/svelte";
import Chat from "./Chat.svelte";
import userEvent from "@testing-library/user-event";
import { TauriInvokePlayback } from "$lib/sample-call-testing";
import { animationSpeed } from "$lib/preferences";

describe("Chat conversation", () => {
  let tauriInvokeMock: Mock;
  let playback: TauriInvokePlayback;

  beforeAll(() => {
    animationSpeed.set(10);
  });

  beforeEach(() => {
    tauriInvokeMock = vi.fn();
    vi.stubGlobal("__TAURI_INVOKE__", tauriInvokeMock);
    playback = new TauriInvokePlayback();
    tauriInvokeMock.mockImplementation(
      (...args: (string | Record<string, string>)[]) =>
        playback.mockCall(...args),
    );

    vi.stubGlobal("requestAnimationFrame", (fn: FrameRequestCallback) => {
      return window.setTimeout(() => fn(Date.now()), 16);
    });
  });

  afterEach(() => {
    vi.unstubAllGlobals();
  });

  async function sendChatMessage(message: string, correspondingApiCallSample: string) {
    expect(tauriInvokeMock).not.toHaveBeenCalled();
    playback.addSamples(correspondingApiCallSample);

    const chatInput = screen.getByLabelText("Chat with the AI:");
    expect(chatInput).toHaveValue("");
    await userEvent.type(chatInput, message);
    await userEvent.click(screen.getByRole("button", { name: "Send" }));

    expect(tauriInvokeMock).toBeCalledTimes(1);
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    tauriInvokeMock.mockClear();
  }

  test("can start a conversation", async () => {
    render(Chat, {});
    await sendChatMessage("Hello, does this work?", "../src-tauri/api/sample-calls/chat-start-conversation.yaml");
  });
});

```

We have the line

```ts
expect(tauriInvokeMock).toHaveReturnedTimes(1);
```

because this test suspicously succeeds even when we don't call it with the right arguments, as we can see when we edit `src-svelte/src/lib/sample-call-testing.ts`:

```ts

export class TauriInvokePlayback {
  ...

  mockCall(
    ...
  ): Promise<Record<string, string>> {
    ...
      if (typeof process === "object") {
        console.error(errorMessage);
        throw new Error(errorMessage);
      } else {
        ...
      }
    ...
  }

  ...
}
```

For some the thrown error does not make the test fail in this case. We replace `toBeCalledTimes` with `toHaveReturnedTimes` in the rest of the tests, just in case the same error lurks there. We also replace `toHaveBeenLastCalledWith("get_system_info")` with `toHaveReturnedTimes` in `src-svelte/src/routes/components/Metadata.test.ts`, but not in:

- `src-svelte/src/lib/Switch.test.ts` because `recordSoundDelaySpy` is not an API call
- `src-svelte/src/routes/components/api-keys/Display.test.ts` because the `API key error` test is mocking a hypothetical error because we haven't triggered an actual error call

In `Display.test.ts`, we realize that we haven't tested the return method, and so we instead do a bit of refactoring by introducing `getOpenAiStatus` and then using that to check the result of the API call with the invalid filename:

```ts
describe("API Keys Display", () => {
  ...

  beforeEach(() => {
    ...
    clearAllMessages();
  });

  function getOpenAiStatus() {
    const openAiRow = screen.getByRole("row", { name: /OpenAI/ });
    const openAiKeyCell = within(openAiRow).getAllByRole("cell")[1];
    return openAiKeyCell.textContent;
  }

  async function checkSampleCall(filename: string, expected_display: string) {
    ...
    expect(getOpenAiStatus()).toBe(expected_display);
  }

  ...

  test("can submit with invalid file", async () => {
    systemInfo.set({
      ...NullSystemInfo,
    });
    await checkSampleCall(
      "../src-tauri/api/sample-calls/get_api_keys-empty.yaml",
      "Inactive",
    );
    tauriInvokeMock.mockClear();
    playback.addSamples(
      "../src-tauri/api/sample-calls/set_api_key-invalid-filename.yaml",
      "../src-tauri/api/sample-calls/get_api_keys-openai.yaml",
    );

    await toggleOpenAIForm();
    await userEvent.type(screen.getByLabelText("Export from:"), "/");
    await userEvent.type(screen.getByLabelText("API key:"), "0p3n41-4p1-k3y");
    await userEvent.click(screen.getByRole("button", { name: "Save" }));
    await waitFor(() => expect(tauriInvokeMock).toHaveBeenCalledTimes(2));
    expect(getOpenAiStatus()).toBe("Active");
    expect(tauriInvokeMock).toHaveReturnedTimes(1);

    render(Snackbar, {});
    const alerts = screen.queryAllByRole("alertdialog");
    expect(alerts).toHaveLength(1);
    expect(alerts[0]).toHaveTextContent("Is a directory (os error 21)");
  });
});
```

All our tests pass after all, so it appears the same error is not lurking around secretly. We continue by creating a second API request that continues the conversation:

```yaml
request:
  - chat
  - >
    {
      "provider": "OpenAI",
      "llm": "gpt-3.5-turbo",
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
          "text": "Hello! I'm ZAMM, a chat program. I'm here to assist you. What can I help you with today?"
        },
        {
          "role": "Human",
          "text": "Tell me something funny."
        }
      ]
    }
response:
  message: >
    {
      "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
      "timestamp": "2024-01-16T08:50:19.738093890",
      "llm": {
        "name": "gpt-3.5-turbo-0613",
        "requested": "gpt-3.5-turbo",
        "provider": "OpenAI"
      },
      "request": {
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
            "text": "Hello! I'm ZAMM, a chat program. I'm here to assist you. What can I help you with today?"
          },
          {
            "role": "Human",
            "text": "Tell me something funny."
          }
        ],
        "temperature": 1.0
      },
      "response": {
        "completion": {
          "role": "AI",
          "text": "Sure, here's a light-hearted joke for you:\n\nWhy don't scientists trust atoms?\n\nBecause they make up everything!"
        }
      },
      "tokens": {
        "prompt": 72,
        "response": 24
      }
    }

```

We refactor the existing test at `src-tauri/src/commands/llms/chat.rs` to generalize to other API calls and network requests, and apply the new function to our new API call sample:

```rs
...

#[cfg(test)]
mod tests {
    ...

    async fn test_llm_api_call(recording_path: &str, sample_path: &str) {
        let recording_path = PathBuf::from(recording_path);
        ...
        let sample = read_sample(sample_path);
        ...
    }

    #[tokio::test]
    async fn test_start_conversation() {
        test_llm_api_call(
            "api/sample-call-requests/start-conversation.json",
            "api/sample-calls/chat-start-conversation.yaml",
        ).await;        
    }

    #[tokio::test]
    async fn test_continue_conversation() {
        test_llm_api_call(
            "api/sample-call-requests/continue-conversation.json",
            "api/sample-calls/chat-continue-conversation.yaml",
        ).await;        
    }
}

```

The file at `src-tauri/api/sample-call-requests/continue-conversation.json` gets automatically created with the first run of the test, and we update the `yaml` file based on that.

We can now modify `src-svelte/src/routes/chat/Chat.test.ts` to continue the conversation:

```ts
  test("can start and continue a conversation", async () => {
    ...
    await sendChatMessage(
      "Tell me something funny.",
      "../src-tauri/api/sample-calls/chat-continue-conversation.yaml",
    );
  });
```

This works, but note that nothing is displayed onscreen. We edit `src-svelte/src/routes/chat/Chat.svelte` to show the chat message bubbles as well, using the strategy documented [here](https://css-tricks.com/snippets/css/css-triangle/):

```svelte
<script lang="ts">
  ...
  export let conversation: ChatMessage[] = [
    ...
  ];

  async function sendChat() {
    const message = currentMessage.trim();
    if (message) {
      ...
      conversation = [...conversation, chatMessage];
      ...

      try {
        ...
        conversation = [...conversation, llmCall.response.completion];
      } ...
    }
  }
</script>

<InfoBox title="Chat">
  <div class="conversation" role="list">
    {#if conversation.length > 1}
      {#each conversation.slice(1) as message}
        <div class:message class={message.role.toLowerCase()} role="listitem">
          <div class="arrow"></div>
          <div class="text">
            {message.text}
          </div>
        </div>
      {/each}
    {/if}
  </div>

  ...
</InfoBox>

<style>
  .conversation {
    margin-bottom: 1rem;
  }

  .message {
    --message-color: gray;
    --arrow-size: 0.5rem;
    position: relative;
  }

  .message .text {
    margin: 0.5rem var(--arrow-size);
    border-radius: var(--corner-roundness);
    width: fit-content;
    max-width: 60%;
    padding: 0.75rem;
    box-sizing: border-box;
    background-color: var(--message-color);
    white-space: pre-line;
  }

  .message .arrow {
    position: absolute;
    width: 0;
    height: 0;
    bottom: var(--arrow-size);
    border: var(--arrow-size) solid transparent;
  }

  .message.human {
    --message-color: #E5FFE5;
  }

  .message.human .text {
    margin-left: auto;
  }

  .message.human .arrow {
    float: right;
    right: calc(-1 * var(--arrow-size));
    border-left-color: var(--message-color); 
  }

  .message.ai {
    --message-color: #E5E5FF;
  }

  .message.ai .text {
    margin-right: auto;
  }

  .message.ai .arrow {
    float: left;
    left: calc(-1 * var(--arrow-size));
    border-right-color: var(--message-color); 
  }

  ...
</style>

```

Note that we need to change the messages update code from `conversation.push` to `conversation = [...]` because otherwise, Svelte doesn't realize it needs to refresh the DOM.

Also note that we need to set `white-space: pre-line` for the newlines. We make sure to demonstate:

- a long message that requires word wrapping
- a short message that doesn't require word wrapping but should be flush with the side it's next to
- a message with multiple lines.

We edit `src-svelte/src/routes/chat/Chat.stories.ts` to make use of the newly exported `conversation` prop:

```ts
...
import type { ChatMessage } from "$lib/bindings";

...


export const Empty: StoryObj = Template.bind({}) as any;
Empty.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

export const NotEmpty: StoryObj = Template.bind({}) as any;
const conversation: ChatMessage[] = [
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
    text: "Hello! I'm ZAMM, a chat program. I'm here to assist you. What can I help you with today?",
  },
  {
    role: "Human",
    text: "Tell me something really funny, like really funny. Make me laugh hard.",
  },
  {
    role: "AI",
    text: "Sure, here's a light-hearted joke for you:\n\nWhy don't scientists trust atoms?\n\nBecause they make up everything!",
  },
];
NotEmpty.args = {
  conversation,
};
...

```

We test these changes in `src-svelte/src/routes/chat/Chat.test.ts`:

```ts
...

import { render, screen, waitFor } from "@testing-library/svelte";
...
import { TauriInvokePlayback, type ParsedCall } from "$lib/sample-call-testing";
...
import type { ChatMessage, LlmCall } from "$lib/bindings";

describe("Chat conversation", () => {
  ...

  async function sendChatMessage(
    ...
  ) {
    ...
    const nextExpectedApiCall: ParsedCall = playback.unmatchedCalls.slice(-1)[0];
    const nextExpectedCallArgs = nextExpectedApiCall.request[1] as Record<string, any>;
    const nextExpectedMessage = nextExpectedCallArgs["prompt"].slice(-1)[0] as ChatMessage;
    const nextExpectedHumanPrompt = nextExpectedMessage.text;

    ...
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(screen.getByText(nextExpectedHumanPrompt)).toBeInTheDocument();

    ...
    const lastResult: LlmCall = tauriInvokeMock.mock.results[0].value;
    const aiResponse = lastResult.response.completion.text;
    const lastSentence = aiResponse.split("\n").slice(-1)[0];
    await waitFor(() => {
      expect(screen.getByText(new RegExp(lastSentence))).toBeInTheDocument();
    });

    tauriInvokeMock.mockClear();
  }

  ...
});
```

`eslint` complains about the line length in `src-svelte/src/routes/chat/Chat.stories.ts`, so instead we do:

```ts
const conversation: ChatMessage[] = [
  ...
  {
    role: "AI",
    text:
      "Hello! I'm ZAMM, a chat program. I'm here to assist you. " +
      "What can I help you with today?",
  },
  ...
  {
    role: "AI",
    text:
      "Sure, here's a light-hearted joke for you:\n\n" +
      "Why don't scientists trust atoms?\n\n" +
      "Because they make up everything!",
  },
];
```

Now that we have the basic functionality down, we tweak the look of the page further. We add some placeholder text for new empty conversations at `src-svelte/src/routes/chat/Chat.svelte`:

```svelte
<InfoBox title="Chat">
  <div class="conversation" role="list">
    {#if conversation.length > 1}
      ...
    {:else}
      <p class="empty-conversation">This conversation is currently empty.<br>Get it started by typing a message below.</p>
    {/if}
  </div>

  ...
</InfoBox>

<style>
  ...

  .empty-conversation {
    color: var(--color-faded);
    font-size: 0.85rem;
    font-style: italic;
    text-align: center;
  }

  ...
</style>
```

We can also try to wrap things in a bigger container, which we'll need eventually to grow the conversation panel:

```svelte
<InfoBox title="Chat">
  <div class="chat-container">
    <div class="conversation" role="list">
      ...
    </div>

    ...
  </div>
</InfoBox>

<style>
  .chat-container {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .conversation {
    flex: 1;
  }

  ...
</style>
```

We refactor the message display out into `src-svelte/src/routes/chat/Message.svelte`:

```svelte
<script lang="ts">
  import type { ChatMessage } from "$lib/bindings";
  export let message: ChatMessage;
</script>

<div class:message class={message.role.toLowerCase()} role="listitem">
  <div class="arrow"></div>
  <div class="text">
    {message.text}
  </div>
</div>

<style>
  .message {
    --message-color: gray;
    --arrow-size: 0.5rem;
    position: relative;
  }

  .message .text {
    margin: 0.5rem var(--arrow-size);
    border-radius: var(--corner-roundness);
    width: fit-content;
    max-width: 60%;
    padding: 0.75rem;
    box-sizing: border-box;
    background-color: var(--message-color);
    white-space: pre-line;
  }

  .message .arrow {
    position: absolute;
    width: 0;
    height: 0;
    bottom: var(--arrow-size);
    border: var(--arrow-size) solid transparent;
  }

  .message.human {
    --message-color: #e5ffe5;
  }

  .message.human .text {
    margin-left: auto;
  }

  .message.human .arrow {
    float: right;
    right: calc(-1 * var(--arrow-size));
    border-left-color: var(--message-color);
  }

  .message.ai {
    --message-color: #e5e5ff;
  }

  .message.ai .text {
    margin-right: auto;
  }

  .message.ai .arrow {
    float: left;
    left: calc(-1 * var(--arrow-size));
    border-right-color: var(--message-color);
  }
</style>

```

so that `src-svelte/src/routes/chat/Chat.svelte` will start looking like:

```svelte
<script lang="ts">
  ...
  import Message from "./Message.svelte";
  ...
</script>

<InfoBox title="Chat">
  ...
        {#each conversation.slice(1) as message}
          <Message message={message} />
        {/each}
  ...
</InfoBox>

<style>
  ... message-specific styles snipped ...
</style>
```

We create stories for individual messages at `src-svelte/src/routes/chat/Message.stories.ts`:

```ts
import Message from "./Message.svelte";
import type { StoryObj } from "@storybook/svelte";

export default {
  component: Message,
  title: "Screens/Chat/Message",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: Message,
  props: args,
});

export const Human: StoryObj = Template.bind({}) as any;
Human.args = {
  message: {
    role: "Human",
    text: "Hello, does this work?",
  },
};
Human.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

export const AI: StoryObj = Template.bind({}) as any;
AI.args = {
  message: {
    role: "AI",
    text:
      "Hello! I'm ZAMM, a chat program. I'm here to assist you. " +
      "What can I help you with today?",
  },
};
AI.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

export const AIMultiline: StoryObj = Template.bind({}) as any;
AIMultiline.args = {
  message: {
    role: "AI",
    text:
      "Sure, here's a light-hearted joke for you:\n\n" +
      "Why don't scientists trust atoms?\n\n" +
      "Because they make up everything!",
  },
};
AIMultiline.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

```

and record these in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...
    {
    path: ["screens", "chat", "message"],
    variants: ["human", "ai", "ai-multiline"],
  },
];
```

and add the new screenshots at `src-svelte/screenshots/baseline/screens/chat/message`.

Now, we'll try to make the conversation span the entire viewport. We repeatedly run into problems here, so we try a test with demo CSS [here](https://jsfiddle.net/x237gqsz/):

```html
<div class="A">
    <p>This is the prescript.</p>
    <div class="B">
        <div class="C">
            <p>This is C content.</p>
            <p>It keeps going and going.</p>
            <p>It overflows B.</p>
            <p>It just doesn't stop.</p>
            <p>Eventually, it should scroll.</p>
            <p>We're not quite there yet.</p>
            <p>But we will be soon.</p>
            <p>Ah finally, we have arrived at our destination.</p>
        </div>
    </div>
    <p>This is the postscript.</p>
</div>

```

```css
.A {
    display: flex;
    flex-direction: column;
    height: 95vh; /* or a specific height */
    background-color: red;
    padding: 1rem;
    box-sizing: border-box;
}

.B {
    flex: 1; /* This makes B take up all available space */
    overflow-y: scroll; /* Scrollbar if content overflows */
    background-color: blue;
    padding: 1rem;
}

.C {
    background-color: green;
}
```

This looks as we expect, but doesn't reproduce the problem in our code. We try again and reproduce *and* fix it [here](https://jsfiddle.net/gpxmde2s/):

```html
<div class="page">
  <div class="info-box">
    <p>This is the info box title.</p>
    <div class="chat">
        <div class="conversation-container">
            <div class="conversation">
                <p>This is conversational content.</p>
                <p>It keeps going and going.</p>
                <p>It overflows B.</p>
                <p>It just doesn't stop.</p>
                <p>Eventually, it should scroll.</p>
                <p>We're not quite there yet.</p>
                <p>But we will be soon.</p>
                <p>Very soon.</p>
                <p>Ah finally, we have arrived at our destination.</p>
            </div>
        </div>
        <p>This is where you enter in new chat messages.</p>
    </div>
  </div>
</div>

```

```css
p:first-child {
  margin-top: 0;
}

p:last-child {
  margin-bottom: 0;
}

.page {
  height: 95vh;
  background-color: black;
  padding: 0.5rem;
  box-sizing: border-box;
}

.info-box {
  background-color: purple;
  color: white;
  padding: 1rem;
  height: 100%;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.chat {
  flex: 1;
  display: flex;
  flex-direction: column;
  background-color: red;
  padding: 1rem;
  box-sizing: border-box;
}

.conversation-container {
  flex: 1;
  background-color: blue;
  padding: 0.5rem;
  box-sizing: border-box;
}

.conversation {
  background-color: green;
  padding: 1rem;
  max-height: 10rem;
  overflow-y: scroll;
}
```

The fix involves explicitly setting the `max-height` of the conversation container. Take that one line out, and no scrollbar appears. We will have to do this with JavaScript after the component gets mounted. We edit `src-svelte/src/routes/chat/Chat.svelte`:

```svelte
<script lang="ts">
  ...
  import { onMount } from "svelte";

  ...
  let conversationContainer: HTMLDivElement;
  let conversationView: HTMLDivElement;

  onMount(() => {
    conversationView.style.maxHeight = `${conversationContainer.clientHeight}px`;
    // scroll to last element
    conversationView.scrollTop = conversationView.scrollHeight;
  });
  ...
</script>

</script>

<InfoBox title="Chat" fullHeight>
  <div class="chat-container">
    <div class="conversation-container" bind:this={conversationContainer}>
      <div class="conversation" role="list" bind:this={conversationView}>
        ...
      </div>
    </div>

    ...
  </div>
</InfoBox>

<style>
  .chat-container {
    height: 100%;
    ...
  }

  .conversation-container {
    flex-grow: 1;
  }

  .conversation {
    max-height: 8rem;
    overflow-y: scroll;
  }

  ...
</style>

```

where we set an initial `max-height` of `8rem` to make it possible to view some of the conversation even if the JavaScript fails for some reason. We use the code mentioned [here](https://stackoverflow.com/a/270628) to scroll; we don't use the more "modern" `scrollIntoView` because that requires identifying the specific message element we want to scroll into the view.

We now have to set all the parent elements to take up the full space, starting with `src-svelte/src/lib/InfoBox.svelte` to take in the `fullHeight` prop:

```svelte
<script lang="ts">
  ...
  export let fullHeight: boolean = false;
  ...
</script>

<section
  ...
  class:full-height={fullHeight}
  ...
>
  ...
</section>

<style>
  ...

  .container.full-height, .container.full-height .border-container, .container.full-height .info-box {
    height: 100%;
    box-sizing: border-box;
  }

  .container.full-height .info-box {
    display: flex;
    flex-direction: column;
  }

  .container.full-height .info-content {
    flex: 1;
  }

  ...
</style>
```

We continue to one of the root files at `src-svelte/src/routes/AppLayout.svelte`:

```css
  main {
    ...
    height: 100%;
  }
```

We go down to `src-svelte/src/routes/PageTransition.svelte`:

```css
  .transition-container {
    ...
    height: 100%;
    ...
  }
```

That's it for prod, but we edit `src-svelte/src/lib/__mocks__/MockAppLayout.svelte` as well:

```css
  .storybook-wrapper {
    height: calc(100vh - 2rem);
    ...
    box-sizing: border-box;
  }
```

Note: We find out below that this fails existing screenshot tests, so we copy it and apply these two lines to the copy instead.

We subtract `2rem` here to account for the padding that Storybook puts around the story. Note that `AnimationControl` is in the parent hierarchy here, unlike with the regular prod hierarchy, so we edit it in `src-svelte/src/routes/AnimationControl.svelte` too:

```css
  .container {
    height: 100%;
  }
```

Horizontal scrollbars are appearing for the messages too, so we edit `src-svelte/src/routes/chat/Message.svelte` to fix the insivible portions of the chat trianges, and also shift them to be more vertically centered for single-line messages:

```css
  ...

  .message .arrow {
    ...
    bottom: 0.75rem;
    ...
  }

  ...

  .message.human .arrow {
    right: 0;
    border-right: none;
    ...
  }

  ...

  .message.ai .arrow {
    left: 0;
    border-left: none;
    ...
  }
```

Doing this slightly modifies how the edges of the arrows are rendered, so we update the gold screenshots.

Storybook's default tablet screen size is a bit too vertically large for the chat messages to render nicely for our screenshots, so we follow the instructions [here](https://storybook.js.org/docs/essentials/viewport#add-new-devices) to add new devices to `src-svelte/.storybook/preview.ts`:

```ts
...
import { MINIMAL_VIEWPORTS } from '@storybook/addon-viewport';

...
const preview = {
  parameters: {
    ...
    viewport: {
      viewports: {
        ...MINIMAL_VIEWPORTS,
        smallTablet: {
          name: "Small Tablet",
          styles: {
            width: "834px",
            height: "800px",
          },
        },
      },
    },
  },
};

...
```

This is still too large, as Playwright screenshots on Webkit show empty space if the element is larger than the viewport, so we shrink it further to

```ts
          styles: {
            width: "834px",
            height: "650px",
          },
```

Finally, we edit `src-svelte/src/routes/chat/Chat.stories.ts` to use the new `MockAppLayout`, which will now span the entire height of the screen (unlike `storybook-root`) and extend the chat conversation history to produce a scrollbar:

```ts
...
import MockAppLayout from "$lib/__mocks__/MockAppLayout.svelte";
import type { StoryFn, ... } from "@storybook/svelte";
...

export default {
  ...,
  decorators: [
    (story: StoryFn) => {
      return {
        Component: MockAppLayout,
        slot: story,
      };
    },
  ],
};

...

export const Empty: StoryObj = Template.bind({}) as any;
...
Empty.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};

export const NotEmpty: StoryObj = Template.bind({}) as any;
const conversation: ChatMessage[] = [
  ...
  {
    role: "Human",
    text: "Okay, we need to fill this chat up to produce a scrollbar for Storybook. Say short phrases like \"Yup\" to fill this chat up quickly.",
  },
  {
    role: "AI",
    text: "Yup",
  },
  {
    role: "Human",
    text: "Nay",
  },
  {
    role: "AI",
    text: "Yay",
  },
  {
    role: "Human",
    text: "Say...",
  },
  {
    role: "AI",
    text: "AIs don't actually talk like this, you know? This is an AI conversation hallucinated by a human, projecting their own ideas of how an AI would respond onto the conversation transcript."
  },
];
...
NotEmpty.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};

```

We realize that we also want to scroll to the bottom of the chat conversation history every time a new message is sent, so we edit `src-svelte/src/routes/chat/Chat.svelte` to scroll to the bottom every time a new chat message is appended to the conversation view -- but to do so only after a slight delay so that the browser has time to render the new page:

```ts
  ...
  let conversationContainer: HTMLDivElement | undefined = undefined;
  let conversationView: HTMLDivElement | undefined = undefined;

  onMount(() => {
    if (conversationView && conversationContainer) {
      conversationView.style.maxHeight = `${conversationContainer.clientHeight}px`;
    }
    showChatBottom();
  });

  function showChatBottom() {
    if (conversationView) {
      conversationView.scrollTop = conversationView.scrollHeight;
    }
  }

  async function sendChat() {
    ...
    if (message) {
      ...
      conversation = [...conversation, chatMessage];
      ...
      setTimeout(showChatBottom, 50);

      try {
        ...
        conversation = [...conversation, llmCall.response.completion];
        setTimeout(showChatBottom, 50);
      } ...
    }
  }
```

Unfortunately, editing `MockAppLayout.svelte` causes the existing full-body screenshot tests to fail, so we instead copy it over to `src-svelte/src/lib/__mocks__/MockFullPageLayout.svelte` and restore `MockAppLayout` to its original form. We edit `src-svelte/src/routes/chat/Chat.stories.ts` again to use this instead:

```ts
...
import MockFullPageLayout from "$lib/__mocks__/MockFullPageLayout.svelte";
...

export default {
  ...,
  decorators: [
    (story: StoryFn) => {
      return {
        Component: MockFullPageLayout,
        ...
      };
    },
  ],
};
```

We've developed the conversation page enough now that we should add this to the Storybook tests, by creating a new entry in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...
    {
    path: ["screens", "chat", "conversation"],
    variants: ["empty", "not-empty"],
    screenshotEntireBody: true,
  },
];
```

From this, we realize that we need to update the conversation view when the window resizes (and therefore changes the info box size as well). We edit `src-svelte/src/routes/chat/Chat.svelte` to take this into account by defining this function to resize the conversation view:

```ts
  function resizeConversationView() {
    if (conversationView && conversationContainer) {
      conversationView.style.maxHeight = "8rem";
      setTimeout(() => {
        if (conversationView && conversationContainer) {
          conversationView.style.maxHeight = `${conversationContainer.clientHeight}px`;
          showChatBottom();
        }
      }, 10);
    }
  }
```

Note that we initially set the `maxHeight` to something small in order to force the browser to recalculate the `clientHeight` of the conversation container. We also automatically scroll to the bottom as well after resetting the height, because otherwise there may be a lot of empty space at the bottom on Webkit. We keep the `maxHeight` big enough to still see individual messages, in case there are any problems with the CSS rendering.

We realize that we only need to depend on `conversationView` for the first part, and that we can do `requestAnimationFrame` first to do a smoother repaint that avoids making the whole screen flash for the user:

```ts
  function resizeConversationView() {
    if (conversationView) {
      ...
      requestAnimationFrame(() => {
        ...
      });
    }
  }
```

At first, we try using a ResizeObserver:

```ts
      const resizeObserver = new ResizeObserver(() => {
        resizeConversationView();
      });
      resizeObserver.observe(conversationContainer);
```

However, this doesn't work as well as listening for a window resize, because the conversation container will only resize if it has the opportunity to get bigger, but will not automatically get smaller because its size is bounded on the lower end by the conversation view. As such, we use the return value from the lifecycle hook as demonstrated [here](https://blog.sethcorker.com/question/how-do-you-use-the-resize-observer-api-in-svelte/) to do:

```ts
  onMount(() => {
    resizeConversationView();
    window.addEventListener("resize", resizeConversationView);

    return () => {
      window.removeEventListener("resize", resizeConversationView);
    };
  });
```

Now we finally have screenshots that are mostly acceptable, except that in Webkit (and only in Webkit) the rendered messages [overlap](/zamm-notes/screenshots/02b77e5.png) with the `h2` element. We commit our progress so far before proceeding any further with the fix.

To fix this, we first try editing `src-svelte/src/lib/InfoBox.svelte` to insert a div whose only purpose is to provide the h2 element with a background:

```svelte
<script lang="ts">
  ...
  let h2Background: HTMLDivElement | undefined = undefined;
  ...
</script>

<section ...>
  ...
    <div class="info-box">
      <h2 ...>{title}</h2>
      <div class="h2-background" bind:this={h2Background}></div>
      ...
    </div>
  ...
</section>

<style>
  .info-box {
    --content-padding: 1rem;
    --h2-top-margin: -0.25rem;
    --h2-bottom-margin: calc(0.5 * var(--cut));
    position: relative;
    z-index: 2;
    padding: var(--content-padding);
    ...
  }

  .info-box .h2-background {
    --background-height: calc((var(--h2-font-size)) + var(--content-padding) + var(--h2-bottom-margin));
    z-index: 1;
    background-color: white;
    position: absolute;
    min-height: var(--background-height);
    left: 0;
    top: 0;
    right: 0;
  }

  .info-box .info-content {
    position: relative;
    z-index: 0;
  }

  ...

  .info-box h2 {
    ...
    position: relative;
    z-index: 2;
    margin: var(--h2-top-margin) 0 var(--h2-bottom-margin) var(--cut);
  }

  ...
</style>
```

where we now edit `src-svelte/src/routes/styles.css` to define:

```css
:root {
  ...
  --h2-font-size: 1.2rem;
  ...
}

...

h2 {
  ...
  font-size: var(--h2-font-size);
  ...
}
```

We go back to `src-svelte/src/lib/InfoBox.svelte` and try editing `.info-box` to clip the content box and prevent the h2 background element from overlapping with the border box edges, using `background-color: red` during development to check that we're clipping the edges successfully:

```css
  .info-box {
    ...

    --clip-width: 100%;
    --clip-height: 100%;
    --clip-cut: calc(2.2 * 0.707 * var(--cut));
    --rounded-cut: calc(0.5 * 0.707rem);
    clip-path: polygon(
      0% var(--clip-cut),
      var(--clip-cut) 0%,

      calc(var(--clip-width) - var(--rounded-cut)) 0%,
      var(--clip-width) var(--rounded-cut),

      var(--clip-width) calc(var(--clip-height) - var(--clip-cut)),
      calc(var(--clip-width) - var(--clip-cut)) var(--clip-height),

      var(--rounded-cut) var(--clip-height),
      0% calc(var(--clip-height) - var(--rounded-cut))
    );
  }
```

This presents problems with overlapping with the info box grow animations. We tried to get the clip to grow at the same exact rate as the border box animation by copying their `PropertyAnimation` definitions, but this made the animation lag by too much:

```ts
  function revealOutline(
    ...
  ): TransitionConfig {
    ...

    const growClipWidth = new PropertyAnimation({
      timing: timing.growX(),
      property: "--clip-width",
      min: minWidth,
      max: actualWidth,
      unit: "px",
      easingFunction: titleIsMultiline ? cubicOut : linear,
    });

    const growClipHeight = new PropertyAnimation({
      timing: timing.growY(),
      property: "--clip-height",
      min: minHeight,
      max: actualHeight,
      unit: "px",
      easingFunction: cubicInOut,
    });

    return {
      ...,
      tick: (tGlobalFraction: number) => {
        ...

        if (h2Background) {
          const clipWidth = growClipWidth.tickForGlobalTime(tGlobalFraction);
          const clipHeight = growClipHeight.tickForGlobalTime(tGlobalFraction);
          h2Background.setAttribute(
            "style",
            clipWidth + clipHeight,
          );
        }

        ...
      },
    };
  }
```

Instead, we try simply making it appear at the very end, when animations are finished:

```svelte
<script lang="ts">
  ...

  function revealOutline(
    ...
  ): TransitionConfig {
    ...

    return {
      ...,
      tick: (tGlobalFraction: number) => {
        ...
        if (tGlobalFraction === 0) {
          if (h2Background) {
            h2Background.classList.add("wait-for-infobox");
          }
        }
        else if (tGlobalFraction === 1) {
          ...
          if (h2Background) {
            h2Background.classList.remove("wait-for-infobox");
            h2Background.removeAttribute("style");
          }
        }
      },
    };
  }

  ...
</script>

...

<style>
  ...

  .info-box :global(.h2-background.wait-for-infobox) {
    display: none;
  }

  ...
</style>
```

In the process of fixing this, we notice that we have inadvertently placed `h2` styles in two separate places in `src-svelte/src/routes/styles.css`, so we merge the two, from:

```css
h2 {
  ...
}

h2 {
  margin-bottom: 0;
}
```

to

```css
h2 {
  ...
  margin-bottom: 0;
  ...
}
```

We also merge the two `.info-box h2` definitions in `src-svelte/src/lib/InfoBox.svelte`:

```css
  .info-box h2 {
    --cursor-opacity: 0;
    margin: -0.25rem 0 0.5rem var(--cut);
    text-align: left;
  }
```

The screenshot for the chat conversation now looks a lot better, but the ones for the API keys displays are now off. This is complicated enough that we try to produce a minimal reproduction, and eventually do so at [https://jsfiddle.net/e7os0xpu/](https://jsfiddle.net/e7os0xpu/):

```html
<div class="border-container">
  <div class="info-box">
    <h2>Chat</h2>
    <div class="conversation">
      <div class="human message">I'm trying to create a chat component</div>
      
      <div class="ai message">There's two participants</div>
      
      <div class="human message">Blah blah blah</div>
      
      <div class="ai message">And blabbly blabbly blah. Blah blah blah. Really long messages seem to do it.</div>
      
      <div class="human message">Okay, let's try even longer messages. Even longer. And longer. And longer. And longer. And longer. And longer. And longer. And longer. And longer.</div>
      
      <div class="ai message">Yup</div>
      
      <div class="human message">Nay</div>
      
      <div class="ai message">Yay</div>
      
      <div class="human message">Say...</div>
      
      <div class="ai message">Let's cap this off with a really long message. It will span multiple lines. Many lines. It just keeps going and going and going and going and going and going and now it's done.</div>
    </div>
  </div>
</div>

```

with the corresponding CSS that produces a similar effect on Webkit:

```css
body {
  font-family: sans-serif;
}

.border-container {
  filter: drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.4));
}

.info-box {
  padding: 1rem;
}

.conversation {
  max-height: 41px;
  overflow-y: scroll;
  background-color: pink;
}

.message {
  position: relative;
  margin: 0.5rem;
  max-width: 60%;
  padding: 0.75rem;
  background-color: #e5ffe5;
}

.message.human {
  margin-left: auto;
}

.message.ai {
  margin-right: auto;
}

```

While this bug does not exist on Firefox or Chrome, we should still care about supporting Webkit browsers because Tauri depends on Webkit, and this bug reproduces on the development version of Tauri. As such, we follow [these instructions](https://webkit.org/reporting-bugs/) and [these guidelines](https://webkit.org/bug-report-guidelines/) to report [this bug](https://bugs.webkit.org/show_bug.cgi?id=267972) to the Webkit team.

Given that it is triggered by the drop-shadow filter, we try applying the filter only to the border box in `src-svelte/src/lib/InfoBox.svelte`:

```css
  .border-box {
    ...
    filter: url(#round) drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.4));;
    ...
  }
```

Incidentally, this also fixes the problem caused by the "Save" button appearing too early in the form open/close animation on Tauri, which therefore appears to be caused by the same Webkit bug. In retrospect, we should have created the minimal repro and then applied this minimal fix, instead of going straight for the workaround.

#### Nits

Disable autocomplete on the chat form at `src-svelte/src/routes/chat/Chat.svelte`:

```svelte
    <form autocomplete="off" ...>
      ...
    </form>
```

On Chrome, the scrollbars actually appear on the conversation window. This is when we find out the difference between `overflow: scroll;` and `overflow: auto;`. We therefore edit `src-svelte/src/routes/chat/Chat.svelte` as such:

```css
  .conversation {
    ...
    overflow-y: auto;
    ...
  }
```

#### Stylizing the form as a single button group

We can try to stylize the elements separately, for example by editing `src-svelte/src/lib/controls/Button.svelte`:

```svelte
<script lang="ts">
  ...
  export let rightEnd: boolean = false;
</script>

<button class="outer" class:right-end={rightEnd} ...>
  <div class="inner" class:right-end={rightEnd}>{text}</div>
</button>

<style>
  .outer,
  .inner {
    --cut: var(--controls-corner-cut);
    ...
  }

  .outer.right-end, .inner.right-end {
    background:
      linear-gradient(
        -45deg,
        var(--border-color) 0 calc(var(--cut) + var(--diagonal-border)),
        var(--background-color) 0
      )
      bottom right;
    background-origin: border-box;
    background-repeat: no-repeat;
    -webkit-mask:
      linear-gradient(-45deg, transparent 0 var(--cut), #fff 0) bottom right;
    mask:
      linear-gradient(-45deg, transparent 0 var(--cut), #fff 0) bottom right;
    mask-size: 100%;
    mask-repeat: no-repeat;
  }

  ...
</style>
```

while defining the new CSS variable in `src-svelte/src/routes/styles.css`:

```css
  :root {
    ...
    --controls-corner-cut: 7px;
    ...
  }
```

And we also edit `src-svelte/src/lib/controls/TextInput.svelte` in a similar manner:

```svelte
<script lang="ts">
  ...
  export let leftEnd = false;
</script>

<div class="fancy-input" class:left-end={leftEnd}>
  ...
</div>

<style>
  ...

  .fancy-input.left-end {
    --cut: var(--controls-corner-cut);
    --background-color: var(--color-background);
    --border-color: #ccc;
    --border: 0.15rem;
    --diagonal-border: 0.001rem;
    padding: 0.375rem 0.75rem;
    margin-right: -1px;
    border: var(--border) solid var(--border-color);
    background:
    linear-gradient(
          135deg,
          var(--border-color) 0 calc(var(--cut) + var(--diagonal-border)),
          var(--background-color) 0
        )
        top left;
    background-origin: border-box;
    background-repeat: no-repeat;
    -webkit-mask:
      linear-gradient(135deg, transparent 0 var(--cut), #fff 0) top left;
    mask:
      linear-gradient(135deg, transparent 0 var(--cut), #fff 0) top left;
    mask-size: 100%;
    mask-repeat: no-repeat;

    transition-property: filter;
    transition: calc(0.5 * var(--standard-duration)) ease-out;
  }

  .fancy-input.left-end:hover {
    filter: brightness(1.05);
  }

  .fancy-input.left-end input[type="text"] {
    font-family: var(--font-body);
    font-weight: normal;
    border-bottom: none;
  }

  .fancy-input.left-end input[type="text"] + .focus-border {
    display: none;
  }
</style>
```

And `src-svelte/src/routes/chat/Chat.svelte`, removing the gap from the form and setting the new boolean prompts on the text input and button:

```svelte
    <form ...>
      ...
      <TextInput
        leftEnd
        ...
      />
      <Button rightEnd ... />
    </form>
```

Unfortunately this way of aligning the two elements works only on one browser at a time, as different browsers will calculate ever so slightly different pixel height values. We therefore do a larger refactor. We start off by editing `src-svelte/src/lib/controls/Button.svelte` to allow for showing just the inner part of the button for us:

```svelte
<script lang="ts">
  ...
  export let unwrapped = false;
  export let rightEnd = false;
</script>

{#if unwrapped}
  <button class="cut-corners inner" class:right-end={rightEnd} type="submit">
    {text}
  </button>
{:else}
  <button class="cut-corners outer" class:right-end={rightEnd} type="submit">
    <div class="cut-corners inner" class:right-end={rightEnd}>{text}</div>
  </button>
{/if}

<style>
  ...

  .outer.right-end,
  .inner.right-end {
    --cut-top-left: 0.01rem;
    height: 100%;
  }

  ...
</style>
```

Note that the `height: 100%;` will be used to make the button grow in height along with the textarea later on when we implement the form.

We add options to this component instead of creating a new one because the two buttons share a lot of similar code. We refactor out the inner and outer css so that it just looks like

```css
  .outer,
  .inner {
    --cut: var(--controls-corner-cut);
    font-size: 0.9rem;
    font-family: var(--font-body);
    text-transform: uppercase;
    transition-property: filter, transform;
    transition: calc(0.5 * var(--standard-duration)) ease-out;
  }
```

with the rest of it in `src-svelte/src/routes/styles.css`:

```css
.cut-corners {
  --cut-top-left: var(--controls-corner-cut);
  --cut-bottom-right: var(--controls-corner-cut);
  --border: 0.15rem;
  --diagonal-border: calc(var(--border) * 0.8);
  --border-color: var(--color-border);
  --background-color: var(--color-background);

  border: var(--border) solid var(--border-color);
  background:
    linear-gradient(
        -45deg,
        var(--border-color) 0 calc(var(--cut-bottom-right) + var(--diagonal-border)),
        var(--background-color) 0
      )
      bottom right / 50% 100%,
    linear-gradient(
        135deg,
        var(--border-color) 0 calc(var(--cut-top-left) + var(--diagonal-border)),
        var(--background-color) 0
      )
      top left / 50% 100%;
  background-origin: border-box;
  background-repeat: no-repeat;
  -webkit-mask:
    linear-gradient(-45deg, transparent 0 var(--cut-bottom-right), #fff 0) bottom right,
    linear-gradient(135deg, transparent 0 var(--cut-top-left), #fff 0) top left;
  -webkit-mask-size: 51% 100%;
  -webkit-mask-repeat: no-repeat;
  mask:
    linear-gradient(-45deg, transparent 0 var(--cut-bottom-right), #fff 0) bottom right,
    linear-gradient(135deg, transparent 0 var(--cut-top-left), #fff 0) top left;
  mask-size: 51% 100%;
  mask-repeat: no-repeat;
}

.cut-corners.outer {
  padding: 1px;
  --background-color: var(--color-background);
  --border: 2px;
  --diagonal-border: 2.5px;
  --cut-top-left: 8px;
  --cut-bottom-right: 8px;
}
```

because we will want to reuse that logic for the form component. We also edit `src-svelte/src/lib/controls/TextInput.svelte` to refactor out most of the CSS that applies to generic form elements for this app, so that the remaining CSS for the input element is simply:

```css
  input[type="text"] {
    min-width: 1rem;
    width: 100%;
    border-bottom: 1px solid var(--color-border);
    background-color: var(--color-background);
    font-family: var(--font-mono);
    font-weight: bold;
    font-size: 1rem;
  }
```

The rest, we once again copy into `src-svelte/src/routes/styles.css`:

```css
input[type="text"], textarea {
  border: none;
}

input[type="text"]:focus, textarea:focus {
  outline: none;
}

input[type="text"]::placeholder, textarea::placeholder {
  font-style: italic;
}
```

We create our new form component at `src-svelte/src/routes/chat/Form.svelte`, making use of the previous refactors we just did:

```svelte
<script lang="ts">
  import autosize from "autosize";
  import Button from "$lib/controls/Button.svelte";
  import { onMount } from "svelte";

  export let sendChatMessage: (message: string) => void;
  let currentMessage = "";
  let textareaInput: HTMLTextAreaElement;

  onMount(() => {
    autosize(textareaInput);

    return () => {
      autosize.destroy(textareaInput);
    };
  });

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Enter" && !event.shiftKey && !event.ctrlKey) {
      event.preventDefault(); // Prevent newline insertion
      submitChat();
    }
  }

  function submitChat() {
    const message = currentMessage.trim();
    if (message) {
      sendChatMessage(currentMessage);
      currentMessage = "";
      requestAnimationFrame(() => {
        autosize.update(textareaInput);
      });
    }
  }
</script>

<form
  class="cut-corners outer"
  autocomplete="off"
  on:submit|preventDefault={submitChat}
>
  <label for="message" class="accessibility-only">Chat with the AI:</label>
  <textarea
    id="message"
    name="message"
    placeholder="Type your message here..."
    rows="1"
    on:keydown={handleKeydown}
    bind:this={textareaInput}
    bind:value={currentMessage}
  />
  <Button unwrapped rightEnd text="Send" />
</form>

<style>
  form {
    display: flex;
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }

  textarea {
    margin: 0 0.75rem;
    flex: 1;
    background-color: transparent;
    font-size: 1rem;
    font-family: var(--font-body);
    width: 100%;
    min-height: 1.2rem;
    resize: none;
  }
</style>

```

We want the input to grow as the user types in enough text for the text to wrap around. As it turns out, this is a somewhat non-trivial problem that appears to be best solved by the [`autosize`](http://www.jacklmoore.com/autosize/) package. The Svelte-specific [`svelte-autoresize-textarea`](https://github.com/ankurrsinghal/svelte-autoresize-textarea) project appears to not handle our textarea correctly. We add the types for `autosize` too:

```bash
$ yarn add -D @types/autosize
```

As mentioned earlier, we ensure that the send button also grows as the textarea grows, so that it doesn't remain at an awkwardly small size. However, it is a bit awkward for the relevant CSS to be in a different component's code.

We use this new form in `src-svelte/src/routes/chat/Chat.svelte`, rewriting the `sendChat` function to fit the new expected form and removing all form-related logic that are now in the form component:

```svelte
<script lang="ts">
  ...
  import Form from "./Form.svelte";
  ...

  async function sendChatMessage(message: string) {
    const chatMessage: ChatMessage = {
      role: "Human",
      text: message,
    };
    conversation = [...conversation, chatMessage];
    setTimeout(showChatBottom, 50);

    try {
      let llmCall = await chat("OpenAI", "gpt-3.5-turbo", null, conversation);
      conversation = [...conversation, llmCall.response.completion];
      setTimeout(showChatBottom, 50);
    } catch (err) {
      snackbarError(err as string);
    }
  }
</script>

<InfoBox ...>
  <div ...>
    ...

    <Form {sendChatMessage} />
  </div>
</InfoBox>
```

We update the chat screenshots and commit.

Of course, if we let the text area grow, we should screenshot that. We edit `src-svelte/src/routes/chat/Form.svelte` to allow parent components to set an initial `currentMessage`:

```ts
  ...
  export let currentMessage = "";
  ...
```

We edit `src-svelte/src/routes/chat/Chat.svelte` to do the same:

```svelte
<script lang="ts">
  ...
  export let initialMessage: string = "";
  ...
</script>

<InfoBox ...>
  <div ...>
    ...

    <Form ... currentMessage={initialMessage} />
  </div>
</InfoBox>
```

and add the new story to `src-svelte/src/routes/chat/Chat.stories.ts`

```ts
export const MultilineChat: StoryObj = Template.bind({}) as any;
MultilineChat.args = {
  conversation,
  initialMessage: "This is what happens when the user types in so much text, it wraps around and turns the text input area into a multiline input. The send button's height should grow in line with the overall text area height.",
};
MultilineChat.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};
```

We make sure to test this in our Storybook screenshots at `src-svelte/src/routes/storybook.test.ts`:

```ts
  {
    path: ["screens", "chat", "conversation"],
    variants: [..., "multiline-chat"],
    ...
  },
```

We realize upon interacting with this new story that the textarea actually grows downwards. We need to also resize the chat conversation display whenever an autosize event gets kicked off. We edit `src-svelte/src/routes/chat/Form.svelte`:

```ts
  ...
  export let onTextInputResize: () => void = () => undefined;
  ...

  onMount(() => {
    ...
    textareaInput.addEventListener("autosize:resized", onTextInputResize);

    ...
  });
```

and then we make use of this in `src-svelte/src/routes/chat/Chat.svelte`:

```svelte
<InfoBox ...>
  ...
  <Form ... onTextInputResize={resizeConversationView} />
</InfoBox>
```

Doing a screenshot test of this fix will be left as a TODO, as it requires an interaction before the screenshot takes place, so we simply test this manually.

#### Stylizing top and bottom shadows

There appears to be a [pure CSS](https://lea.verou.me/blog/2012/04/background-attachment-local/) solution to adding top and bottom shadows to the scrollable div. We see from [this question](https://stackoverflow.com/questions/71658855/horizontall-scrolling-div-shadow-does-not-cover-the-child-background) that this strategy is known to not work for div's; even the second proposed answer does not work. As such, we resort to using some JavaScript after all.

We find [this resource](https://css-tricks.com/scroll-shadows-with-javascript/) on implementing scroll shadows, which links to [this page](https://cushionapp.com/journal/overflow-shadows-using-the-intersection-observer-api) on doing it with the `IntersectionObserver` API, which is a slightly more efficient way of checking if scroll shadows should be shown than the usual way of checking the scroll height, such as in [this example](https://codepen.io/mindstorm/pen/PoqxXN). We see that this API is [supported](https://caniuse.com/?search=intersectionobserver) across 97% of browsers, and read [the documentation](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) on it.

We implement it in `src-svelte/src/routes/chat/Chat.svelte`:

```svelte
<script lang="ts">
  ...
  let topIndicator: HTMLDivElement;
  let bottomIndicator: HTMLDivElement;
  let topShadow: HTMLDivElement;
  let bottomShadow: HTMLDivElement;

  onMount(() => {
    ...
    let topScrollObserver = new IntersectionObserver(
      intersectionCallback(topShadow),
    );
    topScrollObserver.observe(topIndicator);
    let bottomScrollObserver = new IntersectionObserver(
      intersectionCallback(bottomShadow),
    );
    bottomScrollObserver.observe(bottomIndicator);

    return () => {
      ...
      topScrollObserver.disconnect();
      bottomScrollObserver.disconnect();
    };
  });

  function intersectionCallback(shadow: HTMLDivElement) {
    return (entries: IntersectionObserverEntry[]) => {
      let indicator = entries[0];
      if (indicator.isIntersecting) {
        shadow.classList.remove("visible");
      } else {
        shadow.classList.add("visible");
      }
    };
  }

  ...
</script>

<InfoBox ...>
  ...
    <div class="conversation-container" ...>
      <div class="shadow top" bind:this={topShadow}></div>
      <div class="conversation" ...>
        <div class="indicator top" bind:this={topIndicator}></div>
        ...
        <div class="indicator bottom" bind:this={bottomIndicator}></div>
      </div>
      <div class="shadow bottom" bind:this={bottomShadow}></div>
    </div>
  ...
</InfoBox>

<style>
  ...

  .conversation-container {
    ...
    position: relative;
  }

  .conversation {
    ...
    position: relative;
  }

  ...

  .shadow {
    z-index: 1;
    height: 0.375rem;
    width: 100%;
    position: absolute;
    display: none;
  }

  .conversation-container :global(.shadow.visible) {
    display: block;
  }

  .shadow.top {
    top: 0;
    background-image: radial-gradient(
      farthest-side at 50% 0%,
      rgba(150, 150, 150, 0.4) 0%,
      rgba(0, 0, 0, 0) 100%
    );
  }

  .shadow.bottom {
    bottom: 0;
    background-image: radial-gradient(
      farthest-side at 50% 100%,
      rgba(150, 150, 150, 0.4) 0%,
      rgba(0, 0, 0, 0) 100%
    );
  }

  .indicator {
    height: 1px;
    width: 100%;
  }

  .indicator.top {
    margin-bottom: -1px;
  }

  .indicator.bottom {
    margin-top: -1px;
  }
</style>
```

Note that:

- we make sure that the top and bottom indicators have negative margins so as to preserve the original layout
- we set `position: relative;` on the conversation container so as to ensure that the shadows `z-index` and absolute positioning work with respect to the conversation container and not any parent elements

Now our Vitest tests are failing because Vitest/Jest does not mock the `IntersectionObserver` API, so we mock it ourselves in `src-svelte/src/routes/chat/Chat.test.ts` by looking at the examples [here](https://stackoverflow.com/questions/58884397/jest-mock-intersectionobserver) and [here](https://stackoverflow.com/questions/63665377/mock-for-intersection-observer-in-jest-and-typescript):

```ts
  beforeEach(() => {
    ...
    window.IntersectionObserver = vi.fn(() => {
      return {
        observe: vi.fn(),
        unobserve: vi.fn(),
        disconnect: vi.fn(),
      };
    });
  });
```

However, this produces the TypeScript error of:

```
/root/zamm/src-svelte/src/routes/chat/Chat.test.ts:31:5
Error: Type 'Mock<[], { observe: Mock<[target: Element], void>; unobserve: Mock<[target: Element], void>; disconnect: Mock<[], void>; }>' is not assignable to type '{ new (callback: IntersectionObserverCallback, options?: IntersectionObserverInit | undefined): IntersectionObserver; prototype: IntersectionObserver; }'.
  Type '{ observe: Mock<[target: Element], void>; unobserve: Mock<[target: Element], void>; disconnect: Mock<[], void>; }' is missing the following properties from type 'IntersectionObserver': root, rootMargin, thresholds, takeRecords
    });
    window.IntersectionObserver = vi.fn(() => {
      return {
```

We do a quick fix by forcibly overriding the type checker:

```ts
    window.IntersectionObserver = vi.fn(() => {
      ...
    }) as unknown as typeof IntersectionObserver;
```

As expected, the only screenshots that change are for the `multiline chat` and `not empty` conversations; the empty conversation has no scrollbars and therefore renders exactly the same.

For completeness' sake, we should add a screenshot for the top shadow as well. We edit `src-svelte/src/routes/chat/Chat.svelte` for an additional option not to scroll to the bottom on initial render:

```ts
  ...
  export let showMostRecentMessage = true;
  ...

  function resizeConversationView() {
    if (conversationView) {
      ...
      requestAnimationFrame(() => {
        if (conversationView && conversationContainer) {
          ...
          if (showMostRecentMessage) {
            showChatBottom();
          }
        }
      });
    }
  }
```

We add a story at `src-svelte/src/routes/chat/Chat.stories.ts`:

```ts
export const BottomScrollIndicator: StoryObj = Template.bind({}) as any;
BottomScrollIndicator.args = {
  conversation,
  showMostRecentMessage: false,
};
BottomScrollIndicator.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};
```

and register it at `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...
  {
    path: ["screens", "chat", "conversation"],
    variants: [..., "bottom-scroll-indicator"],
    screenshotEntireBody: true,
  },
];
```

We check the new screenshot at `src-svelte/screenshots/baseline/screens/chat/conversation/bottom-scroll-indicator.png` to see that it looks as expected, and commit everything.

#### Adding chat bubbles

We should make this more like a regular chat interface and add chat bubbles. We see that there is a great tutorial already [here](https://phuoc.ng/collection/css-animation/typing-indicator/), which makes use of the single-keyframe technique mentioned [here](https://lea.verou.me/blog/2012/12/animations-with-one-keyframe/).

We start off by refactoring the previous `Message` component into `src-svelte/src/routes/chat/MessageUI.svelte`:

```svelte
<script lang="ts">
  export let role: "System" | "Human" | "AI";
</script>

<div class:message={true} class={role.toLowerCase()} role="listitem">
  <div class="arrow"></div>
  <div class="text">
    <slot />
  </div>
</div>

<style>
  ...
</style>
```

We then edit the original `src-svelte/src/routes/chat/Message.svelte` to make use of this:

```svelte
<script lang="ts">
  import type { ChatMessage } from "$lib/bindings";
  import MessageUI from "./MessageUI.svelte";
  export let message: ChatMessage;
</script>

<MessageUI role={message.role}>
  {message.text}
</MessageUI>

```

Now we can create our typing indicator chat bubble that makes use of our existing chat bubble styling. We create `src-svelte/src/routes/chat/TypingIndicator.svelte`:

```svelte
<script lang="ts">
  import MessageUI from "./MessageUI.svelte";
  import { animationsOn } from "$lib/preferences";

  $: animationState = $animationsOn ? "running" : "paused";
</script>

<MessageUI role="AI">
  <div class="typing-indicator" style="--animation-state: {animationState};">
    <div class="dot"></div>
    <div class="dot"></div>
    <div class="dot"></div>
  </div>
</MessageUI>

<style>
  .typing-indicator {
    --animation-state: running;
    --dot-speed: 0.5s;
    align-items: center;
    display: flex;
    justify-content: center;
    gap: 0.25rem;
    min-height: 22px;
  }

  .dot {
    --initial-animation-progress: 0;
    border-radius: 9999px;
    height: 0.5rem;
    width: 0.5rem;

    background: rgba(148 163 184 / 1);
    opacity: 1;
    animation: blink var(--dot-speed) infinite alternate;
    animation-delay: calc(
      (-2 + var(--initial-animation-progress)) * var(--dot-speed)
    );
    animation-play-state: var(--animation-state);
  }

  .dot:nth-child(1) {
    --initial-animation-progress: 0;
  }

  .dot:nth-child(2) {
    --initial-animation-progress: 0.25;
  }

  .dot:nth-child(3) {
    --initial-animation-progress: 0.5;
  }

  @keyframes blink {
    100% {
      opacity: 0;
    }
  }
</style>

```

Note that we have modified the original CSS to make our interactions with the `--animation-state` variable more understandable:

- We start the animation from full opacity to zero, and then make sure that each successive dot is further along in the animation than the previous one, so that it looks like the chat animation is going towards the right rather than the left
- We make the animation delay negative so that even in the animation disabled state, the different dots still appear in different amounts of opacity
- We make it so that the dots still look good even when animations are disabled; we could customize the animated state to display them at different points in the animation than they would appear in the animation-disabled state, but we avoid adding such complexity in order to keep the screenshot tests highly correlated with the working state of the animated versions
- We set the keyframe to 100% instead of 50% for easier reasoning, and set the animation to alternate so that we get the same effect as before
- We separate overall animation speed from how far along in the animation each dot should appear as

Now we can make use of this in `src-svelte/src/routes/chat/Chat.svelte`:

```svelte
<script lang="ts">
  ...
  import TypingIndicator from "./TypingIndicator.svelte";
  ...
  export let expectingResponse = false;
  ...

  async function sendChatMessage(message: string) {
    ...
    conversation = [...conversation, chatMessage];
    expectingResponse = true;
    ...

    try {
      let llmCall = await chat(...);
      ...
    } catch (err) {
      ...
    } finally {
      expectingResponse = false;
    }
  }
</script>

<InfoBox ...>
  ...
        {#if conversation.length > 1}
          {#each ... as message}
            ...
          {/each}
          {#if expectingResponse}
            <TypingIndicator />
          {/if}
        {:else}
          ...
        {/if}
  ...
</InfoBox>
```

Finally, we add two stories, one with animations on and one with animations off, to `src-svelte/src/routes/chat/Chat.stories.ts`, making sure to also add in the Svelte stores decorator:

```ts
...
import SvelteStoresDecorator from "$lib/__mocks__/stores";
...

export default {
  ...,
  decorators: [
    SvelteStoresDecorator,
    ...
  ],
};

...

const shortConversation: ChatMessage[] = [
  {
    role: "System",
    text: "You are ZAMM, a chat program. Respond in first person.",
  },
  {
    role: "Human",
    text: "Hello, does this work?",
  },
];

...

export const TypingIndicator: StoryObj = Template.bind({}) as any;
TypingIndicator.args = {
  conversation: shortConversation,
  expectingResponse: true,
};
TypingIndicator.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};

export const TypingIndicatorStatic: StoryObj = Template.bind({}) as any;
TypingIndicatorStatic.args = {
  conversation: shortConversation,
  expectingResponse: true,
};
TypingIndicatorStatic.parameters = {
  preferences: {
    animationsOn: false,
  },
  viewport: {
    defaultViewport: "smallTablet",
  },
};

```

We register the second one at `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      ...
      "typing-indicator-static",
    ],
    ...
  },
];
```

and increase the retries because Webkit is unreliably slow at resizing the component frame:

```ts
        {
          retry: 1,
          timeout: ...,
        },
```

We add the new screenshot at `src-svelte/screenshots/baseline/screens/chat/conversation/typing-indicator-static.png`.

#### End-to-end screenshot

We edit `webdriver/test/specs/e2e.test.js`:

```js
describe("Welcome screen", function () {
  ...

  it("should allow navigation to the chat page", async function () {
    findAndClick('a[title="Chat"]');
    await browser.pause(500); // for CSS transitions to finish
    expect(
      await browser.checkFullPageScreen("chat-screen", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  ...
});
```

to produce a new screenshot at `webdriver/screenshots/baseline/desktop_wry/chat-screen-800x600.png`.

#### Initial info box reveal

We check that the chat conversation page does the info box reveal in a way that we expect. We add a new story to `src-svelte/src/routes/chat/Chat.stories.ts`:

```ts
...
import MockPageTransitions from "$lib/__mocks__/MockPageTransitions.svelte";
...

export const FullPage: StoryObj = Template.bind({}) as any;
FullPage.args = {
  conversation,
};
FullPage.decorators = [
  (story: StoryFn) => {
    return {
      Component: MockPageTransitions,
      slot: story,
    };
  },
];
```

The info box actually shrinks into nothingness instead of expanding to cover the entire content. We realize that this is because `src-svelte/src/lib/__mocks__/MockPageTransitions.svelte` should use `MockFullPageLayout` instead of `MockAppLayout`, because the info box is made to use the full height of the parent div but the parent div here has zero height by default:

```ts
<script lang="ts">
  import MockFullPageLayout from "./MockFullPageLayout.svelte";
  ...
</script>

<MockFullPageLayout>
  ...
</MockFullPageLayout>

```

We add a similar story to `src-svelte/src/routes/settings/Settings.stories.ts`, which doesn't have a full page story yet:

```ts
...
import MockPageTransitions from "$lib/__mocks__/MockPageTransitions.svelte";
import type { ..., StoryFn } from "@storybook/svelte";

...

export const FullPage: StoryObj = Template.bind({}) as any;
FullPage.decorators = [
  (story: StoryFn) => {
    return {
      Component: MockPageTransitions,
      slot: story,
    };
  },
];
```

This will allow us to make sure that any changes to the info box reveal animation doesn't impact the settings page, either.

#### Persistent chat

We're not persisting chat across sessions just yet, but we should still persist chat for the duration the app is open. We create `src-svelte/src/routes/chat/PersistentChat.svelte` as such:

```svelte
<script lang="ts" context="module">
  import type { ChatMessage } from "$lib/bindings";

  let persistentConversation: ChatMessage[] = [
    {
      role: "System",
      text: "You are ZAMM, a chat program. Respond in first person.",
    },
  ];
</script>

<script lang="ts">
  import Chat from "./Chat.svelte";
</script>

<Chat bind:conversation={persistentConversation} />

```

We create `src-svelte/src/routes/chat/PersistentChatView.svelte` for purely testing purposes:

```svelte
<script lang="ts">
  import PersistentChat from "./PersistentChat.svelte";

  let visible = true;

  function remount() {
    visible = false;
    setTimeout(() => (visible = true), 50);
  }
</script>

<button on:click={remount}>Remount</button>

{#if visible}
  <PersistentChat />
{/if}

```

Then we create `src-svelte/src/routes/chat/PersistentChat.test.ts` as largely a copy of the regular chat test, except that we remount the component and check that the chat is still there:

```ts
import { expect, test, vi, type Mock } from "vitest";
import "@testing-library/jest-dom";

import { render, screen, waitFor } from "@testing-library/svelte";
import PersistentChatView from "./PersistentChatView.svelte";
import userEvent from "@testing-library/user-event";
import { TauriInvokePlayback, type ParsedCall } from "$lib/sample-call-testing";
import { animationSpeed } from "$lib/preferences";
import type { ChatMessage, LlmCall } from "$lib/bindings";

describe("Chat conversation", () => {
  let tauriInvokeMock: Mock;
  let playback: TauriInvokePlayback;

  beforeAll(() => {
    animationSpeed.set(10);
  });

  beforeEach(() => {
    tauriInvokeMock = vi.fn();
    vi.stubGlobal("__TAURI_INVOKE__", tauriInvokeMock);
    playback = new TauriInvokePlayback();
    tauriInvokeMock.mockImplementation(
      (...args: (string | Record<string, string>)[]) =>
        playback.mockCall(...args),
    );

    vi.stubGlobal("requestAnimationFrame", (fn: FrameRequestCallback) => {
      return window.setTimeout(() => fn(Date.now()), 16);
    });
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

  async function sendChatMessage(
    message: string,
    correspondingApiCallSample: string,
  ) {
    expect(tauriInvokeMock).not.toHaveBeenCalled();
    playback.addSamples(correspondingApiCallSample);
    const nextExpectedApiCall: ParsedCall =
      playback.unmatchedCalls.slice(-1)[0];
    const nextExpectedCallArgs = nextExpectedApiCall.request[1] as Record<
      string,
      any
    >;
    const nextExpectedMessage = nextExpectedCallArgs["prompt"].slice(
      -1,
    )[0] as ChatMessage;
    const nextExpectedHumanPrompt = nextExpectedMessage.text;

    const chatInput = screen.getByLabelText("Chat with the AI:");
    expect(chatInput).toHaveValue("");
    await userEvent.type(chatInput, message);
    await userEvent.click(screen.getByRole("button", { name: "Send" }));
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(screen.getByText(nextExpectedHumanPrompt)).toBeInTheDocument();

    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    const lastResult: LlmCall = tauriInvokeMock.mock.results[0].value;
    const aiResponse = lastResult.response.completion.text;
    const lastSentence = aiResponse.split("\n").slice(-1)[0];
    await waitFor(() => {
      expect(screen.getByText(new RegExp(lastSentence))).toBeInTheDocument();
    });

    tauriInvokeMock.mockClear();
  }

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

To make sure that our test works as expected, we check that it fails if the `bind:` is removed from the `PersistentChat` component.

Finally, we edit `src-svelte/src/routes/chat/+page.svelte` to point to this new component:

```svelte
<script lang="ts">
  import PersistentChat from "./PersistentChat.svelte";
</script>

<PersistentChat />

```

#### Conversation reveal

We make the info box reveal animation smoother for the empty new conversation view. The steps are recorded at [`infobox.md`](/zamm-notes/infobox.md).

### CI run

#### Fixing the build

Our build step is now failing on CI. This is not the first time dependencies have changed during CI build, so we try adding `--frozen-lockfile` to all the steps that involve yarn installation. We start with `.github/workflows/tests.yaml`:

```yaml
...
  pre-commit:
    ...
    steps:
      ...
      - name: Install Node dependencies
        ...
        run: |
          yarn install --frozen-lockfile
      ...
  svelte:
    ...
    steps:
      ...
      - name: Install Node dependencies
        ...
        run: |
          yarn install --frozen-lockfile
          cd src-svelte && yarn install --frozen-lockfile && yarn svelte-kit sync
          cd ../webdriver && yarn install --frozen-lockfile
  e2e:
    ...
    steps:
      ...
      - name: Install Node dependencies
        ...
        run: |
          yarn install --frozen-lockfile
          cd src-svelte && yarn svelte-kit sync
```

We also visit `src-svelte/Makefile`:

```Makefile
build: ...
	yarn install --frozen-lockfile && yarn svelte-kit sync && yarn build
```

This still doesn't work, so we look deeper at the problem and see that it is because:

```
error vite@5.0.12: The engine "node" is incompatible with this module. Expected version "^18.0.0 || >=20.0.0". Got "16.20.2"
error Found incompatible module.
info Visit https://yarnpkg.com/en/docs/cli/install for documentation about this command.
```

We see that `vite` did indeed get upgraded when we asked to upgrade everything earlier. To preserve the build on older versions of GLIBC, we try adding resolutions to `package.json`:

```json
{
  ...,
  "resolutions": {
    "vite": "4.5.2"
  }
}
```

This time, `yarn install` updates `yarn.lock` to look like this:

```lock
vite@4.5.2, "vite@^3.0.0 || ^4.0.0 || ^5.0.0-0", "vite@^3.1.0 || ^4.0.0 || ^5.0.0-0", vite@^4.4.4:
  version "4.5.2"
  resolved "https://registry.yarnpkg.com/vite/-/vite-4.5.2.tgz#d6ea8610e099851dad8c7371599969e0f8b97e82"
  ...
```

We could do this more properly by building the frontend on newer versions of GLIBC, and only building the Rust app on an older operating system. However, that method is too involved for us right now.

#### Fixing Python tests

Next, our Python tests fail.

```
poetry run pytest -v
ERROR: while parsing the following warning configuration:

  ignore::pydantic.warnings.PydanticDeprecatedSince20

This error occurred:

Traceback (most recent call last):
  File "/home/runner/.cache/pypoetry/virtualenvs/zamm-2r_Pdn8S-py3.11/lib/python3.11/site-packages/_pytest/config/__init__.py", line 1761, in parse_warning_filter
Warning: ory: Type[Warning] = _resolve_warning_category(category_)
                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/runner/.cache/pypoetry/virtualenvs/zamm-2r_Pdn8S-py3.11/lib/python3.11/site-packages/_pytest/config/__init__.py", line 1799, in _resolve_warning_category
    m = __import__(module, None, None, [klass])
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ModuleNotFoundError: No module named 'pydantic'
```

We remove the `ignore::pydantic.warnings.PydanticDeprecatedSince20` from `src-python/pyproject.toml` that we had forgotten to remove when ripping the langchain stuff back out.

#### Fixing Svelte tests

The Svelte screenshot tests are failing as expected, so we update those after manually checking that they look as expected. The only Svelte test failing now is

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests > layout/app/static.png should render the same
 FAIL  src/routes/storybook.test.ts > Storybook visual tests > layout/app/static.png should render the same
TimeoutError: locator.getAttribute: Timeout 60000ms exceeded.
Call log:
  - waiting for locator('#storybook-root > :first-child')
```

We check to see if this is a fluke. Testing locally, we find that we run into an issue at the same URL:

```
$page is undefined

instance/$$self.$$.update@http://localhost:6006/src/routes/Sidebar.svelte?t=1706616164007:98:7
init@http://localhost:6006/node_modules/.cache/sb-vite/deps/chunk-AJUXQLVD.js?v=c0d94ec1:2200:6
Sidebar@http://localhost:6006/src/routes/Sidebar.svelte?t=1706616164007:108:7
```

We try editing `src-svelte/src/routes/Sidebar.svelte` from

```ts
  $: currentRoute = $page.url?.pathname || "/";
```

to

```ts
  $: currentRoute = $page?.url?.pathname || "/";
```

The story renders again and the screenshot test now passes locally and on CI, leaving us to wonder what had changed and how this would've worked before.

#### Fixing pre-commit checks

The pre-commit hook made changes:

```
pre-commit hook(s) made changes.
If you are seeing this message in CI, reproduce locally with: `pre-commit run --all-files`.
To run `pre-commit` as part of git workflow, use `pre-commit install`.
All changes made by hooks:
diff --git a/src-svelte/tsconfig.json b/src-svelte/tsconfig.json
index 5c56cee..fd4595a 100644
--- a/src-svelte/tsconfig.json
+++ b/src-svelte/tsconfig.json
@@ -8,6 +8,6 @@
     "resolveJsonModule": true,
     "skipLibCheck": true,
     "sourceMap": true,
-    "strict": true
-  }
+    "strict": true,
+  },
 }
```

This is not reproducible locally, even after upgrading the versions of the pre-commit hooks used locally. (Nor is it clear why the local and CI versions would diverge despite the exact revisions being specified.) The fix doesn't even make sense because the `json` file most definitely shouldn't have trailing commas. We look at what version of pre-commit is running locally:

```bash
$ pre-commit --version
pre-commit 3.6.0
```

and try to see if updating `PRE_COMMIT_VERSION` to `3.6.0` in `.github/workflows/tests.yaml` helps in making pre-commit output consistent between CI and the local build. This doesn't help after all.

We find that there is [an existing issue](https://github.com/prettier/prettier/issues/15942) around this, but this doesn't help how the run is inconsistent between the local machine and CI.

Weirdly enough, the version on CI says it is installed using Python 3.10.12 despite all Python locations pointing to 3.11.4:

```
  pipx install pre-commit==$PRE_COMMIT_VERSION
  shell: /usr/bin/bash -e {0}
  env:
    POETRY_VERSION: 1.5.1
    PYTHON_VERSION: 3.11.4
    NODEJS_VERSION: 20.5.1
    RUST_VERSION: 1.71.1
    PRE_COMMIT_VERSION: 3.6.0
    CACHE_ON_FAILURE: false
    CARGO_INCREMENTAL: 0
    pythonLocation: /opt/hostedtoolcache/Python/3.11.4/x64
    PKG_CONFIG_PATH: /opt/hostedtoolcache/Python/3.11.4/x64/lib/pkgconfig
    Python_ROOT_DIR: /opt/hostedtoolcache/Python/3.11.4/x64
    Python2_ROOT_DIR: /opt/hostedtoolcache/Python/3.11.4/x64
    Python3_ROOT_DIR: /opt/hostedtoolcache/Python/3.11.4/x64
    LD_LIBRARY_PATH: /opt/hostedtoolcache/Python/3.11.4/x64/lib
creating virtual environment...
installing pre-commit from spec 'pre-commit==3.6.0'...
done! ✨ 🌟 ✨
  installed package pre-commit 3.6.0, installed using Python 3.10.12
  These apps are now globally available
    - pre-commit
```

However, this seems unlikely to matter, so we first try clearing the local cache:

```bash
$ rm -rf ~/.cache/pre-commit
```

Now when we run `pre-commit run --show-diff-on-failure --color=always --all-files`, we finally recreate the same output as on CI.

#### Fixing end to end tests

The end-to-end screenshots are somehow missing the bottom half of the info box border on the settings and chat pages. We see if that's just a fluke. A second run causes all tests except for the new chat page screenshot to pass, so we see that this is indeed a fluke, but we must also grab a proper correct render of the page at least once to serve as a gold screenshot. We try upping the timeout in `webdriver/test/specs/e2e.test.js`:

```js
  it("should allow navigation to the chat page", async function () {
    ...
    await browser.pause(2500); // for page to finish rendering
    ...
  });
```

This doesn't appear to help, and in fact the settings page is also flaky, so we edit the test to make it possible to retry twice:

```ts
  it("should allow navigation to the settings page", async function () {
    this.retries(2);
    findAndClick('a[title="Settings"]');
    // check if sound is toggled on first
    const soundToggle = await $("aria/Sounds");
    const soundToggleState = await soundToggle.getAttribute("aria-checked");
    if (soundToggleState === "true") {
      findAndClick("aria/Sounds");
    }
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("settings-screen", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });
```

Upon further inspection, it appears that `aria/Sounds` has never been clicked successfully, and the only reason we don't realize this is because we didn't `await` on it. We change our tests to await on the initial page navigation click, and to await on . Because it appears that some level of mismatch is inevitable here, we also make the default `maxMismatch` non-zero:

```ts
const maxMismatch =
  process.env.MISMATCH_TOLERANCE === undefined
    ? 0.6
    : parseFloat(process.env.MISMATCH_TOLERANCE);

...

describe("App", function () {
  ...

  it("should allow navigation to the chat page", async function () {
    this.retries(2);
    await findAndClick('a[title="Chat"]');
    await $("button");
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("chat-screen", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });

  it("should allow navigation to the settings page", async function () {
    this.retries(2);
    await findAndClick('a[title="Settings"]');
    await $("label");
    await browser.pause(2500); // for page to finish rendering
    expect(
      await browser.checkFullPageScreen("settings-screen", {}),
    ).toBeLessThanOrEqual(maxMismatch);
  });
});
```

It is at this point that we finally get a complete match for the screenshot, so we commit that and are able to set the `maxMismatch` back to zero for the chat page. The settings page still produces a diff, but given that we were able to remove the flakiness for the chat page, we wonder if we can't do the same for the settings page with a longer timeout. We increase the pause to 5 seconds and remove the `await $("label");`, as it turns out that simply awaiting on an element returns immediately if the element isn't visible anyways.

Even this is ultimately unable to reduce the flakiness, so we finally set the `maxMismatch` to 1.0 because the settings page is otherwise flaky, but only for this specific test:

```ts
const maxMismatch = getEnvMismatchTolerance() ?? 0.0;

function getEnvMismatchTolerance() {
  return process.env.MISMATCH_TOLERANCE === undefined
    ? undefined
    : parseFloat(process.env.MISMATCH_TOLERANCE);
}

...

describe("App", function () {
  ...

  it("should allow navigation to the settings page", async function () {
    ...
    const customMaxMismatch = getEnvMismatchTolerance() ?? 1.0;
    ...
    expect(
      ...
    ).toBeLessThanOrEqual(customMaxMismatch);
  });
});
```

Note that we use `??` here because we want it to be possible to set the mismatch to 0 if so desired.

Even after merging, we find that new PRs are blocked due to the flakiness here. We try instead to see if we can tamp down on the flakiness by navigating between the pages a few times so that the full animation no longer plays, as it appears the initial animation render doesn't display shadows properly on Webkit on Linux:

```ts
  it("should allow navigation to the chat page", async function () {
    ...
    await findAndClick('a[title="Chat"]');
    await findAndClick('a[title="Dashboard"]');
    await findAndClick('a[title="Chat"]');
    ...
  });
```

We realize we've renamed the component to `Dashboard`, but haven't updated the link name, so we edit `src-svelte/src/routes/SidebarUI.svelte` accordingly:

```ts
  const routes: App.Route[] = [
    {
      name: "Dashboard",
      path: "/",
      icon: IconDashboard,
    },
    ...
  ];
```

We also set a default timeout so that our tests don't take forever to fail if we don't specify the right element to click on:

```ts
const DEFAULT_TIMEOUT = 5_000;

...

async function findAndClick(selector, timeout) {
  ...
  await button.waitForClickable({
    timeout: timeout ?? DEFAULT_TIMEOUT,
  });
  ...
}
```

When our Svelte screenshot tests fail, we realize we should change the selector in `src-svelte/src/routes/SidebarUI.test.ts` as well:

```ts
  beforeEach(() => {
    ...
    homeLink = screen.getByTitle("Dashboard");
    ...
  });
```

This finally brings the flakiness down to zero, and we are able to set the `maxMismatch` back to zero for all pages. We remove the code we added for a custom mismatch, but keep the refactor we made to make such a change easy.

#### Speeding up the CI build

The build step on CI now takes much longer than before, so we try to update the Docker container to have our new dependencies pre-installed to see if that makes it faster. We edit the `Dockerfile` to clone the new Rust forks before doing some initial Rust compilation:

```Dockerfile
...
COPY src-tauri/Cargo.lock Cargo.lock
RUN git clone --depth 1 --branch zamm/v0.0.0 https://github.com/amosjyng/async-openai.git /tmp/forks/async-openai && \
  git clone --depth 1 --branch zamm/v0.0.0 https://github.com/amosjyng/rvcr.git /tmp/forks/rvcr && \
  mkdir src && \
  ...
```

Then we edit `Makefile` because it appears we were incorrectly nesting folders inside of existing ones. We change it from

```Makefile
copy-docker-deps:
	mv -n /tmp/dependencies/src-svelte/forks/neodrag/packages/svelte/dist ./src-svelte/forks/neodrag/packages/svelte/dist
	mv -n /tmp/dependencies/node_modules ./node_modules
	mv -n /tmp/dependencies/src-svelte/node_modules ./src-svelte/node_modules
	mv -n /tmp/dependencies/target ./src-tauri/target
```

to

```Makefile
copy-docker-deps:
	mv -n /tmp/forks/async-openai/* ./forks/async-openai/
	mv -n /tmp/forks/rvcr/* ./forks/rvcr/
	mv -n /tmp/dependencies/src-svelte/forks/neodrag/packages/svelte/dist ./src-svelte/forks/neodrag/packages/svelte/
	mv -n /tmp/dependencies/node_modules ./
	mv -n /tmp/dependencies/src-svelte/node_modules ./src-svelte/
	mv -n /tmp/dependencies/target ./src-tauri/
```

and check that there is no longer an erroneously nested `src-tauri/target/target` directory.

We have now successfully cut down the "Build Artifacts" step from 10 minutes 24 seconds down to 4 minutes 48 seconds.

## UX nits

### Left-aligned text

The inter-word spacing of the chat messages gets a little too much sometimes. We try to edit `src-svelte/src/routes/chat/MessageUI.svelte` to change the text alignment to `left` instead of `justify`:

```css
  .message .text {
    ...
    text-align: left;
  }
```

Unfortunately, the text shifts left but the `<p>` elements stay at exactly the same width, causing the chat bubbles to appear as if they have heavy right padding. We follow the solution mentioned [here](https://stackoverflow.com/a/73198785):

```svelte
<script lang="ts">
  import { onMount } from "svelte";

  ...
  let textElement: HTMLDivElement;

  onMount(() => {
    setTimeout(() => {
      const range = document.createRange();
      range.selectNodeContents(textElement);
      const textRect = range.getBoundingClientRect();
      textElement.style.width = `${textRect.width}px`;
    }, 10);
  });
</script>

<div ...>
  ...
  <div class="text-container">
    <div class="text" bind:this={textElement}>
      <slot />
    </div>
  </div>
</div>

<style>
  ...

  .message .text-container {
    ...
  }

  .text-element {
    box-sizing: content-box;
  }

  ...

  .message.human .text-container {
    ...
  }

  ...

  .message.ai .text-container {
    ...
  }

  ...
</style>
```

Note that:

1. This requires us to rename the `text` div to `text-container`, and insert another `text` div inside that will have `box-sizing: content-box;` so that we can set the width directly on that.
2. The `requestAnimationFrame` trick does not work here, so we have to wait a bit for the browser to render the content properly before resizing it with JS.

We realize that `.text-element` does not actually affect anything, but that it works anyway because we haven't overridden `content-box` as the default box-sizing for divs. We fix this and keep it anyways to be explicit about our intentions:

```css
  .text {
    ...
  }
```

We now get this test error:

```
TypeError: range.getBoundingClientRect is not a function
 ❯ Timeout._onTimeout src/routes/chat/MessageUI.svelte:12:30
     10|       const range = document.createRange();
     11|       range.selectNodeContents(textElement);
     12|       const textRect = range.getBoundingClientRect();
       |                              ^
     13|       textElement.style.width = `${textRect.width}px`;
     14|     }, 10);
 ❯ listOnTimeout node:internal/timers:573:17
 ❯ processTimers node:internal/timers:514:7
```

We edit `src-svelte/src/routes/chat/Chat.test.ts` to mock the range:

```ts
  beforeEach(() => {
    ...
    window.document.createRange = vi.fn(() => {
      return {
        selectNodeContents: vi.fn(),
        getBoundingClientRect: vi.fn(() => {
          return {
            width: 10,
            height: 10,
            top: 0,
            left: 0,
            right: 10,
            bottom: 10,
          };
        }),
      };
    });
  });
```

This works, but results in the type error

```
/root/zamm/src-svelte/src/routes/chat/Chat.test.ts:38:5
Error: Type 'Mock<[], { new (): Range; prototype: Range; readonly START_TO_START: 0; readonly START_TO_END: 1; readonly END_TO_END: 2; readonly END_TO_START: 3; }>' is not assignable to type '() => Range'.
  Type '{ new (): Range; prototype: Range; readonly START_TO_START: 0; readonly START_TO_END: 1; readonly END_TO_END: 2; readonly END_TO_START: 3; }' is missing the following properties from type 'Range': commonAncestorContainer, cloneContents, cloneRange, collapse, and 25 more.
    }) as unknown as typeof IntersectionObserver;
    window.document.createRange = vi.fn(() => {
      return {
```

We fix this by doing a manual cast:

```ts
    window.document.createRange = vi.fn(() => {
      ...
    }) as unknown as Mock<[], Range>;
```

Now we encounter the error

```
TypeError: Cannot read properties of null (reading 'style')
 ❯ Timeout._onTimeout src/routes/chat/MessageUI.svelte:14:21
     12|         range.selectNodeContents(textElement);
     13|         const textRect = range.getBoundingClientRect();
     14|         textElement.style.width = `${textRect.width}px`;
       |                     ^
     15|       }
     16|     }, 10);
 ❯ listOnTimeout node:internal/timers:573:17
 ❯ processTimers node:internal/timers:514:7

This error originated in "src/routes/chat/Chat.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
```

We edit `src-svelte/src/routes/chat/MessageUI.svelte` to guard our new code with a null check:

```ts
  let textElement: HTMLDivElement | null;

  onMount(() => {
    setTimeout(() => {
      if (textElement) {
        ...
      }
    }, ...);
  });
```

For some reason, the above range mock fix does not work on CI. We still get the same "range.getBoundingClientRect is not a function" error. We discover that this is a known issue and try the workaround mentioned [here](https://github.com/jsdom/jsdom/issues/3002#issuecomment-1118039915) by mocking the function in `src-svelte/src/routes/chat/Chat.test.ts` directly on the prototype:

```ts
  beforeEach(() => {
    ...
    Range.prototype.getBoundingClientRect =  vi.fn(() => {
      return {
        x: 0,
        y: 0,
        width: 10,
        height: 10,
        top: 0,
        left: 0,
        right: 10,
        bottom: 10,
        toJSON: vi.fn(),
      };
    });
  });
```

This still doesn't work in CI. Because this concerns styling rather than essential functionality, and because this is already covered by the screenshot tests, we edit `src-svelte/src/routes/chat/MessageUI.svelte` to simply log and warn if the resizing isn't working:

```ts
  onMount(() => {
    setTimeout(() => {
      if (textElement) {
        try {
          ...
        } catch (err) {
          console.warn("Cannot resize chat message bubble: ", err);
        }
      }
    }, ...);
  });
```

### Chat arrow gap

On Firefox on Windows, we notice a gap between the arrow and the chat bubble. We close this gap in `src-svelte/src/routes/chat/MessageUI.svelte`:

```css
  ...

  .message.human .arrow {
    right: 1px;
    ...
  }

  ...

  .message.ai .arrow {
    left: 1px;
    ...
  }
```

### GPT 4 by default

So long as the user is unable to choose which model they would like to speak to, we should pick a good default for them. The current best model appears to be `GPT-4`, so we edit `src-svelte/src/routes/chat/Chat.svelte` to change the default:

```ts
  ...

  async function sendChatMessage(message: string) {
    ...

    try {
      let llmCall = await chat("OpenAI", "gpt-4", null, conversation);
      ...
    }
  }
```

The backend *code* doesn't need to change for this one. Instead, we just remove `src-tauri/api/sample-call-requests/continue-conversation.json` and `src-tauri/api/sample-call-requests/start-conversation.json` for a re-recording of the API calls, and update `src-tauri/api/sample-calls/chat-continue-conversation.yaml` and `src-tauri/api/sample-calls/chat-start-conversation.yaml` for the backend's own API calls.

Now the frontend test fails as well because the new response from GPT4

> Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!

contains special regex characters and therefore cannot be converted directly into a regex. Instead, we use the regex escape code mentioned in [this answer](https://stackoverflow.com/a/6969486) to edit `src-svelte/src/routes/chat/Chat.test.ts` to escape the special characters first before regex conversion:

```ts
  async function sendChatMessage(
    ...
  ) {
    ...

    await waitFor(() => {
      expect(
        screen.getByText(
          new RegExp(lastSentence.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")),
        ),
      ).toBeInTheDocument();
    });

    ...
  }
```

### Webdriver error

We now run into an error with the end-to-end tests when trying to merge.

```
[0-0] 2024-02-09T00:27:21.459Z ERROR @wdio/runner: Error: Failed to create session.
[0-0] Failed to match capabilities
[0-0]     at startWebDriverSession (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/webdriverio/node_modules/webdriver/build/utils.js:69:15)
[0-0]     at processTicksAndRejections (node:internal/process/task_queues:95:5)
[0-0]     at async Function.newSession (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/webdriverio/node_modules/webdriver/build/index.js:19:45)
[0-0]     at async remote (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/webdriverio/build/index.js:45:22)
[0-0]     at async Runner._startSession (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/@wdio/runner/build/index.js:238:29)
[0-0]     at async Runner._initSession (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/@wdio/runner/build/index.js:204:25)
[0-0]     at async Runner.run (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/@wdio/runner/build/index.js:85:19)
[0-0] FAILED in wry - file:///test/specs/e2e.test.js
```

This is not reproducible locally. Even clearing the CI cache to ensure that the `yarn.lock` is respected with our new `--frozen-lockfile` settings doesn't help, nor does doing `yarn upgrade` help us reproduce the issue locally. We find that there is a more informative stack trace in the logs:

```
[0-0] 2024-02-09T02:56:58.452Z ERROR webdriver: Request failed with status 500 due to session not created: Failed to match capabilities
[0-0] 2024-02-09T02:56:58.454Z ERROR webdriver: session not created: Failed to match capabilities
[0-0]     at getErrorFromResponseBody (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/webdriverio/node_modules/webdriver/build/utils.js:195:12)
[0-0]     at NodeJSRequest._request (file:///home/runner/work/zamm-ui/zamm-ui/node_modules/webdriverio/node_modules/webdriver/build/request/index.js:193:23)
[0-0]     at processTicksAndRejections (node:internal/process/task_queues:95:5)
```

Based on this, we find our local file at `node_modules/webdriverio/node_modules/webdriver/build/request/index.js` and see that the relevant line is:

```js
        const error = getErrorFromResponseBody(response.body, fullRequestOptions.json);
```

It appears that the problem may be coming from the Tauri driver instead. Looking at the [latest versions](https://crates.io/crates/tauri-driver/versions) of `tauri-driver`, we see that version 0.1.4 was released just 5 days ago. We try locking the installed version to `0.1.3` to see if that fixes things. We edit the `tauri-driver` installation step in `.github/workflows/tests.yaml`, making sure to declare the newly pinned version at the top of the file for better visibility, even if it isn't used anywhere else:

```yaml
...

env:
  ...
  TAURI_DRIVER_VERSION: "0.1.3"

jobs:
  ...
  e2e:
    ...
    - name: Install tauri-driver
      uses: actions-rs/cargo@v1
      with:
        command: install
        args: tauri-driver@${{ env.TAURI_DRIVER_VERSION }}
    ...
```

The CI tests finally pass, and we inform the `tauri-driver` maintainers by creating a [new issue](https://github.com/tauri-apps/tauri/issues/8828) on their repo.

Incidentally, Cargo itself [does not](https://github.com/rust-lang/cargo/issues/7169#issuecomment-539226733) make use of the dependency lock by default. This does not appear to affect us at first, but eventually our CI builds fail with

```
Error: failed to compile `tauri-cli v1.5.9`, intermediate artifacts can be found at `/tmp/cargo-install82bABj`
Caused by:
  package `clap_complete v4.5.0` cannot be built because it requires rustc 1.74 or newer, while the currently active rustc version is 1.71.1
  Try re-running cargo install with `--locked`
```

We edit `.github/workflows/tests.yaml` yet again:

```yaml
...

env:
  ...
  TAURI_CLI_VERSION: "1.5.9"
  ...

jobs:
  ...
  e2e:
    ...
    steps:
    ...
      - name: Install tauri-cli
        ...
        with:
          ...
          args: --locked tauri-cli@${{ env.TAURI_CLI_VERSION }}
      - name: Install tauri-driver
        uses: actions-rs/cargo@v1
        with:
          command: install
          args: --locked tauri-driver@${{ env.TAURI_DRIVER_VERSION }}
    ...
```

### Sending only one message at a time

In the future, we'll want to emulate a "natural" conversation style where AIs will wait to allow you to send multiple messages at once, and perhaps only show the latest AI response if the user sends another one while the AI is "typing". However, for now we'll simply edit the send functionality to do nothing while the AI is "typing".

To do that, we first edit `src-svelte/src/lib/sample-call-testing.ts` to allow us to mock latency in promise resolution:

```ts
export class TauriInvokePlayback {
  ...
  callPauseMs?: number;

  ...

  mockCall(
    ...
  ): Promise<Record<string, string>> {
    ...
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        if (matchingCall.succeeded) {
          resolve(matchingCall.response);
        } else {
          reject(matchingCall.response);
        }
      }, this.callPauseMs || 0);
    });
  }

  ...
}
```

This causes several existing tests to fail. We fix them by editing how they wait for the response. For example, we edit `src-svelte/src/routes/components/Metadata.test.ts` from:

```ts
  test("linux system info returned", async () => {
    ...
    render(Metadata, {});
    await tickFor(3);
    expect(tauriInvokeMock).toHaveReturnedTimes(1);

    const shellRow = screen.getByRole("row", { name: /Shell/ });
    const shellValueCell = within(shellRow).getAllByRole("cell")[1];
    await waitFor(() => expect(shellValueCell).toHaveTextContent("Zsh"));
    expect(get(systemInfo)?.shell_init_file).toEqual("/root/.zshrc");
    ...
  });
```

to:

```ts
  test("linux system info returned", async () => {
    ...
    render(Metadata, {});
    await waitFor(() => {
      const shellRow = screen.getByRole("row", { name: /Shell/ });
      const shellValueCell = within(shellRow).getAllByRole("cell")[1];
      expect(shellValueCell).toHaveTextContent("Zsh");
      expect(get(systemInfo)?.shell_init_file).toEqual("/root/.zshrc");
    });

    expect(tauriInvokeMock).toHaveReturnedTimes(1);
    ...
  });
```

We do the same thing in `src-svelte/src/routes/AppLayout.test.ts`:

```ts
describe("AppLayout", () => {
  ...

  test("will set sound if sound preference overridden", async () => {
    ...
    render(AppLayout, { currentRoute: "/" });
    await waitFor(() => {
      expect(get(soundOn)).toBe(false);
    });
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
  });

  test("will set volume if volume preference overridden", async () => {
    ...
    await waitFor(() => {
      expect(get(volume)).toBe(0.8);
    });
    ...
  });

  test("will set animation if animation preference overridden", async () => {
    ...
    await waitFor(() => {
      expect(get(animationsOn)).toBe(false);
    });
    ...
  });

  test("will set animation speed if speed preference overridden", async () => {
    ...
    await waitFor(() => {
      expect(get(animationSpeed)).toBe(0.9);
    });
    ...
  });
});
```

Now we edit `src-svelte/src/routes/chat/Chat.svelte` to implement this:

```ts
  async function sendChatMessage(message: string) {
    if (expectingResponse) {
      return;
    }

    ...
  }
```

and we add a new test to `src-svelte/src/routes/chat/Chat.test.ts` where we make use of the new waiting functionality, making note of which parts we copied from `sendChatMessage` and which parts are different:

```ts
  test("won't send multiple messages at once", async () => {
    render(Chat, {});
    expect(tauriInvokeMock).not.toHaveBeenCalled();
    playback.callPauseMs = 1_000; // this line differs from sendChatMessage
    playback.addSamples(
      "../src-tauri/api/sample-calls/chat-start-conversation.yaml",
    );
    const nextExpectedApiCall: ParsedCall = playback.unmatchedCalls.slice(-1)[0];
    const nextExpectedCallArgs = nextExpectedApiCall.request[1] as Record<
      string,
      any
    >;
    const nextExpectedMessage = nextExpectedCallArgs["prompt"].slice(-1)[0] as ChatMessage;
    const nextExpectedHumanPrompt = nextExpectedMessage.text;

    const chatInput = screen.getByLabelText("Chat with the AI:");
    expect(chatInput).toHaveValue("");
    await userEvent.type(chatInput, "Hello, does this work?");
    await userEvent.click(screen.getByRole("button", { name: "Send" }));
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(screen.getByText(nextExpectedHumanPrompt)).toBeInTheDocument();

    // this part differs from sendChatMessage
    await userEvent.type(chatInput, "Tell me something funny.");
    await userEvent.click(screen.getByRole("button", { name: "Send" }));
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(screen.getByText(nextExpectedHumanPrompt)).toBeInTheDocument();
  });
```

We later "fix" this test without realizing its intended purpose, by having the second call actually go through. We revert it to its original purpose by increasing the delay, and also documenting its intended purpose for next time:

```ts
  test("won't send multiple messages at once", async () => {
    ...
    playback.callPauseMs = 2_000; // this line differs from sendChatMessage
    ...
    // this part differs from sendChatMessage
    await userEvent.type(chatInput, "Tell me something funny.");
    await userEvent.click(screen.getByRole("button", { name: "Send" }));
    // remember, we're intentionally delaying the first return here, so the mock won't be called
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    expect(screen.getByText(nextExpectedHumanPrompt)).toBeInTheDocument();
  });
```

We've actually noticed that the chat message gets lost (even if it doesn't get sent), so we edit `src-svelte\src\routes\chat\Chat.test.ts` to go further and make sure the user's chat message is preserved:

```ts
  async function sendChatMessage(
    ...
  ) {
    ...
    await waitFor(() => {
      expect(
        screen.getByText(
          ...,
        ),
      ).toBeInTheDocument();
    });
    expect(chatInput).toHaveValue("");
    ...
  }

  ...

  test("won't send multiple messages at once", async () => {
    ...
    // remember, we're intentionally delaying the first return here,
    // so the mock won't be called
    expect(tauriInvokeMock).toHaveBeenCalledTimes(1);
    ...
    expect(chatInput).toHaveValue("Tell me something funny.");
  });
```

We edit `src-svelte/src/routes/chat/Form.svelte` to not clear the form and trigger a send if the chat is currently busy:

```ts
  ...
  export let chatBusy = false;
  ...

  function submitChat() {
    const message = ...;
    if (message && !chatBusy) {
      ...
    }
  }
```

Then we pass the state of the chat down in `src-svelte\src\routes\chat\Chat.svelte`:

```svelte
    <Form
      {sendChatMessage}
      chatBusy={expectingResponse}
      ...
    />
```

### Widening the message bubble size

We edit `src-svelte\src\routes\chat\MessageUI.svelte` to change the default message size:

```css
  .message .text-container {
    ...
    max-width: 80%;
    ...
  }
```

This works, but then we realize that the text does not get resized when the window gets resized.

We try adding some more logic. There's a lot of jitter, so we try saving the timeout ID and clearing it if it hasn't fired yet. We get the error

```
Loading svelte-check in workspace: c:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-svelte
Getting Svelte diagnostics...

c:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-svelte\src\routes\chat\MessageUI.svelte:36:9
Error: Type 'Timeout' is not assignable to type 'number'. (ts)        
        }
        resizeTimeoutId = setTimeout(() => {
          if (textElement) {
```

We apply the solution [here](https://stackoverflow.com/a/56239226).

We eventually end up with editing `src-svelte\src\routes\chat\Chat.svelte` to store and inform the child elements of the overall chat width:

```svelte
<script lang="ts">
  ...
  import { writable } from 'svelte/store';

  ...
  let conversationWidthPx = writable(0);
  ...

  function resizeConversationView() {
    if (conversationView) {
      ...
      requestAnimationFrame(() => {
        if (conversationView && ...) {
          ...

          const conversationDimensions = conversationView.getBoundingClientRect();
          conversationWidthPx.set(conversationDimensions.width);
        }
      });
    }
  }

  ...
</script>

<InfoBox title="Chat" fullHeight>
  ...
          {#each conversation.slice(1) as message}
            <Message ... {conversationWidthPx} />
          {/each}
  ...
</InfoBox>
```

We then pass through all extra props in `src-svelte\src\routes\chat\Message.svelte`:

```svelte
...

<MessageUI ... {...$$restProps}>
  ...
</MessageUI>
```

In `src-svelte\src\routes\chat\MessageUI.svelte`, we make use of the new data available to us. What we want is:

1. If the chat window is 400px or less wide, the message bubbles should take up the entire space
2. Otherwise, the message bubbles should take up 80% of the space or 400px, whichever is greater
3. Message bubbles should be at most 600px wide no matter what
4. Message bubbles should resize correctly to be bigger if the window resizes to be wider
5. Because the above cannot be achieved with CSS alone, JavaScript will be needed to fix the width. The CSS should still follow the above as closely as possible to minimize flicker, and therefore the text container still needs to be `width: fit-content;` in order to avoid having flicker that extends across the entire width of the chat window.

```svelte
<script lang="ts">
  ...
  import type { Writable } from "svelte/store";

  ...
  export let conversationWidthPx: Writable<number> | undefined = undefined;
  ...

  let initialResizeTimeoutId: ReturnType<typeof setTimeout> | undefined;
  let finalResizeTimeoutId: ReturnType<typeof setTimeout> | undefined;

  // chat window size at which the message bubble should be full width
  const MIN_FULL_WIDTH_PX = 400;
  const MAX_WIDTH_PX = 600;

  function maxMessageWidth(chatWidthPx: number) {
    if (chatWidthPx <= MIN_FULL_WIDTH_PX) {
      return chatWidthPx;
    }

    const fractionalWidth = Math.max(0.8 * chatWidthPx, MIN_FULL_WIDTH_PX);
    return Math.min(fractionalWidth, MAX_WIDTH_PX);
  }

  function resizeBubble(chatWidthPx: number) {
    if (chatWidthPx > 0 && textElement) {
      try {
        textElement.style.width = "";

        const maxWidth = maxMessageWidth(chatWidthPx);
        const currentWidth = textElement.getBoundingClientRect().width;
        const newWidth = Math.min(currentWidth, maxWidth);
        textElement.style.width = `${newWidth}px`;

        if (finalResizeTimeoutId) {
          clearTimeout(finalResizeTimeoutId);
        }
        finalResizeTimeoutId = setTimeout(() => {
          if (textElement) {
            const range = document.createRange();
            range.selectNodeContents(textElement);
            const textRect = range.getBoundingClientRect();
            const actualTextWidth = textRect.width;

            const finalWidth = Math.min(actualTextWidth, newWidth);
            textElement.style.width = `${finalWidth}px`;
          }
        }, 10);
      } catch (err) {
        console.warn("Cannot resize chat message bubble: ", err);
      }
    }
  }

  onMount(() => {
    conversationWidthPx?.subscribe((chatWidthPx) => {
      if (initialResizeTimeoutId) {
        clearTimeout(initialResizeTimeoutId);
      }
      initialResizeTimeoutId = setTimeout(() => resizeBubble(chatWidthPx), 100);
    });
  });
</script>

...

<style>
  ...

  .message .text-container {
    ...
  }

  .text {
    ...
    max-width: 600px;
  }

  /* this takes sidebar width into account */
  @media (max-width: 635px) {
    .text {
      max-width: 400px;
    }
  }

  @media (min-width: 635px) {
    .message .text-container {
      max-width: calc(80% + 2.1rem);
    }
  }

  ...
</script>
```

Note that:

1. The width on the `textElement` needs to be reset to its natural state before we measure it again, or else the chat bubble will not grow when the window grows because it will be constrained to at most its previous size. This still works to shrink the message bubble, but not grow it.
2. `maxMessageWidth` doesn't need the early return, because the `currentWidth` will still be less than the `maxMessageWidth` and `Math.min` will ensure that the final max width does not exceed the chat window width. However, we might as well return the correct value still for correctness' sake, and to promote defense in depth against bugs.
3. The timeouts are there to avoid excessive flickering when resizing the window. Only when the user pauses for a sufficiently long time will the final resize be done.
4. We remove `max-width` from the regular `.message .text-container` to ensure that the message bubble takes up the entire space by default when the screen is small
5. The max-width CSS rule on the text always applies. If the chat window size is not big enough, it won't matter. If it's big enough to matter, this rule reduces flicker by already setting the max size.
6. We empirically test the screen size at which `maxMessageWidth` starts growing beyond 400px. This turns out to be a screen size of 635px. We use a media query to limit the max text width before this point to reduce screen flicker.
7. Once it starts growing beyond 400px, we want the message bubble to contain the entire text (so that it doesn't suddenly shrink once the screen grows big enough), but also to be small enough to reduce screen flicker. After adding in the margin and the padding for the message bubble, plus a small leeway, this comes out to 80% of the chat window width plus 2.1rem. No matter if we do it here or in the `maxMessageWidth` function, we'll need to do margin and padding calculations.

We add a new story at `src-svelte\src\routes\chat\Chat.stories.ts` to showcase this new full-width display:

```ts
export const FullMessageWidth: StoryObj = Template.bind({}) as any;
FullMessageWidth.args = {
  conversation,
  showMostRecentMessage: false,
};
FullMessageWidth.parameters = {
  viewport: {
    defaultViewport: "mobile1",
  },
};
```

As usual, we also mark this for screenshot testing at `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      ...,
      "full-message-width",
    ],
    screenshotEntireBody: true,
  },
  ...
];
```

A full test of all the requirements listed here (e.g. that chat message bubbles can grow after shrinking) will require enabling some interaction (e.g. resizing the window) in the Storybook tests, and will therefore be left as a TODO. Testing for flicker reduction would require either animation testing or else temporarily disabling the final resize timeout for the purposes of screenshoting the pre-flicker state; this will also be left as a TODO.

#### Fixing width

We notice that while this works fine in the browser, it fails on the app built from the publish workflow for the following interaction:

```json
[
  {"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},
  {"role":"Human","text":"Give me a code snippet a few lines long that I can use in a screenshot."},
  {"role":"AI","text":"Sure, here's a simple Python code snippet that you can use:\n\n```python\ndef greet(name):\n   print(\"Hello, \" + name + \"! How are you today?\")\n\nname = input(\"Enter your name: \")\ngreet(name)\n```"}
]
```

We edit `src-svelte/src/routes/chat/Chat.svelte` to start off with a demonstration:

```ts
  export let conversation: ChatMessage[] = [
    {
      role: "System",
      text: "You are ZAMM, a chat program. Respond in first person.",
    },
    {
      role: "Human",
      text: "Give me a code snippet a few lines long that I can use in a screenshot.",
    },
    {
      role: "AI",
      text: 'Sure, here\'s a simple Python code snippet that you can use:\n\n```python\ndef greet(name):\n   print("Hello, " + name + "! How are you today?")\n\nname = input("Enter your name: ")\ngreet(name)\n```',
    },
  ];
```

Nothing changes. We realize that this is because we have to instead edit `persistentConversation` in `src-svelte/src/routes/chat/PersistentChat.svelte` to pass the conversation to the `Chat` component. Once we do so, we find that we cannot reproduce this behavior.

Perhaps it is because the AppImage was bundled with an older version of Webkit. We try building from our Docker image. Sure enough, it is finally reproducible on this older Webkit.

Upon debugging by increasing the timeout to allow us to use the web inspector, we realize that this is because the text element width is being set to fractional values such as 489.667724609375. We edit `src-svelte/src/routes/chat/MessageUI.svelte` to always round the width up on `p` elements:

```ts
  function resizeChildren(...) {
    ...
    pElements.forEach((pElement) => {
      ...
      const actualTextWidth = Math.ceil(textRect.width);
      ...
    });

    ...
  }
```

If we do this, we might as well ensure that we're using integers everywhere else. We edit this further:

```ts

  function maxMessageWidth(chatWidthPx: number) {
    ...
    if (...) {
      return Math.ceil(...);
    }

    ...
    return Math.ceil(...);
  }
```

We also edit `resizeBubble` to rename `maxWidth` to `maxPotentialWidth` and `newWidth` to `maxActualWidth`, and then to pass in `maxActualWidth` to `resizeChildren`. This is because at every step, the width should never go up, only down.

```ts

  function resizeBubble(chatWidthPx: number) {
    ...
        const maxPotentialWidth = maxMessageWidth(chatWidthPx);
        const currentWidth = markdownElement.getBoundingClientRect().width;
        const maxActualWidth = Math.ceil(Math.min(currentWidth, maxPotentialWidth));
        ...

        ...
        finalResizeTimeoutId = setTimeout(() => {
          resizeChildren(markdownElement, maxActualWidth);
          ...
        }, 10);
    ...
  }
```

### Setting a cap on the chat input height

We'll try to use this chat message:

    Hey, I have this definition for a book object:

    ```python
    class Book:
      def __init__(self, title, author, pages):
          self.title = title
          self.author = author
          self.pages = pages

      def book_info(self):
          return f"'{self.title}' by {self.author} has {self.pages} pages."

      def is_long(self):
          return self.pages > 200
    ```

    Do you have any code comments for me?

We add this to `src-svelte/src/routes/chat/Chat.stories.ts`:

```ts
export const ExtraLongInput: StoryObj = Template.bind({}) as any;
ExtraLongInput.args = {
  conversation,
  initialMessage:
    `Hey, I have this definition for a book object:

\`\`\`python
class Book:
  def __init__(self, title, author, pages):
      self.title = title
      self.author = author
      self.pages = pages

  def book_info(self):
      return f"'{self.title}' by {self.author} has {self.pages} pages."

  def is_long(self):
      return self.pages > 200
\`\`\`

Do you have any code comments for me?`,
};
ExtraLongInput.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};
```

According to the [Autosize documentation](https://www.jacklmoore.com/autosize/), we set a maximum height by using CSS. As such, we edit the CSS in `src-svelte/src/routes/chat/Form.svelte`:

```css
  textarea {
    ...
    max-height: 9.8rem;
    ...
  }
```

Then, we register this as a new gold screenshot in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      ...,
      "extra-long-input",
      ...
    ],
    ...
  },
  ...
];
```

### Rendering markdown

We add a new dependency we found online:

```bash
$ yarn add svelte-markdown
```

Then we follow the instructions and edit our `src-svelte\src\routes\chat\Message.svelte` to incorporate Markdown formatting:

```svelte
<script lang="ts">
  ...
  import SvelteMarkdown from 'svelte-markdown'

  ...
</script>

<MessageUI ...>
  <div class="markdown">
    <SvelteMarkdown source={message.text} />
  </div>
</MessageUI>

<style>
  .markdown :global(:first-child) {
    margin-top: 0;
  }

  .markdown :global(:last-child) {
    margin-bottom: 0;
  }
</style>

```

We get rid of the margins for the first and last elements because they are superfluous when the chat message bubble itself already has internal padding.

We can now edit `src-svelte\src\routes\chat\MessageUI.svelte` to remove this CSS rule:

```css
  .message .text-container {
    ...
    white-space: pre-line;
    ...
  }
```

We add a new story to `src-svelte\src\routes\chat\Message.stories.ts`, based on the other ones already there:

```ts
export const Code: StoryObj = Template.bind({}) as any;
Code.args = {
  message: {
    role: "Human",
    text: "This is some Python code:\n\n" + 
      "```python\n" +
      "def hello_world():\n" +
      "    print('Hello, world!')\n" +
      "```\n\n" +
      "What do you think?",
  },
};
Code.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};
```

We register this new story in `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "message"],
    variants: [..., "code"],
  },
  ...
];
```

We notice that there's no syntax highlighting yet. We commit our changes first before trying to tackle that. However, `prettier` now fails with the error

```
yarn run v1.22.21
$ "C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\.bin\prettier" --write --plugin prettier-plugin-svelte src-svelte/src/routes/chat/Message.stories.ts src-svelte/package.json src-svelte/src/routes/chat/Message.svelte src-svelte/src/routes/chat/MessageUI.svelte
src-svelte\src\routes\chat\Message.stories.ts 285ms
src-svelte\package.json 54ms
{
    "type": "Script",
    "start": 0,
    "end": 295,
    "context": "default",
    "content": {
        "type": "Program",
        "start": 284,
        "end": 286,
        "loc": {
            "start": {
                "line": 1,
                "column": 0
            },
            "end": {
                "line": 1,
                "column": 286
            }
        },
        "body": [
            {
                "type": "BlockStatement",
                "start": 284,
                "end": 286,
                "loc": {
                    "start": {
                        "line": 1,
                        "column": 284
                    },
                    "end": {
                        "line": 1,
                        "column": 286
                    }
                },
                "body": []
            }
        ],
        "sourceType": "module"
    }
}

[error] src-svelte\src\routes\chat\Message.svelte: Error: unknown node type: Script
[error]     at Object.print (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier-plugin-svelte\plugin.js:1452:11)
[error]     at callPluginPrintFunction (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8601:26)
[error]     at mainPrintInternal (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8550:22)
[error]     at mainPrint (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8537:18)
[error]     at AstPath.call (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8359:24)
[error]     at printTopLevelParts (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier-plugin-svelte\plugin.js:1493:33)
[error]     at Object.print (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier-plugin-svelte\plugin.js:903:16)
[error]     at callPluginPrintFunction (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8601:26)
[error]     at mainPrintInternal (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8550:22)
[error]     at mainPrint (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\prettier\index.js:8537:18)
```

The only result online is [this issue](https://github.com/microsoft/parallel-prettier/issues/23), which doesn't immediately seem related because we're already using prettier v3.

Stashing our changes and running prettier again on all frontend files resulted in a lot of reformats. `yarn install --frozen-lockfile` and then running prettier again resulted in files largely being restored to the way they were before, with the sole exception of `sampleCall.ts`. Resetting the repo, applying the stashed changes again, and doing a second `yarn install --frozen-lockfile` appears to allow prettier to successfully reformat. We are now able to commit again. This appears to have been a fluke with the prettier plugin.

We see that Shiki is in the list of [supported extensions](https://marked.js.org/using_advanced#extensions). However, because of speed comparisons like [this](https://begin.com/blog/posts/2021-11-09-tale-of-the-tape-highlightjs-vs-shiki), we decide to try to use highlight.js. We do

```bash
$ yarn add svelte-highlight
```

We look at the [language list](https://github.com/metonym/svelte-highlight/blob/master/SUPPORTED_LANGUAGES.md) to see what our imports should be, [this example](https://www.npmjs.com/package/svelte-markdown#renderers) to see how our code should look, and the [README](https://github.com/metonym/svelte-highlight?tab=readme-ov-file#css-stylesheet) for styling options, and create `src-svelte\src\routes\chat\CodeRender.svelte`:

```svelte
<script lang="ts">
  import Highlight from "svelte-highlight";
  import bash from "svelte-highlight/languages/bash";
  import javascript from "svelte-highlight/languages/javascript";
  import typescript from "svelte-highlight/languages/typescript";
  import rust from "svelte-highlight/languages/rust";
  import python from "svelte-highlight/languages/python";
  import plaintext from "svelte-highlight/languages/plaintext";
  import "svelte-highlight/styles/github.css";

  export let text: string;
  export let lang: string;

  function getLanguageStr() {
    if (lang) {
      return lang.split(" ")[0];
    }
    return "plaintext";
  }

  function getLanguage() {
    let languageStr = getLanguageStr();
    switch (languageStr) {
      case "sh":
      case "bash":
        return bash;
      case "js":
      case "javascript":
        return javascript;
      case "typescript":
        return typescript;
      case "rust":
        return rust;
      case "py":
      case "python":
        return python;
      default:
        return plaintext;
    }
  }

  let language = getLanguage();
</script>

<div class="code">
  <Highlight {language} code={text} />
</div>

<style>
  .code :global(code) {
    border-radius: var(--corner-roundness);
    background-color: #ffffff88;
  }
</style>

```

Note that contrary to what [the documentation](https://marked.js.org/using_pro#renderer) appears to say about the render function signature being `code(string code, string infostring, boolean escaped)`, we discover empirically from `$: console.log($$props);` that the actual props being passed in are:

```json
{
  "raw": "```python\ndef hello_world():\n    print('Hello, world!')\n```",
  "lang": "python",
  "text": "def hello_world():\n    print('Hello, world!')"
}
```

Later on we add more languages, but we import HTML from `svelte-highlight/languages/xml` because it appears `svelte-highlight` does not export HTML as a language, even though HighlightJS itself supports it. We also lowercase the language string for consistency:

```ts
  ...
  import html from "svelte-highlight/languages/xml";
  import css from "svelte-highlight/languages/css";
  ...

  function getLanguageStr() {
    if (lang) {
      return lang...toLowerCase();
    }
    ...
  }
```

Now we just have to edit `src-svelte/src/routes/chat/Message.svelte` to use this new component:

```svelte
<script lang="ts">
  ...
  import CodeRender from "./CodeRender.svelte";
  ...
</script>

<MessageUI ...>
  <div ...>
    <SvelteMarkdown ... renderers={{ code: CodeRender }} />
  </div>
</MessageUI>
```

Fortunately, our screenshot tests are now alerting us to the fact that the code we'd previously written to resize the chat bubbles are no longer having their intended effect. As such, we update `src-svelte\src\routes\chat\MessageUI.svelte` to resize child elements of the new markdown div separately:

```svelte
<script lang="ts">
  ...
  const remPx = 18;
  // arrow size, left padding, and right padding
  const messagePaddingPx = (0.5 + 0.75 + 0.75) * remPx;
  ...

  function maxMessageWidth(chatWidthPx: number) {
    const availableWidthPx = chatWidthPx - messagePaddingPx;
    if (availableWidthPx <= MIN_FULL_WIDTH_PX) {
      return availableWidthPx;
    }

    const fractionalWidth = Math.max(0.8 * availableWidthPx, MIN_FULL_WIDTH_PX);
    return Math.min(fractionalWidth, MAX_WIDTH_PX);
  }

  function resetChildren(textElement: HTMLDivElement) {
    const pElements = textElement.querySelectorAll("p");
    pElements.forEach((pElement) => {
      pElement.style.width = "";
    });

    const codeElements = textElement.querySelectorAll<HTMLDivElement>(".code");
    codeElements.forEach((codeElement) => {
      codeElement.style.width = "";
    });
  }

  function resizeChildren(textElement: HTMLDivElement, maxWidth: number) {
    const pElements = textElement.querySelectorAll("p");
    pElements.forEach((pElement) => {
      const range = document.createRange();
      range.selectNodeContents(pElement);
      const textRect = range.getBoundingClientRect();
      const actualTextWidth = textRect.width;

      pElement.style.width = `${actualTextWidth}px`;
    });

    const codeElements = textElement.querySelectorAll<HTMLDivElement>(".code");
    codeElements.forEach((codeElement) => {
      let existingWidth = codeElement.getBoundingClientRect().width;
      if (existingWidth > maxWidth) {
        codeElement.style.width = `${maxWidth}px`;
      }
    });
  }

  function resizeBubble(chatWidthPx: number) {
    if (chatWidthPx > 0 && textElement) {
      try {
        const markdownElement = textElement.querySelector<HTMLDivElement>(".markdown");
        if (!markdownElement) {
          return;
        }

        resetChildren(markdownElement);

        const maxWidth = maxMessageWidth(chatWidthPx);
        const currentWidth = markdownElement.getBoundingClientRect().width;
        const newWidth = Math.ceil(Math.min(currentWidth, maxWidth));
        markdownElement.style.width = `${newWidth}px`;

        if (finalResizeTimeoutId) {
          clearTimeout(finalResizeTimeoutId);
        }
        finalResizeTimeoutId = setTimeout(() => {
          resizeChildren(markdownElement, maxWidth);
          markdownElement.style.width = "";
        }, 10);
      } catch (err) {
        ...
      }
    }
  }

  ...
</script>

...

<style>
  .message {
    ...
    --internal-spacing: 0.75rem;
    ...
  }

  .message .text-container {
    ...
    padding: var(--internal-spacing);
    ...
  }

  .text {
    ...
    width: fit-content;
    ...
  }

  ...
</style>
```

Note that our strategy is now:

1. Reset all child element widths to their natural state, so that they can grow again if the window got resized to be larger
2. Constrain markdown div to at most the maximum width. Note that we do have to take the `Math.min` so that excessive flicker does not occur if the div is already narrower than the maximum width. Otherwise, the div will be resized to its maximum size before shrinking back down.
3. After the browser has had a chance to rerender and wrap the `p` text, we resize the children again to fit their content. The `p` elements are resized to fit their text content, and the code elements are resized to fit the maximum width, if they exceed it. Later on, we will add the CSS necessary to produce a scroll behavior if the code elements exceed their maximum width.
4. We remove the width set on the markdown div so that it can now `fit-content` to the new size of its child elements, which should all already be constrained to the maximum width. Due to the re-wrapped text in the previous step, the markdown div may shrink in size again compared to step 2.

In `src-svelte\src\routes\chat\CodeRender.svelte`, we add scrollbars when code is overflowing the maximum width expected of it, and move the transparent background and rounded corners to the containing div so that those effects don't disappear when we scroll to the horizontal end of the code. We also set the vertical padding to be consistent with the rest of the message bubble:

```css
  .code {
    overflow-x: auto;
    box-sizing: border-box;
    border-radius: var(--corner-roundness);
    background-color: #ffffff88;
  }

  .code :global(code) {
    padding: var(--internal-spacing) 1rem;
    background-color: transparent;
  }

  .code, .code :global(pre), .code :global(code) {
    width: fit-content;
  }
```

where `--internal-spacing` is from the CSS variable defined above. We use this in `src-svelte\src\routes\chat\Message.svelte` as well for the `p` elements:

```css
  .markdown {
    width: fit-content;
  }

  .markdown :global(p) {
    margin: var(--internal-spacing) 0;
  }

  ...
```

Meanwhile, in `src-svelte\src\routes\chat\Chat.svelte`, we change the logic to only scroll to the bottom if it's the first mount. We don't want to change the user's scroll position on a resize. However, we must scroll to the bottom a second time shortly afterwards in case the internal element resizing produces a greater scroll:

```ts
  ...

  onMount(() => {
    resizeConversationView(true);
    const resizeCallback = () => resizeConversationView(false);
    window.addEventListener("resize", resizeCallback);
    ...

    return () => {
      window.removeEventListener("resize", resizeCallback);
      ...
    };
  });

  ...

  function resizeConversationView(initialMount: boolean = false) {
    if (...) {
      ...
      requestAnimationFrame(() => {
        if(...) {
          ...
          if (initialMount && showMostRecentMessage) {
            showChatBottom();
            // scroll to bottom again in case resized elements produce greater scroll
            setTimeout(showChatBottom, 120);
          }
        }
      });
    }
  }
```

Finally, we edit `src-svelte\src\routes\chat\Chat.stories.ts` to demonstrate the new code, and to fix the copy-paste for `FullMessageWidth` wherein `showMostRecentMessage` was erroneously set to false:

```ts
const conversation: ChatMessage[] = [
  ...,
  {
    role: "Human",
    text:
      "This is some Python code:\n\n" +
      "```python\n" +
      "def hello_world():\n" +
      "    print('Hello, world!')\n" +
      "```\n\n" +
      "Convert it to Rust",
  },
  {
    role: "AI",
    text: "Here's how the Python code you provided would look in Rust:\n\n"
      + "```rust\nfn main() {\n    println!(\"Hello, world!\");\n}\n```",
  },
];

...

export const FullMessageWidth: StoryObj = ...;
FullMessageWidth.args = {
  conversation,
};
...
```

We realize while testing on CI that the monospace font used is inconsistent across platforms. We change this by adding Inconsolata, because at first we don't like the look of JetBrains Mono for this particular use case:

```bash
$ yarn add @fontsource/inconsolata
```

We import it in `src-svelte\src\routes\styles.css`:

```css
...
@import "@fontsource/inconsolata";
...
```

and reference it in `src-svelte\src\routes\chat\CodeRender.svelte`:

```css
  .code :global(code) {
    ...
    font-family: "Inconsolata", monospace;
  }
```

However, we later decide we do prefer the look of JetBrains Mono after all, so we undo the previous steps and instead put

```css
  .code :global(code) {
    ...
    font-family: var(--font-mono);
    font-size: 0.8rem;
  }
```

where we decrease the font size to match the rest of the text we have.

Next, we add an inset effect to the code blocks in `src-svelte\src\routes\chat\MessageUI.svelte`. We add it here instead of `CodeRender.svelte` because we wish to produce differently colored shadow effects based on whether it's a human or AI message, and the information about that only exists at this level.

```svelte
<script lang="ts">
  ...
  export let forceHighlight = false;
  ...
</script>

<div ... class:force-highlight={forceHighlight} ...>
  ...
</div>

<style>
  ...

  .message :global(.code) {
    transition: box-shadow var(--standard-duration);
  }
  
  .message.human :global(.code) {
    box-shadow:
      inset 0.02rem 0.02rem 0.3rem 0.1rem rgba(0, 102, 0, 0.1);
  }

  .message.human :global(.code:hover), .message.human.force-highlight :global(.code) {
    box-shadow:
      inset 0.02rem 0.02rem 0.3rem 0.1rem rgba(0, 102, 0, 0.2),
      0.02rem 0.02rem 0.3rem 0.1rem rgba(0, 102, 0, 0.05);
  }

  .message.ai :global(.code) {
    box-shadow:
      inset 0.02rem 0.02rem 0.3rem 0.1rem rgba(0, 0, 102, 0.1);
  }

  .message.ai :global(.code:hover), .message.ai.force-highlight :global(.code) {
    box-shadow:
      inset 0.02rem 0.02rem 0.3rem 0.1rem rgba(0, 0, 102, 0.2),
      0.02rem 0.02rem 0.3rem 0.1rem rgba(0, 0, 102, 0.05);
  }

  ...
</style>
```

We add new stories to `src-svelte\src\routes\chat\Message.stories.ts` to demonstrate the code blocks for both AI and human messages:

```ts
let humanCodeMessage = {
  role: "Human",
  text:
    "This is some Python code:\n\n" +
    "```python\n" +
    "def hello_world():\n" +
    "    print('Hello, world!')\n" +
    "```\n\n" +
    "Convert it to Rust",
};

let aiCodeMessage = {
  role: "AI",
  text:
    "Here's how the Python code you provided would look in Rust:\n\n" +
    '```rust\nfn main() {\n    println!("Hello, world!");\n}\n```',
};

export const HumanCode: StoryObj = Template.bind({}) as any;
HumanCode.args = {
  message: humanCodeMessage,
};
HumanCode.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

export const HighlightedHumanCode: StoryObj = Template.bind({}) as any;
HighlightedHumanCode.args = {
  message: humanCodeMessage,
  forceHighlight: true,
};
HighlightedHumanCode.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

export const AICode: StoryObj = Template.bind({}) as any;
AICode.args = {
  message: aiCodeMessage,
};
AICode.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

export const HighlightedAiCode: StoryObj = Template.bind({}) as any;
HighlightedAiCode.args = {
  message: aiCodeMessage,
  forceHighlight: true,
};
HighlightedAiCode.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

```

We register these new snapshots in `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "message"],
    variants: [..., "human-code", "highlighted-human-code", "ai-code", "highlighted-ai-code"],
  },
  ...
];
```

### Removing top and bottom margins from first and last messages

We notice that there is more whitespace than we'd like to see, because of the margins on the first and last messages sent. We edit `src-svelte\src\routes\chat\MessageUI.svelte` accordingly:

```css
  .message:first-child .text-container {
    margin-top: 0;
  }

  .message:last-child .text-container {
    margin-bottom: 0;
  }
```

### Getting the "Send" chat button to be at full height

We realize that on Linux, the "Send" button isn't reaching the full height of the chat input. We [find out](https://stackoverflow.com/a/32466333) that this is because we're supposed to use `align-items` instead of `height` for that. We edit `src-svelte/src/routes/chat/Form.svelte` to change `align-items` from `center` to `stretch`:

```css
  form {
    align-items: stretch;
  }
```

We also remove `height: 100%;` from this `src-svelte/src/lib/controls/Button.svelte` selector:

```css
  .outer.right-end,
  .inner.right-end {
    ...
  }
```

Unfortunately, upon closer inspection, it turns out that the input text is no longer vertically centered in `src-svelte/src/routes/chat/Form.svelte`. As such, we try to make the vertical margin "auto" instead:

```css
  textarea {
    margin: auto 0.75rem;
    ...
  }
```

### Using an icon for the send symbol

We'd like to try using an icon for the send symbol. However, we must first edit `src-svelte/src/lib/controls/Button.svelte` to take in a slot instead of a prop for the content. We remove the `text` prop:

```svelte
<script lang="ts">
  ...
</script>

{#if unwrapped}
  <button
    ...
  >
    <slot />
  </button>
{:else}
  <button
    ...
  >
    <div class="cut-corners ..." ...>
      <slot />
    </div>
  </button>
{/if}

```

We edit files such as `src-svelte/src/routes/api-calls/[slug]/Actions.svelte` next:

```svelte
<InfoBox title="Actions" ...>
  <div ...>
    <Button ...>Restore conversation</Button>
  </div>
</InfoBox>
```

As it turns out, the button screenshot test is now failing because the Storybook story can't pass in any args. This is likely why the component was made with args in the first place. We create `src-svelte/src/lib/controls/ButtonView.svelte`:

```svelte
<script lang="ts">
  import Button from "./Button.svelte";

  export let text: string;
</script>

<Button>{text}</Button>

```

and edit `src-svelte/src/lib/controls/Button.stories.ts` to import from `ButtonView.svelte` instead of `Button.svelte`:

```ts
import Button from "./ButtonView.svelte";
...
```

Now, we check the existing dimensions of the "Send" button. They're at 73.58 px wide by 35 px tall. We convert this into 4 rem by 2 rem, and edit `src-svelte/src/routes/chat/Form.svelte` accordingly:

```svelte
<script lang="ts">
  ...
  import IconSend from "~icons/gravity-ui/arrow-right";
  ...
</script>

<form
  ...
>
  ...
  <Button ...><IconSend /></Button>
</form>

<style>
  ...

  form :global(button) {
    width: 4rem;
    min-height: 2rem;
    display: flex;
    justify-content: center;
    align-items: center;
  }
</style>
```

This now looks much more in line with the sleek retrofuturistic look of most of the rest of the UI. We add a new entry to `src-svelte/src/routes/credits/Credits.svelte`:

```svelte
              <Creditor
                name="Gravity UI"
                logo="gravity-ui.png"
                url="https://gravity-ui.com/"
              />
```

and download the logo to `src-svelte/static/logos/gravity-ui.png`. Unfortunately, this logo is completely square and doesn't fit in well with the existing credit images. As such, we edit `src-svelte/src/routes/credits/Creditor.svelte` to move the `border-radius: var(--corner-roundness);` from `img.person` to the generic `img`. This produces ever so slight changes in the screenshot for the credits page, but they are not really perceptible to the naked eye.

We find that the test `src-svelte/src/routes/chat/Chat.test.ts` is failing now. We edit `src-svelte/src/lib/controls/Button.svelte` to add ARIA labeling:

```svelte
<script lang="ts">
  ...
  export let ariaLabel: string | undefined = undefined;
  ...
</script>

{#if unwrapped}
  <button
    ...
    aria-label={ariaLabel}
    ...
  >
    ...
  </button>
{:else}
  <button
    ...
    aria-label={ariaLabel}
    ...
  >
    ...
  </button>
{/if}
```

We make use of this option in `src-svelte/src/routes/chat/Form.svelte`:

```svelte
<form ...>
  ...
  <Button ariaLabel="Send" ...><IconSend /></Button>
</form>
```

and now the test passes. We also have to edit `src-svelte/src/routes/storybook.test.ts` to use a new way of clicking on the Send button:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      ...,
      {
        name: "new-message-sent",
        prefix: "extra-long-input",
        additionalAction: async (frame: Frame) => {
          await frame.click('button[aria-label="Send"]');
          ...
        },
      },
    ],
    ...
  },
  ...
];
```

## Adding a type field for variant API calls

In case we ever add incompatible LLM calls (for example, to non-chat models), we'll make use of Serde's [enum representations](https://serde.rs/enum-representations.html) in order to add a `type` field to our API calls. We first edit the models in `src-tauri/src/models/llm_calls.rs`:

```rs
...

#[derive(
    Debug,
    Clone,
    Serialize,
    Deserialize,
    PartialEq,
    specta::Type,
)]
pub struct ChatPrompt {
    pub prompt: Vec<ChatMessage>,
}

#[derive(
    Debug,
    Clone,
    Serialize,
    Deserialize,
    PartialEq,
    AsExpression,
    FromSqlRow,
    specta::Type,
)]
#[diesel(sql_type = Text)]
#[serde(tag = "type")]
pub enum Prompt {
    Chat(ChatPrompt),
}

...

impl ToSql<Text, Sqlite> for Prompt
...

impl<DB> FromSql<Text, DB> for Prompt
...

pub struct Request {
    #[serde(flatten)]
    pub prompt: Prompt,
    ...
}

...

pub struct LlmCallRow {
    ...,
    pub prompt: Prompt,
    ...
}

pub struct NewLlmCallRow<'a> {
    ...,
    pub prompt: &'a Prompt,
    ...
}
```

We realize after seeing the compilation error

```
error[E0599]: no method named `len` found for enum `models::llm_calls::Prompt` in the current scope
    --> src/commands/llms/chat.rs:281:42
     |
281  |         assert!(ok_result.request.prompt.len() > 0);
     |                                          ^^^ method not found in `Prompt`
     |
    ::: src/models/llm_calls.rs:211:1
     |
211  | pub enum Prompt {
     | --------------- method `len` not found for this enum
     |
```

that it might be easier after all right now to just keep `ChatPrompt` in the request and only convert it when we need an `LlmCallRow`. Unfortunately, if we do that and change

```rs
impl LlmCall {
    pub fn as_sql_row(&self) -> NewLlmCallRow {
        let prompt = Prompt::Chat(self.request.prompt.clone());
        NewLlmCallRow {
            ...,
            prompt,
            ...
        }
    }
}
```

we get

```
error[E0308]: mismatched types
   --> src/models/llm_calls.rs:351:13
    |
351 |             prompt,
    |             ^^^^^^ expected `&Prompt`, found `Prompt`
    |
```

We have to construct a `Prompt::Chat`, but then we can't return a reference to it. We go back to the other way.

We edit `src-tauri/src/commands/llms/chat.rs`:

```rs
    async fn test_llm_api_call(recording_path: &str, sample_path: &str) {
        ...
        // check that it made it into the database
        let stored_llm_call = ...
        ...

        // do a sanity check that everything is non-empty
        let prompt = match ok_result.request.prompt {
            Prompt::Chat(ChatPrompt { prompt }) => prompt,
        };
        assert!(prompt.len() > 0);
        match &ok_result.response.completion {
            ChatMessage::AI { text } => assert!(!text.is_empty()),
            _ => panic!("Unexpected response type"),
        }
    }
```

Now we get

```
error[E0277]: the trait bound `models::llm_calls::Prompt: specta::Flatten` is not satisfied
   --> src/models/llm_calls.rs:279:17
    |
279 |     pub prompt: Prompt,
    |                 ^^^^^^ the trait `specta::Flatten` is not implemented for `models::llm_calls::Prompt`
    |
    = help: the following other types implement trait `specta::Flatten`:
              BTreeMap<K, V>
              ChatMessage
              HashMap<K, V>
              Llm
              SystemTime
              TokenMetadata
              models::llm_calls::ChatPrompt
              models::llm_calls::EntityId
            and 8 others
note: required by a bound in `models::llm_calls::_::<impl NamedType for models::llm_calls::Request>::named_data_type::validate_flatten`
   --> src/models/llm_calls.rs:276:48
    |
276 | #[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
    |                                                ^^^^^^^^^^^^ required by this bound in `validate_flatten`
    = note: this error originates in the derive macro `specta::Type` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
```

No Google results come up, but we realize this is because of `#[serde(flatten)]` on the field. We remove that.

It compiles, but tests fail. We now get

```
---- commands::llms::chat::tests::test_continue_conversation stdout ----
thread 'commands::llms::chat::tests::test_continue_conversation' panicked at 'called `Result::unwrap()` on an `Err` value: Error("invalid type: map, expected variant identifier", line: 12, column: 6)', src/commands/llms/chat.rs:204:44
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

---- commands::llms::chat::tests::test_start_conversation stdout ----
thread 'commands::llms::chat::tests::test_start_conversation' panicked at 'called `Result::unwrap()` on an `Err` value: Error("invalid type: map, expected variant identifier", line: 11, column: 6)', src/commands/llms/chat.rs:204:44
```

After inspecting `src-svelte/src/lib/bindings.ts`, we rename the field

```rs
pub struct ChatPrompt {
    pub messages: Vec<ChatMessage>,
}
```

because that field is no longer being flattened. We edit the transcripts like `src-tauri/api/sample-calls/chat-continue-conversation.yaml` to conform:

```yaml
...
response:
  ...
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
        ...
      }
```

Nothing should change on the frontend because the frontend only makes use of completions. We do a manual test and check the database:

```bash
$ sqlite3 /root/.local/share/zamm/zamm.sqlite3
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> select * from llm_calls;
41ce000b-2095-43bf-857c-423cbeda24d4|2024-02-14 09:38:24.046338350|open_ai|gpt-4|gpt-4-0613|1.0|27|9|36|{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"hi"}]}|{"role":"AI","text":"Hello! How can I assist you today?"}
```

Sure enough, our new `"type": "Chat"` field is present. We try to commit, but we get hit with a surprising new message from pre-commit that has never appeared before and that we certainly didn't affect just now:

```
warning: variant name ends with the enum's name
 --> src/commands/system.rs:9:5
  |
9 |     MacOS,
  |     ^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#enum_variant_names
  = note: `-D clippy::enum-variant-names` implied by `-D warnings`
```

We change it as requested, and realize for the first time that it was displayed as "MacOS" without a space in the first place. We change the frontend at `src-svelte/src/routes/components/Metadata.svelte`:

```svelte
<script lang="ts">
  ...
  import { getSystemInfo, type OS } from "$lib/bindings";
  ...
  
  function formatOsString(os: OS | null | undefined) {
    if (os === "Mac") {
      return "Mac OS"
    }
    return os ?? "Unknown";
  }

  $: os = formatOsString($systemInfo?.os);
</script>

<div class="container">
  ...
        <tr>
          <td>OS</td>
          <td>{os}</td>
        </tr>
  ...
</div>
```

## Refactoring out scroll behavior

We want to replicate the scroll behavior in other parts of the app, so we'll refactor it out into a new component. We create `src-svelte/src/lib/FixedScrollable.svelte` based on existing code:

```svelte
<script lang="ts">
  import { onMount } from "svelte";

  export let maxHeight: string;
  export let initialPosition: "top" | "bottom" = "top";
  let scrollContents: HTMLDivElement | undefined = undefined;
  let topIndicator: HTMLDivElement;
  let bottomIndicator: HTMLDivElement;
  let topShadow: HTMLDivElement;
  let bottomShadow: HTMLDivElement;

  export function getDimensions() {
    if (scrollContents) {
      return scrollContents.getBoundingClientRect();
    }

    throw new Error("Scrollable component not mounted");
  }

  export function scrollToBottom() {
    if (scrollContents) {
      scrollContents.scrollTop = scrollContents.scrollHeight;
    }
  }

  function intersectionCallback(shadow: HTMLDivElement) {
    return (entries: IntersectionObserverEntry[]) => {
      let indicator = entries[0];
      if (indicator.isIntersecting) {
        shadow.classList.remove("visible");
      } else {
        shadow.classList.add("visible");
      }
    };
  }

  onMount(() => {
    let topScrollObserver = new IntersectionObserver(
      intersectionCallback(topShadow),
    );
    topScrollObserver.observe(topIndicator);
    let bottomScrollObserver = new IntersectionObserver(
      intersectionCallback(bottomShadow),
    );
    bottomScrollObserver.observe(bottomIndicator);

    if (initialPosition === "bottom") {
      scrollToBottom();
    }

    return () => {
      topScrollObserver.disconnect();
      bottomScrollObserver.disconnect();
    };
  });

  $: style = `max-height: ${maxHeight}`;
</script>

<div class="scrollable composite-reveal" {style}>
  <div class="shadow top" bind:this={topShadow}></div>
  <div
    class="scroll-contents composite-reveal"
    {style}
    bind:this={scrollContents}
  >
    <div class="indicator top" bind:this={topIndicator}></div>
    <slot />
    <div class="indicator bottom" bind:this={bottomIndicator}></div>
  </div>
  <div class="shadow bottom" bind:this={bottomShadow}></div>
</div>

<style>
  .scrollable {
    position: relative;
  }

  .scrollable :global(.shadow.visible) {
    display: block;
  }

  .scroll-contents {
    overflow-y: auto;
  }

  .shadow {
    z-index: 1;
    height: 0.375rem;
    width: 100%;
    position: absolute;
    display: none;
  }

  .shadow.top {
    top: 0;
    background-image: radial-gradient(
      farthest-side at 50% 0%,
      rgba(150, 150, 150, 0.4) 0%,
      rgba(0, 0, 0, 0) 100%
    );
  }

  .shadow.bottom {
    bottom: 0;
    background-image: radial-gradient(
      farthest-side at 50% 100%,
      rgba(150, 150, 150, 0.4) 0%,
      rgba(0, 0, 0, 0) 100%
    );
  }

  .indicator {
    height: 1px;
    width: 100%;
  }

  .indicator.top {
    margin-bottom: -1px;
  }

  .indicator.bottom {
    margin-top: -1px;
  }
</style>

```

This component will be responsible for maintaining scroll behavior, initial scroll offset, and showing shadows at the top and bottom of the scrollable area, which will have a fixed height. Note the export of component functions in non-module scope, which depends on [the upgrade](/general-notes/coding/frameworks/sveltekit.md) (in section "Upgrading to SvelteKit 2") to SvelteKit 2.

Because this takes a slot, we create a view for this in `src-svelte/src/lib/FixedScrollableView.svelte` before we create a Storybook story for it, using content generated from a [lorem ipsum generator](https://www.lipsum.com/):

```svelte
<script lang="ts">
  import FixedScrollable from "./FixedScrollable.svelte";
</script>

<FixedScrollable maxHeight="10rem" {...$$restProps}>
  <p>
    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Quisque faucibus
    vulputate rhoncus. Maecenas velit urna, consectetur id ipsum ac, porta
    accumsan neque. Vestibulum sit amet magna aliquet, posuere lorem eu, aliquet
    sapien. Duis ac arcu a lacus dictum hendrerit. Integer vel lorem dapibus,
    interdum erat eget, iaculis sem. Mauris massa turpis, vulputate ut iaculis
    nec, aliquet id massa. Aenean nunc tortor, viverra sed auctor ut, viverra
    vel dolor. Duis molestie maximus felis id lacinia. Pellentesque venenatis
    diam sit amet posuere tincidunt. In hac habitasse platea dictumst.
    Pellentesque in mattis ipsum. Donec tempus mi eu varius egestas. Nulla
    mattis metus lectus, eget auctor risus aliquam vitae. Aenean ullamcorper
    elementum enim, vitae viverra mi dignissim sit amet. Cras semper enim eget
    sapien gravida, eget laoreet elit lobortis. Nam dictum, dolor sit amet
    rhoncus tincidunt, quam lorem faucibus nunc, sed convallis orci sapien
    dignissim nisi.
  </p>

  <p>
    Donec eu hendrerit mauris, hendrerit iaculis nulla. Nam venenatis volutpat
    eleifend. Donec quis dolor at arcu commodo aliquet. Duis feugiat nisi in ex
    viverra, eu facilisis ante tempus. Cras sed porta ipsum. Donec dui diam,
    ultrices eget neque feugiat, malesuada cursus arcu. In eget laoreet ligula,
    eget finibus quam. Duis congue porta libero, vitae porttitor erat.
    Suspendisse ipsum dui, condimentum vitae dictum sed, feugiat non leo.
    Integer tristique sollicitudin ullamcorper. Nullam elementum facilisis nulla
    vitae congue. In cursus est commodo ante mollis pharetra.
  </p>

  <p>
    Integer at nunc suscipit sem molestie facilisis. Morbi iaculis bibendum dui,
    quis convallis mauris bibendum ut. Suspendisse arcu nunc, tristique sit amet
    nisi ut, varius lacinia orci. Nulla tincidunt orci id tellus ultricies, eu
    vestibulum neque tincidunt. Nam at nisi justo. Etiam lacinia aliquet sem,
    sit amet facilisis erat vestibulum quis. Proin ut semper turpis, vel euismod
    nibh. Ut in tempor risus. Vivamus ullamcorper molestie laoreet. Maecenas id
    fermentum tellus. Curabitur dui leo, porta id tortor non, dignissim vehicula
    lectus. Nulla et diam at metus accumsan varius sit amet id sapien.
  </p>

  <p>
    Pellentesque lacinia enim ut commodo cursus. Nullam rhoncus pretium ornare.
    Donec pulvinar urna vel tempor accumsan. Nunc viverra purus non lacus
    condimentum, ut sodales orci placerat. Vivamus luctus dictum nisi. Donec
    turpis diam, dictum eget ex non, vulputate molestie dui. Duis urna lectus,
    tempor in bibendum venenatis, congue eu tortor. Duis at dictum sapien,
    bibendum porta magna. Quisque nisi eros, vulputate nec dui ut, scelerisque
    rhoncus ante. In varius metus mi, in elementum lorem venenatis lacinia.
    Proin risus lacus, ultrices eget dolor sed, sodales ullamcorper orci.
    Vivamus non augue pulvinar, congue orci nec, imperdiet neque. Cras ligula
    nisl, pharetra vel libero vitae, efficitur lacinia nunc. Maecenas facilisis
    tortor a arcu egestas commodo.
  </p>

  <p>
    Nunc ac pellentesque ex, sit amet posuere purus. In ac tempor libero, vel
    pretium metus. Nam fermentum ut arcu vehicula fermentum. Mauris euismod
    magna vitae ipsum aliquam tristique. Duis luctus arcu dictum nibh egestas,
    eu fringilla augue ultricies. Fusce in augue consequat, luctus nisl vitae,
    sollicitudin mauris. In ultrices augue sit amet nisi luctus imperdiet. In
    non felis euismod, viverra metus id, hendrerit leo. Curabitur tristique erat
    et nisl mattis, eu tempus nisl tristique. Praesent vel sem pellentesque
    libero sodales lobortis nec a neque. Mauris at lorem auctor, tincidunt augue
    nec, lobortis sem.
  </p>
</FixedScrollable>

```

We add a story for this in `src-svelte/src/lib/FixedScrollable.stories.ts`:

```ts
import FixedScrollableComponent from "./FixedScrollableView.svelte";
import type { StoryObj } from "@storybook/svelte";

export default {
  component: FixedScrollableComponent,
  title: "Reusable/Scrollable/Fixed",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: FixedScrollableComponent,
  props: args,
});

export const Top: StoryObj = Template.bind({}) as any;
Top.args = {
  initialPosition: "top",
};

export const Bottom: StoryObj = Template.bind({}) as any;
Bottom.args = {
  initialPosition: "bottom",
};

```

As usual, we register the new component screenshots at `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["reusable", "scrollable", "fixed"],
    variants: ["top", "bottom"],
  },
  ...
];
```

However, we see from the new bottom screenshot that the component does not scroll all the way to the bottom. It is not clear why this only happens on Webkit, but we find that we can reproduce it if the details pane is open on initial page load. As such, we edit `src-svelte/src/lib/FixedScrollable.svelte`:

```ts
  onMount(() => {
    ...

    if (initialPosition === "bottom") {
      scrollToBottom();

      // hack for Storybook screenshot test on Webkit
      setTimeout(() => scrollToBottom(), 10);
    }

    ...
  });
```

Now for our main Scrollable component, we create `src-svelte/src/lib/Scrollable.svelte`:

```svelte
<script lang="ts" context="module">
  export type ResizedEvent = CustomEvent<DOMRect>;
</script>

<script lang="ts">
  import { onMount } from "svelte";
  import { createEventDispatcher } from "svelte";
  import FixedScrollable from "./FixedScrollable.svelte";

  export let minHeight = "8rem";
  export let initialPosition: "top" | "bottom" = "bottom";
  let scrollableHeight: string = minHeight;
  let container: HTMLDivElement | null = null;
  let scrollable: FixedScrollable | null = null;
  const dispatchResizeEvent = createEventDispatcher();

  export function resizeScrollable() {
    if (!scrollable) {
      return;
    }

    scrollableHeight = minHeight;
    requestAnimationFrame(() => {
      if (!container || !scrollable) {
        console.warn("Container or scrollable not mounted");
        return;
      }

      scrollableHeight = `${container.clientHeight}px`;
      dispatchResizeEvent("resize", scrollable.getDimensions());
    });
  }

  export function scrollToBottom() {
    scrollable?.scrollToBottom();
  }

  onMount(() => {
    resizeScrollable();
    const windowResizeCallback = () => resizeScrollable();
    window.addEventListener("resize", windowResizeCallback);

    return () => {
      window.removeEventListener("resize", windowResizeCallback);
    };
  });
</script>

<div class="growable container composite-reveal" bind:this={container}>
  <FixedScrollable
    {initialPosition}
    maxHeight={scrollableHeight}
    bind:this={scrollable}
  >
    <slot />
  </FixedScrollable>
</div>

<style>
  .container {
    flex-grow: 1;
  }
</style>

```

This one is responsible for resizing the fixed-size scrollable element whenever the window gets resized. It emits a resize event if it does. We see from [this answer](https://stackoverflow.com/a/67951078) how to correctly type the resize event in Svelte.

Because this component also operates on a slot, we create a view in `src-svelte/src/lib/ScrollableView.svelte`:

```svelte
<script lang="ts">
  import Scrollable from "./Scrollable.svelte";
</script>

<div class="container">
  <Scrollable {...$$restProps}>
    <p>
      Lorem ipsum dolor sit amet, consectetur adipiscing elit. Quisque faucibus
      vulputate rhoncus. Maecenas velit urna, consectetur id ipsum ac, porta
      accumsan neque. Vestibulum sit amet magna aliquet, posuere lorem eu,
      aliquet sapien. Duis ac arcu a lacus dictum hendrerit. Integer vel lorem
      dapibus, interdum erat eget, iaculis sem. Mauris massa turpis, vulputate
      ut iaculis nec, aliquet id massa. Aenean nunc tortor, viverra sed auctor
      ut, viverra vel dolor. Duis molestie maximus felis id lacinia.
      Pellentesque venenatis diam sit amet posuere tincidunt. In hac habitasse
      platea dictumst. Pellentesque in mattis ipsum. Donec tempus mi eu varius
      egestas. Nulla mattis metus lectus, eget auctor risus aliquam vitae.
      Aenean ullamcorper elementum enim, vitae viverra mi dignissim sit amet.
      Cras semper enim eget sapien gravida, eget laoreet elit lobortis. Nam
      dictum, dolor sit amet rhoncus tincidunt, quam lorem faucibus nunc, sed
      convallis orci sapien dignissim nisi.
    </p>

    <p>
      Donec eu hendrerit mauris, hendrerit iaculis nulla. Nam venenatis volutpat
      eleifend. Donec quis dolor at arcu commodo aliquet. Duis feugiat nisi in
      ex viverra, eu facilisis ante tempus. Cras sed porta ipsum. Donec dui
      diam, ultrices eget neque feugiat, malesuada cursus arcu. In eget laoreet
      ligula, eget finibus quam. Duis congue porta libero, vitae porttitor erat.
      Suspendisse ipsum dui, condimentum vitae dictum sed, feugiat non leo.
      Integer tristique sollicitudin ullamcorper. Nullam elementum facilisis
      nulla vitae congue. In cursus est commodo ante mollis pharetra.
    </p>

    <p>
      Integer at nunc suscipit sem molestie facilisis. Morbi iaculis bibendum
      dui, quis convallis mauris bibendum ut. Suspendisse arcu nunc, tristique
      sit amet nisi ut, varius lacinia orci. Nulla tincidunt orci id tellus
      ultricies, eu vestibulum neque tincidunt. Nam at nisi justo. Etiam lacinia
      aliquet sem, sit amet facilisis erat vestibulum quis. Proin ut semper
      turpis, vel euismod nibh. Ut in tempor risus. Vivamus ullamcorper molestie
      laoreet. Maecenas id fermentum tellus. Curabitur dui leo, porta id tortor
      non, dignissim vehicula lectus. Nulla et diam at metus accumsan varius sit
      amet id sapien.
    </p>

    <p>
      Pellentesque lacinia enim ut commodo cursus. Nullam rhoncus pretium
      ornare. Donec pulvinar urna vel tempor accumsan. Nunc viverra purus non
      lacus condimentum, ut sodales orci placerat. Vivamus luctus dictum nisi.
      Donec turpis diam, dictum eget ex non, vulputate molestie dui. Duis urna
      lectus, tempor in bibendum venenatis, congue eu tortor. Duis at dictum
      sapien, bibendum porta magna. Quisque nisi eros, vulputate nec dui ut,
      scelerisque rhoncus ante. In varius metus mi, in elementum lorem venenatis
      lacinia. Proin risus lacus, ultrices eget dolor sed, sodales ullamcorper
      orci. Vivamus non augue pulvinar, congue orci nec, imperdiet neque. Cras
      ligula nisl, pharetra vel libero vitae, efficitur lacinia nunc. Maecenas
      facilisis tortor a arcu egestas commodo.
    </p>

    <p>
      Nunc ac pellentesque ex, sit amet posuere purus. In ac tempor libero, vel
      pretium metus. Nam fermentum ut arcu vehicula fermentum. Mauris euismod
      magna vitae ipsum aliquam tristique. Duis luctus arcu dictum nibh egestas,
      eu fringilla augue ultricies. Fusce in augue consequat, luctus nisl vitae,
      sollicitudin mauris. In ultrices augue sit amet nisi luctus imperdiet. In
      non felis euismod, viverra metus id, hendrerit leo. Curabitur tristique
      erat et nisl mattis, eu tempus nisl tristique. Praesent vel sem
      pellentesque libero sodales lobortis nec a neque. Mauris at lorem auctor,
      tincidunt augue nec, lobortis sem.
    </p>
  </Scrollable>
  <div class="other-element">
    <p>
      This element takes up fixed space. The Scrollable above should expand to fill the rest of the space.
    </p>
  </div>
</div>

<style>
  .container {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    height: calc(100vh - 2rem);
  }

  .other-element {
    background-color: #f0f0f0;
    padding: 1rem;
  }
</style>

```

and add stories to match at `src-svelte/src/lib/Scrollable.stories.ts`:

```ts
import ScrollableComponent from "./ScrollableView.svelte";
import type { StoryObj } from "@storybook/svelte";

export default {
  component: ScrollableComponent,
  title: "Reusable/Scrollable/Growable",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: ScrollableComponent,
  props: args,
});

export const Small: StoryObj = Template.bind({}) as any;
Small.args = {
  initialPosition: "top",
};
Small.parameters = {
  viewport: {
    defaultViewport: "mobile1",
  },
};

export const Large: StoryObj = Template.bind({}) as any;
Large.args = {
  initialPosition: "top",
};
Large.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};

```

As usual, we register the screenshots in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["reusable", "scrollable", "growable"],
    variants: ["small", "large"],
  },
  ...
];
```

We manually test that window resizing works consistently here. From repeated runs of the automate tests, we see that there is actually some non-determinism in how the larger screenshot is rendered. However, the non-determinism is consistent, so we can at least add it as a variant.

Next, we edit `src-svelte/src/routes/chat/MessageUI.svelte` to no longer listen to `conversationWidthPx`, as store event updates add another layer of complexity. Instead, we remove that as a prop, remove `initialResizeTimeoutId`, export `resizeBubble` to be controlled entirely by the parent, and add a proper cleanup method to ensure the timeout is taken care of on dismount:

```ts
  export function resizeBubble(...) {
    ...
  }

  onMount(() => {
    return () => {
      if (finalResizeTimeoutId) {
        clearTimeout(finalResizeTimeoutId);
      }
    };
  });
```

Next, we re-export `resizeBubble` in the parent component `src-svelte/src/routes/chat/Message.svelte`:

```svelte
<script lang="ts">
  ...
  let resizeBubbleBound: (chatWidthPx: number) => void;

  export function resizeBubble(chatWidthPx: number) {
    resizeBubbleBound(chatWidthPx);
  }
</script>

<MessageUI
  ...
  bind:resizeBubble={resizeBubbleBound}
  ...
>
  ...
</MessageUI>

...
```

We do it this way because it's unclear how to do a re-export directly. Here, we are using the more [recommended way](https://stackoverflow.com/a/61355264) of binding a child component's function directly to a parent variable.

Finally, we make use of all this in `src-svelte/src/routes/chat/Chat.svelte`. We rip out all the logic that is now part of our refactored components, and replace it with:

```svelte
<script lang="ts">
  ...
  import Scrollable, {
    type ResizedEvent,
  } from "$lib/Scrollable.svelte";
  ...

  let initialMount = true;
  let messageComponents: Message[] = [];
  let growable: Scrollable | undefined;
  let conversationWidthPx = 100;

  function resizeConversationView() {
    growable?.resizeScrollable();
  }

  function onScrollableResized(e: ResizedEvent) {
    conversationWidthPx = e.detail.width;
    messageComponents.forEach((message) => {
      message.resizeBubble(e.detail.width);
    });
    if (initialMount && showMostRecentMessage) {
      growable?.scrollToBottom();
    }
  }

  function appendMessage(message: ChatMessage) {
    conversation = [...conversation, message];
    setTimeout(() => {
      const latestMessage = messageComponents[messageComponents.length - 1];
      latestMessage.resizeBubble(conversationWidthPx);
      growable?.scrollToBottom();
    }, 10);
  }

  async function sendChatMessage(message: string) {
    ...

    const chatMessage: ChatMessage = {
      ...
    };
    appendMessage(chatMessage);
    ...

    try {
      let llmCall = await chat(...);
      appendMessage(llmCall.response.completion);
    } ...
  }

  onMount(() => {
    setTimeout(() => {
      // hack: Storybook window resize doesn't cause remount
      initialMount = false;
    }, 1_000);
  });
</script>

<InfoBox ...>
  <div ...>
    <Scrollable
      initialPosition={showMostRecentMessage ? "bottom" : "top"}
      on:resize={onScrollableResized}
      bind:this={growable}
    >
      <div class="composite-reveal" role="list">
        {#if conversation.length > 1}
          {#each conversation.slice(1) as message, i (i)}
            <Message
              {message}
              {conversationWidthPx}
              bind:this={messageComponents[i]}
            />
          {/each}
        {/if}
      </div>
    </Scrollable>

    ...
  </div>
</InfoBox>
```

Note that:

1. We create the function `appendMessage` to handle all scroll-to-bottom behavior after every new message
2. We add a unique id to each rendered conversation message in order to prevent any unexpected mounts. Since the conversation array is append-only for now, we can use the index as the id.
3. The initial resize for the message should already produce all the final word wrapping. The second resize is only there to reduce the padding, and shouldn't affect the word wrapping (and by extension, shouldn't affect the rendered height of the elements). As such, we can just scroll to the bottom as soon as the initial resize is done, without waiting for a callback from the second resize to finish.
4. The `initialMount` is set to true for the entire first second because otherwise the Storybook story doesn't scroll to the bottom on WebKit

Our screenshot tests are highly useful in bolstering our confidence that all these changes still keep the chat UI looking as expected. We manually test that window resize events still behave as expected.

We fix up `src-svelte/src/routes/chat/Chat.stories.ts` after realizing that the component has been named `Chatcomponent` with a lowercase c, so we rename it to `ChatComponent`. We also remove the first `Empty.parameters`, because it gets overridden by the second one anyways. These don't actually change the tests, so there is no need to update any screenshots.

### Testing with interactions

During development, we noticed a regression around the scrollable chat div failing to resize when a multi-line message was sent. We should ideally get this behavior automatically tested. As such, we edit `src-svelte\src\routes\storybook.test.ts`:

```ts
import {
 ...,
  type Frame,
} from "@playwright/test";
...

interface ComponentTestConfig {
  ...
  variants: (string | VariantConfig)[];
  ...
}

interface VariantConfig {
  name: string;
  prefix?: string;
  ...
  additionalAction?: (frame: Frame) => Promise<void>;
}

const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      ...,
      {
        name: "new-message-sent",
        prefix: "extra-long-input",
        additionalAction: async (frame: Frame) => {
          await frame.click('button:has-text("Send")');
          await frame.click('button[title="Dismiss"]');
        },
      },
    ],
    screenshotEntireBody: true,
  },
  ...
];

...

describe.concurrent("Storybook visual tests", () => {
  ...

  const takeScreenshot = async (frame: Frame, ...) => {
    ...
  };

  ...

  for (const config of components) {
    ...

      test<StorybookTestContext>(
        `${testName} should render the same`,
        async ({ expect, page }) => {
          const variantPrefixStr = variantConfig.prefix ?? variantConfig.name;
          const variantPrefix = `--${variantPrefixStr}`;
          
          ...

          const frame = page.frame({ name: "storybook-preview-iframe" });
          if (!frame) {
            throw new Error("Could not find Storybook iframe");
          }

          if (variantConfig.additionalAction) {
            await variantConfig.additionalAction(frame);
          }

          const screenshot = await takeScreenshot(
            frame,
            ...
          );

          ...
        },
        ...
      );
  }
});
```

Note that:

1. The `ComponentTestConfig.variants` should actually be the more flexible `(string | VariantConfig)[]` type rather than the `string[] | VariantConfig[]` type we specified at first
2. We add a `prefix` variable to the variant config so that we don't have to create a new Storybook story just to test out behavior
3. Because the newly defined variant config function will want to act on the Storybook frame, we refactor the frame locator outside of the `takeScreenshot` function

#### Adding keyboard events

The above behavior with `initialMount` is still flaky on CI. As such, we rip the hacky logic out of `src-svelte\src\routes\chat\Chat.svelte`, and add the test-specific logic into `src-svelte\src\routes\storybook.test.ts`:

```ts
...

interface VariantConfig {
  ...
  additionalAction?: (frame: Frame, page: Page) => Promise<void>;
}

const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "conversation"],
    variants: [
      "empty",
      "not-empty",
      "multiline-chat",
      "extra-long-input",
      "bottom-scroll-indicator",
      "typing-indicator-static",
      {
        name: "full-message-width",
        additionalAction: async (frame: Frame, page: Page) => {
          await new Promise((r) => setTimeout(r, 1000));
          // need to do a manual scroll because Storybook resize messes things up on CI
          const scrollContents = frame.locator(".scroll-contents");
          await scrollContents.focus();
          await page.keyboard.press("End");
        },
      },
      ...
  },
  ...
];

...
      test<StorybookTestContext>(
        `${testName} should render the same`,
        async ({ expect, page }) => {
          ...
          if (variantConfig.additionalAction) {
            await variantConfig.additionalAction(frame, page);
          }
          ...
        },
        ...
      );
...
```

### Windows WebKit non-determinism

We notice while testing on Windows that the screenshot tests occasionally non-deterministically fail due to the reported size somehow being smaller than it actually is rendered on screen. This only happens on the initial Storybook page load; if the component is re-mounted, the effect does not appear. This may well be a fluke specific to the combination of WebKit 17.4, Storybook, and Windows, but this does prevent us from further development on Windows. As such, we will have to fix the problem.

We add a zero wait to `src-svelte\src\routes\chat\MessageUI.svelte` right before doing the width calculations:

```svelte
  export async function resizeBubble(chatWidthPx: number) {
    if (chatWidthPx > 0 && textElement) {
      try {
        ...
        await new Promise(r => setTimeout(r, 0));
        const maxPotentialWidth = maxMessageWidth(chatWidthPx);
        ...
      } ...
    }
  }
```

Since this is not a "real" phenomenon, but rather a test-related fluke, it seems fine to set a timeout of 0 for the wait.

We propagate this new signature up to `src-svelte\src\routes\chat\Message.svelte`:

```ts
  let resizeBubbleBound: (chatWidthPx: number) => Promise<void>;

  export function resizeBubble(chatWidthPx: number) {
    return resizeBubbleBound(chatWidthPx);
  }
```

Finally, we await all these new promises in `src-svelte\src\routes\chat\Chat.svelte`:

```ts
  async function onScrollableResized(e: ResizedEvent) {
    ...
    const resizePromises = messageComponents.map((message) => message.resizeBubble(...));
    await Promise.all(resizePromises);
    ...
  }

  function appendMessage(message: ChatMessage) {
    ...
    setTimeout(async () => {
      ...
      await latestMessage.resizeBubble(conversationWidthPx);
      ...
    }, 10);
  }
```

Note that because `Message` no longer relies on `conversationWidthPx`, we remove that from the props.

### Getting tests on CI to pass

Non-screenshot tests are failing:

```
TypeError: Cannot read properties of null (reading 'resizeBubble')
 ❯ Timeout._onTimeout src/routes/chat/Chat.svelte:44:27
     42|     setTimeout(async () => {
     43|       const latestMessage = messageComponents[messageComponents.length…
     44|       await latestMessage.resizeBubble(conversationWidthPx);
       |                           ^
     45|       growable?.scrollToBottom();
     46|     }, 10);
 ❯ listOnTimeout node:internal/timers:569:17
 ❯ processTimers node:internal/timers:512:7

This error originated in "src/routes/chat/Chat.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
The latest test that might've caused the error is "won't send multiple messages at once". It might mean one of the following:
- The error was thrown, while Vitest was running this test.
- If the error occurred after the test had been completed, this was the last documented test before it was thrown.
```

We edit `src-svelte\src\routes\chat\Chat.svelte` accordingly:

```ts
  function appendMessage(message: ChatMessage) {
    ...
    setTimeout(async () => {
      const latestMessage = ...;
      await latestMessage?.resizeBubble(conversationWidthPx);
      ...
    }, ...);
  }
```

#### Failing switch and slider tests

Some switch and slider tests are failing on CI with the message:

```
 ❯ src/lib/Switch.playwright.test.ts  (5 tests | 5 failed) 17469ms
   ❯ src/lib/Switch.playwright.test.ts > Switch drag test > switches state when drag released at end
     → expect(received).toHaveAttribute()

received value must be an HTMLElement or an SVGElement.

     → expect(received).toHaveAttribute()

received value must be an HTMLElement or an SVGElement.

     → expect(received).toHaveAttribute()

received value must be an HTMLElement or an SVGElement.
```

Because this test is flaky and hard to reproduce locally, we try simply running it again on CI. Unfortunately (or fortunately) it fails consistently on CI. This is surprising given that it hasn't failed before, and appears to be yet another example of bit rot.

We try to fix this by defining the function

```ts
const expectAriaValue = async (slider: Locator, expectedValue: number) => {
    const sliderValueStr = await slider.evaluate((el) =>
      el.getAttribute("aria-valuenow"),
    );
    expect(sliderValueStr).not.toBeNull();
    if (!sliderValueStr) {
      // just for type-checking
      throw new Error("aria-valuenow attribute was null");
    }
    const sliderValue = parseFloat(sliderValueStr);
    expect(sliderValue).toEqual(expectedValue);
  };
```

and refactoring all our tests to use this new function. This appears to solve the issue for the slider tests, so we apply it to the switch tests too. We edit `src-svelte\src\lib\Switch.playwright.test.ts` along with all the tests:

```ts
  const expectAriaValue = async (switchElement: Locator, expectedValue: boolean) => {
    const switchValueStr = await switchElement.evaluate((el) =>
      el.getAttribute("aria-checked"),
    );
    expect(switchValueStr).not.toBeNull();
    expect(switchValueStr).toEqual(expectedValue.toString());
  };

  test(
    "switches state when drag released at end",
    async () => {
      ...
      await expectAriaValue(onOffSwitch, false);
      ...
      await expectAriaValue(onOffSwitch, true);
      ...
    },
    ...
  );

  ...
```

Note that we name the argument `switchElement` instead of `switch` because "switch" is a reserved keyword, and naming it that results in arcane parsing errors such as "Argument expression expected" or "Cannot find name 'async'".

## Fixing filled conversation reveal

After we make it possible to populate the chat view with a pre-existing conversation via the API call restoration button, we find that if that's the first time the chat view is loaded, the InfoBox reveal does not happen properly. While debugging, we find that some Storybook stories are now also inadequate for our debugging. We edit `src-svelte/src/lib/__mocks__/MockPageTransitions.svelte` so that mocked page transition stories don't produce a scrollbar:

```css
  :global(.storybook-wrapper.full-height) {
    margin: -1rem;
  }
```

We export the conversations and move the `FullPage` story out of `src-svelte/src/routes/chat/Chat.stories.ts`:

```ts
export const shortConversation: ChatMessage[] = [
  ...
];

export const conversation: ChatMessage[] = [
  ...
];
```

We create `src-svelte/src/routes/chat/Chat.reveal.stories.ts`:

```ts
import ChatComponent from "./Chat.svelte";
import SvelteStoresDecorator from "$lib/__mocks__/stores";
import MockPageTransitions from "$lib/__mocks__/MockPageTransitions.svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";
import type { StoryFn, StoryObj } from "@storybook/svelte";
import { conversation } from "./Chat.stories";

export default {
  component: ChatComponent,
  title: "Screens/Chat/Conversation",
  argTypes: {},
  decorators: [
    SvelteStoresDecorator,
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
  Component: ChatComponent,
  props: args,
});

export const FullPage: StoryObj = Template.bind({}) as any;

export const FullPageConversation: StoryObj = Template.bind({}) as any;
FullPageConversation.parameters = {
  stores: {
    conversation,
  },
};

```

Somehow, the existing Storybook tests look wrong now. To help debug, we edit `src-svelte/src/lib/__mocks__/MockAppLayout.svelte` to include an ID:

```svelte
<div id="mock-app-layout" class="storybook-wrapper">
  ...
</div>
```

We do the same for `src-svelte/src/lib/__mocks__/MockFullPageLayout.svelte`:

```svelte
<div id="mock-full-page-layout" class="storybook-wrapper ...">
  ...
</div>
```

And we do the same for `src-svelte/src/routes/AnimationControl.svelte`:

```svelte
<div
  id="animation-control"
  ...
>
  ...
</div>

<style>
  #animation-control {
    ...
  }

  #animation-control.animations-disabled :global(*) {
    ...
  }
</style>

```

This gives us failing tests:

```
 FAIL  src/routes/AnimationControl.test.ts > AnimationControl > will enable animations by default
TypeError: Cannot read properties of null (reading 'classList')
 ❯ src/routes/AnimationControl.test.ts:17:29
     15| 
     16|     const animationControl = document.querySelector(".container") as E…
     17|     expect(animationControl.classList).not.toContainEqual(
       |                             ^
     18|       "animations-disabled",
     19|     );
```

As such, we edit `src-svelte/src/routes/AnimationControl.test.ts` to have

```ts
  const animationControl = document.getElementById("animation-control") as Element;
```

instead in each test.

Next, it turns out all exports from `src-svelte/src/routes/chat/Chat.stories.ts`, even the non-story exports, get listed as a story in Storybook. So, we create `src-svelte/src/routes/chat/Chat.mock-data.ts`:

```ts
import type { ChatMessage } from "$lib/bindings";

export const shortConversation: ChatMessage[] = [
  ...
];

export const conversation: ChatMessage[] = [
  ...
];

```

and import them in `src-svelte/src/routes/chat/Chat.stories.ts`:

```ts
import { conversation, shortConversation } from "./Chat.mock-data";
```

and ditto in `src-svelte/src/routes/chat/Chat.reveal.stories.ts`.

Finally, it turns out that the global style we applied in `src-svelte/src/lib/__mocks__/MockPageTransitions.svelte` is interfering with other tests. So we edit that file:

```svelte
<div id="mock-transitions">
  <MockFullPageLayout>
    ...
  </MockFullPageLayout>
</div>

<style>
  #mock-transitions {
    margin: -1rem;
  }
</style>

```

Now we can finally consistently reproduce the problem. We start solving it by first editing `src-svelte/src/lib/InfoBox.svelte` to make the problem less severe, for now and for if it ever crops up again in the future:

```ts
  function revealInfoBox(node: Element, timing: InfoBoxTiming) {
    ...

    const getChildKickoffFraction = (...) => {
      ...
      const delayFraction = Math.max(
        0,
        adjustedYProgress * theoreticalTotalKickoffFraction,
      );
      ...
    };

    ...
  }
```

We edit `src-svelte/src/routes/chat/Chat.svelte`, following the advice [here](https://dev.to/mohamadharith/workaround-for-bubbling-custom-events-in-svelte-3khk) to use our own custom event dispatcher:

```svelte
<script lang="ts">
  ...
  let chatContainer: HTMLDivElement | undefined = undefined;
  ...

  async function onScrollableResized(e: ResizedEvent) {
    ...
    if (!chatContainer) {
      console.warn("Chat container not initialized");
      return;
    }
    chatContainer.dispatchEvent(
      new CustomEvent("info-box-update", { bubbles: true }),
    );
  }

  ...
</script>

<InfoBox title="Chat" fullHeight>
  <div
    class="chat-container ..."
    bind:this={chatContainer}
  >
    ...
  </div>
</InfoBox>
```

Note that we can't reuse the existing `growable` variable in this component because it gets bound as a `Scrollable`, and therefore doesn't have the `dispatchEvent` function. If we try it, we get

```
TypeError: growable.dispatchEvent is not a function
```

Now edit `src-svelte/src/lib/InfoBox.svelte` again to now listen for and act on this event, refactoring the original MutationObserver callback function into a `recalculateReveals` function that can be reused for the new event:

```ts
  function revealInfoBox(node: Element, timing: InfoBoxTiming) {
    ...
    const recalculateReveals = () => {
      revealAnimations = getNodeAnimations(node);
      ...
    };
    const mutationObserver = new MutationObserver(recalculateReveals);
    mutationObserver.observe(node, config);
    node.addEventListener("info-box-update", recalculateReveals);

    return {
      ...,
      tick: (tGlobalFraction: number) => {
        ...

        if (tGlobalFraction === 1) {
          mutationObserver.disconnect();
          node.removeEventListener("info-box-update", recalculateReveals);
        }
      },
    };
  }
```

We leave as a TODO moving this code to the `Scrollable` component, to fire whenever a resize happens.

## Recording continuation of API calls

We want to record instances where one API call is a continuation of another because it is part of the same conversation chain. We create a new Diesel migration:

```bash
$ diesel migration generate add_api_call_continuations
Creating migrations/2024-05-13-135101_add_api_call_continuations/up.sql
Creating migrations/2024-05-13-135101_add_api_call_continuations/down.sql
```

We define `src-tauri/migrations/2024-05-13-135101_add_api_call_continuations/up.sql` as:

```sql
CREATE TABLE llm_call_continuations (
  previous_call_id VARCHAR NOT NULL,
  next_call_id VARCHAR NOT NULL,
  PRIMARY KEY (previous_call_id, next_call_id),
  FOREIGN KEY (previous_call_id) REFERENCES llm_calls (id) ON DELETE CASCADE,
  FOREIGN KEY (next_call_id) REFERENCES llm_calls (id) ON DELETE CASCADE
)

```

and `src-tauri/migrations/2024-05-13-135101_add_api_call_continuations/down.sql` as:

```sql
DROP TABLE llm_call_continuations

```

Then we do

```bash
$ diesel migration run --database-url /root/.local/share/zamm/zamm.sqlite3
```

`src-tauri/src/schema.rs` gets automatically updated for us with this new table:

```rs
diesel::table! {
    llm_call_continuations (previous_call_id, next_call_id) {
        previous_call_id -> Text,
        next_call_id -> Text,
    }
}
```

We wish to edit `src-tauri/src/models/llm_calls.rs` to add these fields to the higher-level `LlmCall` struct:

```rs
#[derive(...)]
pub struct LlmCall {
    ...,
    pub previous_call_id: Option<EntityId>,
    pub next_call_id: Option<EntityId>,
}
```

But this means that we must now do some joins to populate these new fields. We look at the documentation on aliasing tables in Diesel [here](https://docs.rs/diesel/latest/diesel/macro.alias.html), and try editing `src-tauri/src/commands/llms/get_api_call.rs`:

```rs
...
use crate::models::llm_calls::{..., LlmCallJoinQueryRow};
use crate::schema::{..., llm_call_continuations};
...

async fn get_api_call_helper(
    ...
) -> ZammResult<LlmCall> {
    ...

    let (previous_calls, next_calls) = diesel::alias!(
        llm_call_continuations as previous_call,
        llm_call_continuations as next_call
    );

    let result: LlmCallJoinQueryRow = llm_calls::table
        .left_join(
            previous_calls.on(
                llm_calls::id.eq(
                    previous_calls.field(llm_call_continuations::next_call_id),
                ),
            ),
        )
        .left_join(
            next_calls.on(
                llm_calls::id.eq(
                    next_calls.field(llm_call_continuations::previous_call_id),
                ),
            ),
        )
        .select((
            llm_calls::all_columns,
            previous_calls.fields(llm_call_continuations::previous_call_id).nullable(),
            next_calls.fields(llm_call_continuations::previous_call_id).nullable(),
        ))
        .filter(llm_calls::id.eq(parsed_uuid))
        .first::<LlmCallJoinQueryRow>(conn)?;
    Ok(result.into())
}

...
```

where `src-tauri/src/models/llm_calls.rs` in turn has been further edited as such:

```rs
pub type LlmCallJoinQueryRow = (LlmCallRow, Option<EntityId>, Option<EntityId>);

impl From<LlmCallJoinQueryRow> for LlmCall {
    fn from(joined_row: LlmCallJoinQueryRow) -> Self {
        let (llmCallRow, previous_call_id, next_call_id) = joined_row;

        let id = llmCallRow.id;
        let timestamp = llmCallRow.timestamp;
        ...
        LlmCall {
            ...
            previous_call_id,
            next_call_id,
        }
    }
}

```

We then edit `src-tauri/src/commands/llms/chat.rs`:

```rs
...
use crate::schema::{llm_calls, llm_call_continuations};
...
use diesel::prelude::*;

async fn chat_helper(
    ...,
    prompt: ...,
    previous_call_id: Option<Uuid>,
    http_client: ...,
) -> ZammResult<LlmCall> {
    ...
    let llm_call = LlmCall {
        ...,
        previous_call_id: previous_call_id.map(|id| EntityId { uuid: id }),
        next_call_id: None,
    };

    if let Some(conn) = db.as_mut() {
        diesel::insert_into(llm_calls::table)
            ...;
        
        if let Some(previous_call_uuid) = llm_call.previous_call_id.as_ref() {
            diesel::insert_into(llm_call_continuations::table)
                .values((
                    llm_call_continuations::dsl::previous_call_id.eq(previous_call_uuid),
                    llm_call_continuations::dsl::next_call_id.eq(llm_call.id.clone()),
                ))
                .execute(conn)?;
        }
    } // todo: warn users if DB write unsuccessful

    ...
}

#[tauri::command(async)]
#[specta]
pub async fn chat(
    ...
    prompt: ...,
    previous_call_id: Option<Uuid>,
) -> ZammResult<LlmCall> {
    ...
    chat_helper(
        ...,
        previous_call_id,
        ...,
    )
    .await
}
```

"is not an iterator" errors come up if we fail to use the `dsl` and `prelude` imports, which can be seen in the comment thread [here](https://lemmy.technosorcery.net/post/8718).

We realize that this is actually a one-to-many relationship, and so we need to change the `next_call_id` field to be `Vec<EntityId>` in `src-tauri/src/models/llm_calls.rs`. We decide to wrap these fields in a conversation metadata struct:

```rs
#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct ConversationMetadata {
    pub previous_call_id: Option<EntityId>,
    pub next_call_ids: Vec<EntityId>,
}

#[derive(...)]
pub struct LlmCall {
    ...,
    pub conversation: ConversationMetadata,
}

...

pub type LlmCallLeftJoinResult = (LlmCallRow, Option<EntityId>);
pub type LlmCallQueryResults = (LlmCallLeftJoinResult, Vec<EntityId>);

impl From<LlmCallQueryResults> for LlmCall {
    fn from(query_results: LlmCallQueryResults) -> Self {
        let ((llmCallRow, previous_call_id), next_call_ids) = query_results;
        ...
        let conversation_metadata = ConversationMetadata {
            previous_call_id,
            next_call_ids,
        };
        LlmCall {
            ...,
            conversation: conversation_metadata,
        }
    }
}
```

Now, we can edit `src-tauri/src/commands/llms/get_api_call.rs` again, where we don't even need the table aliasing anymore:

```rs
...
use crate::models::llm_calls::{..., LlmCallLeftJoinResult};
...

async fn get_api_call_helper(
    ...
) -> ZammResult<LlmCall> {
    ...

    let previous_call_result: LlmCallLeftJoinResult = llm_calls::table
        .left_join(
            llm_call_continuations::dsl::llm_call_continuations.on(
                llm_calls::id.eq(
                    llm_call_continuations::next_call_id,
                ),
            ),
        )
        .select((
            llm_calls::all_columns,
            llm_call_continuations::previous_call_id.nullable(),
        ))
        .filter(llm_calls::id.eq(&parsed_uuid))
        .first::<LlmCallLeftJoinResult>(conn)?;
    let next_calls_result = llm_call_continuations::table
        .select(llm_call_continuations::next_call_id)
        .filter(llm_call_continuations::previous_call_id.eq(parsed_uuid))
        .load::<EntityId>(conn)?;
    Ok((previous_call_result, next_calls_result).into())
}
```

`src-tauri/src/commands/llms/chat.rs` is largely the same:

```rs
async fn chat_helper(
    ...
) -> ZammResult<LlmCall> {
    ...
    let conversation_metadata = ConversationMetadata {
        previous_call_id: previous_call_id.map(|id| EntityId { uuid: id }),
        next_call_ids: [].to_vec(),
    };
    ...
    let llm_call = LlmCall {
      ...,
      conversation: conversation_metadata,
    };

    if let Some(conn) = db.as_mut() {
        ...
        
        if let Some(previous_call_uuid) = llm_call.conversation.previous_call_id.as_ref() {
            diesel::insert_into(llm_call_continuations::table)
                ...;
        }
    } // todo: warn users if DB write unsuccessful

    Ok(llm_call)
}
}
```

Next, we have to edit `src-tauri/src/test_helpers/database_contents.rs` and `src-tauri/src/commands/llms/get_api_calls.rs`. The latter will require a change to provide a list of condensed API call data, only containing information necessary for the table view. But doing so will require changes to that API. We commit the current changes, and do that refactor first in a different branch.

Before doing that refactor, we realize that the `src-tauri/src/models/llm_calls.rs` file is already bloated, so we refactor out its components into a module folder with a `src-tauri\src\models\llm_calls\mod.rs` that looks like this:

```rs
mod chat_message;
mod entity_id;
mod llm_call;
mod prompt;
mod row;
mod various;

pub use chat_message::ChatMessage;
pub use entity_id::EntityId;
pub use llm_call::LlmCall;
pub use prompt::{ChatPrompt, Prompt};
#[allow(unused_imports)]
pub use row::{LlmCallRow, NewLlmCallRow};
pub use various::{Llm, Request, Response, TokenMetadata};
```

The `unused_imports` refers to `NewLlmCallRow`, which right now is only explicitly referenced in test code.

We then create our lightweight LLM call at `src-tauri\src\models\llm_calls\lightweight_llm_call.rs`:

```rs
use crate::models::llm_calls::chat_message::ChatMessage;
use crate::models::llm_calls::entity_id::EntityId;
use crate::models::llm_calls::llm_call::LlmCall;
use crate::models::llm_calls::row::LlmCallRow;
use chrono::naive::NaiveDateTime;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct LightweightLlmCall {
    pub id: EntityId,
    pub timestamp: NaiveDateTime,
    pub response_message: ChatMessage,
}

impl From<LlmCall> for LightweightLlmCall {
    fn from(value: LlmCall) -> Self {
        LightweightLlmCall {
            id: value.id,
            timestamp: value.timestamp,
            response_message: value.response.completion,
        }
    }
}

impl From<LlmCallRow> for LightweightLlmCall {
    fn from(value: LlmCallRow) -> Self {
        LightweightLlmCall {
            id: value.id,
            timestamp: value.timestamp,
            response_message: value.completion,
        }
    }
}

```

and then expose this in `src-tauri\src\models\llm_calls\mod.rs`:

```rs
...
mod lightweight_llm_call;
...

...
pub use lightweight_llm_call::LightweightLlmCall;
...
```

We can now make use of this in `src-tauri\src\commands\llms\get_api_calls.rs`:

```rs
...
use crate::models::llm_calls::{LightweightLlmCall, LlmCallRow};
...

async fn get_api_calls_helper(
    ...
) -> ZammResult<Vec<LightweightLlmCall>> {
    ...
    let result: Vec<LlmCallRow> = llm_calls::table
        ...;
    let calls: Vec<LightweightLlmCall> =
        result.into_iter().map(|row| row.into()).collect();
    Ok(calls)
}

...
```

Everything else is similarly a straightforward find-and-replace of `LlmCall` to `LightweightLlmCall`, including the tests in that file.

We edit `src-tauri\api\sample-database-writes\many-api-calls\generate.py`:

```py
def generate_api_call_json(i: int) -> str:
    return """      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a{0:02d}",
        "timestamp": "2024-01-16T08:{0:02d}:50.738093890",
        "response_message": {
          "role": "AI",
          "text": "Mocking number {0}."
        }
      }""".replace(
        ...
    )...
```

and run it, which generates `src-tauri/api/sample-calls/get_api_calls-full.yaml` and `src-tauri/api/sample-calls/get_api_calls-offset.yaml`. `src-tauri/api/sample-calls/get_api_calls-small.yaml` we manually edit ourselves:

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
        "response_message": {
          "role": "AI",
          "text": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
        }
      },
      {
        "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
        "timestamp": "2024-01-16T08:50:19.738093890",
        "response_message": {
          "role": "AI",
          "text": "Yes, it works. How can I assist you today?"
        }
      }
    ]
sideEffects:
  database:
    startStateDump: conversation-continued
    endStateDump: conversation-continued

```

We then edit `src-tauri/src/commands/llms/chat.rs` in a similar fashion, replacing `LlmCall` with `LightweightLlmCall` in most places, except that we still construct the `LlmCall` in order to seamlessly convert it to a `NewLlmCallRow` and then into `LlmCall`.

```rs
...
use crate::models::llm_calls::{
    ..., LightweightLlmCall, ...
};
...

async fn chat_helper(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...
    let llm_call = LlmCall {
        ...
    };
    ...

    Ok(llm_call.into())
}

#[tauri::command(async)]
#[specta]
pub async fn chat(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...
}
```

We manually edit `src-tauri/api/sample-calls/chat-start-conversation.yaml`:

```yaml
...
response:
  message: >
    {
      "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
      "timestamp": "2024-01-16T08:50:19.738093890",
      "response_message": {
        "role": "AI",
        "text": "Yes, it works. How can I assist you today?"
      }
    }
...
```

and `src-tauri\api\sample-calls\chat-continue-conversation.yaml`:

```yaml
...
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
...
```

Specta auto-updates `src-svelte/src/lib/bindings.ts`, which for some reason does not generate a clean diff this time. We then edit `src-svelte/src/routes/api-calls/ApiCalls.svelte` to use these new bindings:

```svelte
<script lang="ts">
  import { ..., type LightweightLlmCall } from "$lib/bindings";
  ...

  let llmCalls: LightweightLlmCall[] = [];
  ...
</script>

<InfoBox ...>
  ...
                  <div class="text">{call.response_message.text}</div>
  ...
</InfoBox>
```

and `src-svelte/src/routes/chat/Chat.svelte`:

```ts
  async function sendChatMessage(message: string) {
    ...
    try {
      let llmCall = await chat(...);
      appendMessage(llmCall.response_message);
    } ...
  }
```

and `src-svelte/src/routes/chat/Chat.test.ts`:

```ts
...
import type { ..., LightweightLlmCall } from "$lib/bindings";
...

  async function sendChatMessage(
    ...
  ) {
    ...
    const lastResult: LightweightLlmCall =
      ...;
    const aiResponse = lastResult.response_message.text;
    ...
  }
```

We then rebase our previous changes onto this refactor, which is largely straightforward. The main change is that we also edit the test code in `src-tauri/src/commands/llms/chat.rs`:

```rs
#[cfg(test)]
mod tests {
    ...

    #[derive(...)]
    struct ChatRequest {
        ...
        previous_call_id: Option<Uuid>,
    }

    ...

    impl SampleCallTestCase<ChatRequest, ZammResult<LightweightLlmCall>> for ChatTestCase {
        ...

        async fn make_request(
            ...
        ) -> ZammResult<LightweightLlmCall> {
            ...

            chat_helper(
                ...,
                actual_args.previous_call_id,
                ...
            )
            .await
        }

        ...
    }

    ...
}
```

Then, in `src-tauri/src/test_helpers/database_contents.rs`, we try to read in the database:

```rs
pub async fn get_database_contents(
    zamm_db: &ZammDatabase,
) -> ZammResult<DatabaseContents> {
    let db_mutex: &mut MutexGuard<'_, Option<SqliteConnection>> =
        &mut zamm_db.0.lock().await;
    let db = ...;
    let api_keys = ...;
    let llm_call_left_joins = llm_calls::table
        .left_join(
            llm_call_continuations::dsl::llm_call_continuations
                .on(llm_calls::id.eq(llm_call_continuations::next_call_id)),
        )
        .select((
            llm_calls::all_columns,
            llm_call_continuations::previous_call_id.nullable(),
        ))
        .get_results::<LlmCallLeftJoinResult>(db)?;
    let llm_calls_result: ZammResult<Vec<LlmCall>> = llm_call_left_joins.into_iter().map(|lf| {
        let (llm_call_row, previous_call_id) = lf;
        let next_calls_result = llm_call_continuations::table
            .select(llm_call_continuations::next_call_id)
            .filter(llm_call_continuations::previous_call_id.eq(previous_call_id))
            .load::<EntityId>(db)?;
        Ok(((llm_call_row, previous_call_id), next_calls_result).into())
    }).collect();
    Ok(DatabaseContents {
        api_keys,
        llm_calls: llm_calls_result?,
    })
}

```

Unfortunately this results in quite a few errors:

```
error[E0277]: the trait bound `std::option::Option<EntityId>: diesel::Expression` is not satisfied
  --> src/test_helpers/database_contents.rs:50:62
   |
50 | ...ions::previous_call_id.eq(previous_call_id))
   |                           ^^ the trait `diesel::Expression` is not implemented for `std::option::Option<EntityId>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             Box<T>
             diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on
             diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version
             diesel_migrations::migration_harness::__diesel_schema_migrations::columns::star
             schema::llm_calls::columns::completion
             schema::llm_calls::columns::prompt
             schema::llm_calls::columns::total_tokens
             schema::llm_calls::columns::response_tokens
           and 98 others
   = note: required for `std::option::Option<EntityId>` to implement `AsExpression<diesel::sql_types::Text>`

error[E0277]: the trait bound `std::option::Option<EntityId>: ValidGrouping<()>` is not satisfied
  --> src/test_helpers/database_contents.rs:50:14
   |
50 |             .filter(llm_call_continuations::previous_call_id.eq...
   |              ^^^^^^ the trait `ValidGrouping<()>` is not implemented for `std::option::Option<EntityId>`
   |
   = help: the following other types implement trait `ValidGrouping<GroupByClause>`:
             <Box<T> as ValidGrouping<GB>>
             <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on as ValidGrouping<()>>
             <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on as ValidGrouping<__GB>>
             <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version as ValidGrouping<()>>
             <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version as ValidGrouping<__GB>>
             <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::star as ValidGrouping<__GB>>
             <schema::llm_calls::columns::completion as ValidGrouping<()>>
             <schema::llm_calls::columns::completion as ValidGrouping<__GB>>
           and 117 others
   = note: required for `Eq<previous_call_id, Option<EntityId>>` to implement `ValidGrouping<()>`
   = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-15765192543693787375.txt'
   = note: required for `Grouped<Eq<previous_call_id, ...>>` to implement `NonAggregate`
   = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-1346938644422633050.txt'
   = note: required for `SelectStatement<FromClause<table>, ...>` to implement `FilterDsl<expression::grouped::Grouped<expression::operators::Eq<llm_call_continuations::columns::previous_call_id, std::option::Option<EntityId>>>>`
   = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-7654985430760998731.txt'

error[E0277]: the trait bound `std::option::Option<EntityId>: diesel::Expression` is not satisfied
  --> src/test_helpers/database_contents.rs:50:14
   |
50 |             .filter(llm_call_continuations::previous_call_id.eq...
   |              ^^^^^^ the trait `diesel::Expression` is not implemented for `std::option::Option<EntityId>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             Box<T>
             diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on
             diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version
             diesel_migrations::migration_harness::__diesel_schema_migrations::columns::star
             schema::llm_calls::columns::completion
             schema::llm_calls::columns::prompt
             schema::llm_calls::columns::total_tokens
             schema::llm_calls::columns::response_tokens
           and 98 others
   = note: required for `Eq<previous_call_id, Option<EntityId>>` to implement `diesel::Expression`
   = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-15765192543693787375.txt'
   = note: required for `SelectStatement<FromClause<table>, ...>` to implement `FilterDsl<expression::grouped::Grouped<expression::operators::Eq<llm_call_continuations::columns::previous_call_id, std::option::Option<EntityId>>>>`
   = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-7654985430760998731.txt'

error[E0277]: the trait bound `std::option::Option<EntityId>: AppearsOnTable<llm_call_continuations::table>` is not satisfied
    --> src/test_helpers/database_contents.rs:51:31
     |
51   |             .load::<EntityId>(db)?;
     |              ----             ^^ the trait `AppearsOnTable<llm_call_continuations::table>` is not implemented for `std::option::Option<EntityId>`
     |              |
     |              required by a bound introduced by this call
     |
     = help: the following other types implement trait `AppearsOnTable<QS>`:
               <Box<T> as AppearsOnTable<QS>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on as AppearsOnTable<QS>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version as AppearsOnTable<QS>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::star as AppearsOnTable<diesel_migrations::migration_harness::__diesel_schema_migrations::table>>
               <schema::llm_calls::columns::completion as AppearsOnTable<QS>>
               <schema::llm_calls::columns::prompt as AppearsOnTable<QS>>
               <schema::llm_calls::columns::total_tokens as AppearsOnTable<QS>>
               <schema::llm_calls::columns::response_tokens as AppearsOnTable<QS>>
             and 98 others
     = note: required for `Eq<previous_call_id, Option<EntityId>>` to implement `AppearsOnTable<llm_call_continuations::table>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-15765192543693787375.txt'
     = note: 1 redundant requirement hidden
     = note: required for `Grouped<Eq<previous_call_id, ...>>` to implement `AppearsOnTable<llm_call_continuations::table>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-1346938644422633050.txt'
     = note: required for `WhereClause<Grouped<Eq<..., ...>>>` to implement `diesel::query_builder::where_clause::ValidWhereClause<FromClause<llm_call_continuations::table>>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6945533729776045286.txt'
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `Query`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `LoadQuery<'_, _, EntityId>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/query_dsl/mod.rs:1543:15
     |
1541 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<...
     |        ---- required by a bound in this associated function
1542 |     where
1543 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`

error[E0277]: the trait bound `std::option::Option<EntityId>: QueryId` is not satisfied
    --> src/test_helpers/database_contents.rs:51:31
     |
51   |             .load::<EntityId>(db)?;
     |              ----             ^^ the trait `QueryId` is not implemented for `std::option::Option<EntityId>`
     |              |
     |              required by a bound introduced by this call
     |
     = help: the following other types implement trait `QueryId`:
               Box<T>
               diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on
               diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version
               diesel_migrations::migration_harness::__diesel_schema_migrations::columns::star
               diesel_migrations::migration_harness::__diesel_schema_migrations::table
               DeleteStatement<T, U, Ret>
               FromClause<F>
               diesel::query_builder::insert_statement::insert_with_default_for_sqlite::SqliteBatchInsertWrapper<V, T, QId, STATIC_QUERY_ID>
             and 193 others
     = note: required for `Eq<previous_call_id, Option<EntityId>>` to implement `QueryId`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-15765192543693787375.txt'
     = note: 3 redundant requirements hidden
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `QueryId`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `LoadQuery<'_, _, EntityId>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/query_dsl/mod.rs:1543:15
     |
1541 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<...
     |        ---- required by a bound in this associated function
1542 |     where
1543 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`

error[E0277]: the trait bound `EntityId: QueryFragment<Sqlite>` is not satisfied
    --> src/test_helpers/database_contents.rs:51:31
     |
51   |             .load::<EntityId>(db)?;
     |              ----             ^^ the trait `QueryFragment<Sqlite>` is not implemented for `EntityId`
     |              |
     |              required by a bound introduced by this call
     |
     = help: the following other types implement trait `QueryFragment<DB, SP>`:
               <Box<T> as QueryFragment<DB>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::run_on as QueryFragment<DB>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::version as QueryFragment<DB>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::columns::star as QueryFragment<DB>>
               <diesel_migrations::migration_harness::__diesel_schema_migrations::table as QueryFragment<DB>>
               <DeleteStatement<T, U, Ret> as QueryFragment<DB>>
               <FromClause<F> as QueryFragment<DB>>
               <diesel::query_builder::insert_statement::insert_with_default_for_sqlite::SqliteBatchInsertWrapper<Vec<diesel::query_builder::insert_statement::ValuesClause<V, Tab>>, Tab, QId, STATIC_QUERY_ID> as QueryFragment<Sqlite>>
             and 232 others
     = note: required for `std::option::Option<EntityId>` to implement `QueryFragment<Sqlite>`
     = note: 8 redundant requirements hidden
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `QueryFragment<Sqlite>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `LoadQuery<'_, _, EntityId>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/query_dsl/mod.rs:1543:15
     |
1541 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<...
     |        ---- required by a bound in this associated function
1542 |     where
1543 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`

error[E0271]: type mismatch resolving `<Sqlite as SqlDialect>::SelectStatementSyntax == NotSpecialized`
    --> src/test_helpers/database_contents.rs:51:31
     |
51   |             .load::<EntityId>(db)?;
     |              ----             ^^ expected `NotSpecialized`, found `AnsiSqlSelectStatement`
     |              |
     |              required by a bound introduced by this call
     |
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `QueryFragment<Sqlite>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
     = note: required for `SelectStatement<FromClause<...>, ..., ..., ...>` to implement `LoadQuery<'_, _, EntityId>`
     = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3040bd9d14ab4a72.long-type-6635517348700182216.txt'
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/query_dsl/mod.rs:1543:15
     |
1541 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<...
     |        ---- required by a bound in this associated function
1542 |     where
1543 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`
```

It turns out that all those errors was just to say that we shouldn't do a SQL query on an `Option`. Instead, we change the code to:

```rs
    let llm_calls_result: ZammResult<Vec<LlmCall>> = llm_call_left_joins.into_iter().map(|lf| {
        let (llm_call_row, previous_call_id) = lf;
        let next_calls_result = match previous_call_id.as_ref() {
            Some(previous_id) =>
                llm_call_continuations::table
                .select(llm_call_continuations::next_call_id)
                .filter(llm_call_continuations::previous_call_id.eq(previous_id))
                .load::<EntityId>(db)?,
            None => Vec::new(),
        };
        Ok(((llm_call_row, previous_call_id), next_calls_result).into())
    }).collect();
```

We do the same variable rename in `src-tauri/src/models/llm_calls/llm_call.rs` as well, from `llmCallRow` to `llm_call_row`, so that that file now has this function:

```rs
impl From<LlmCallQueryResults> for LlmCall {
    fn from(query_results: LlmCallQueryResults) -> Self {
        let ((llm_call_row, previous_call_id), next_call_ids) = query_results;
        ...
    }
}
```

We need to write this to the database too, so we create `src-tauri/src/models/llm_calls/linkage.rs`:

```rs
use crate::models::llm_calls::entity_id::EntityId;
use crate::schema::llm_call_continuations;
use diesel::prelude::*;

#[derive(Insertable)]
#[diesel(table_name = llm_call_continuations)]
pub struct NewLlmCallContinuation<'a> {
    pub previous_call_id: &'a EntityId,
    pub next_call_id: &'a EntityId,
}

```

We expose that in `src-tauri/src/models/llm_calls/mod.rs`, using `#[cfg(test)]` to prevent it from being removed as part of linting:

```rs
...
mod linkage;
...

#[cfg(test)]
pub use linkage::NewLlmCallContinuation;
...
```

and make use of this in `src-tauri/src/models/llm_calls/llm_call.rs`:

```rs
...
use crate::models::llm_calls::linkage::NewLlmCallContinuation;
...

impl LlmCall {
    ...

    pub fn as_continuation(&self) -> Option<NewLlmCallContinuation> {
        self.conversation.previous_call_id.as_ref().map(|id| NewLlmCallContinuation {
            previous_call_id: id,
            next_call_id: &self.id,
        })
    }
}
```

Finally, we can modify `src-tauri/src/test_helpers/database_contents.rs`:

```rs
...
use crate::models::llm_calls::{
    ..., NewLlmCallContinuation,
};
...
use crate::schema::{..., llm_call_continuations, ...};
...

impl DatabaseContents {
    ...

    pub fn insertable_call_continuations(&self) -> Vec<NewLlmCallContinuation> {
        self.llm_calls.iter().filter_map(|k| k.as_continuation()).collect()
    }
}

...

pub async fn read_database_contents(
    ...
) -> ZammResult<()> {
    ...
    db.transaction::<(), diesel::result::Error, _>(|conn| {
        ...
        diesel::insert_into(llm_call_continuations::table)
            .values(&db_contents.insertable_call_continuations())
            .execute(conn)?;
        Ok(())
    })?;
    ...
}
```

Tests are failing because serialization/deserialization is failing. We edit `src-tauri/src/models/llm_calls/various.rs` to make the conversation metadata fields optional:

```rs
#[derive(...)]
pub struct ConversationMetadata {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub previous_call_id: Option<EntityId>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub next_call_ids: Vec<EntityId>,
}
```

We then edit `src-tauri/src/models/llm_calls/llm_call.rs` to make the conversation metadata itself optional:

```rs
#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct LlmCall {
    ...,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub conversation: Option<ConversationMetadata>,
}

impl LlmCall {
    ...

    pub fn as_continuation(&self) -> Option<NewLlmCallContinuation> {
        self.conversation.as_ref().and_then(|c| {
            c.previous_call_id
                .as_ref()
                .map(|id| NewLlmCallContinuation {
                    previous_call_id: id,
                    next_call_id: &self.id,
                })
        })
    }
}

...

impl From<LlmCallQueryResults> for LlmCall {
    fn from(query_results: LlmCallQueryResults) -> Self {
        ...
        let conversation_metadata =
            if previous_call_id.is_some() || !next_call_ids.is_empty() {
                Some(ConversationMetadata {
                    previous_call_id,
                    next_call_ids,
                })
            } else {
                None
            };
        ...
    }
}
```

Next, we edit `src-tauri/src/commands/llms/chat.rs`:

```rs
async fn chat_helper(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...
    let conversation_metadata = previous_call_id.map(|id| ConversationMetadata {
        previous_call_id: Some(EntityId { uuid: id }),
        next_call_ids: [].to_vec(),
    });
    ...
    if let Some(conn) = db.as_mut() {
        ...
        if let Some(previous_call_uuid) = llm_call
            .conversation
            .as_ref()
            .and_then(|c| c.previous_call_id.as_ref())
        {
            ...
        }
    }
    ...
}
```

The tests pass for now because we aren't triggering any of the new functionality yet. However, we can't quite commit yet because of the warning

```
warning: method `as_continuation` is never used
  --> src/models/llm_calls/llm_call.rs:39:12
   |
22 | impl LlmCall {
   | ------------ method in this implementation
...
39 |     pub fn as_continuation(&self) -> Option<NewLlmCallContinuation> {
   |            ^^^^^^^^^^^^^^^
   |
   = note: `-D dead-code` implied by `-D warnings`
   = help: to override `-D warnings` add `#[allow(dead_code)]`
```

We edit `src-tauri/src/commands/llms/chat.rs` to reuse the functionality instead of inserting the row directly:

```rs
async fn chat_helper(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...
    if let Some(conn) = db.as_mut() {
        ...

        if llm_call.conversation.as_ref().and_then(|c| c.previous_call_id.as_ref()).is_some()
        {
            diesel::insert_into(llm_call_continuations::table)
                .values(llm_call.as_continuation())
                .execute(conn)?;
        }
    } // todo: warn users if DB write unsuccessful

    ...
}
```

Finally, we have the warning

```
warning: this function has too many arguments (8/7)
  --> src/commands/llms/chat.rs:20:1
   |
20 | / async fn chat_helper(
21 | |     zamm_api_keys: &ZammApiKeys,
22 | |     zamm_db: &ZammDatabase,
23 | |     provider: Service,
...  |
28 | |     http_client: reqwest_middleware::ClientWithMiddleware,
29 | | ) -> ZammResult<LightweightLlmCall> {
   | |___________________________________^
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#too_many_arguments
   = note: `-D clippy::too-many-arguments` implied by `-D warnings`
   = help: to override `-D warnings` add `#[allow(clippy::too_many_arguments)]`

warning: `zamm` (bin "zamm") generated 2 warnings
warning: `zamm` (bin "zamm" test) generated 1 warning (1 duplicate)
```

As such, we edit `src-tauri/src/commands/llms/chat.rs` to put all arguments into a single struct:

```rs
...
use serde::{Deserialize, Serialize};
...

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, specta::Type)]
pub struct ChatArgs {
    provider: Service,
    llm: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    temperature: Option<f32>,
    prompt: Vec<ChatMessage>,
    #[serde(skip_serializing_if = "Option::is_none")]
    previous_call_id: Option<Uuid>,
}

async fn chat_helper(
    zamm_api_keys: &ZammApiKeys,
    zamm_db: &ZammDatabase,
    args: ChatArgs,
    http_client: reqwest_middleware::ClientWithMiddleware,
) -> ZammResult<LightweightLlmCall> {
    ...
    let config = match args.provider {
        ...
    };
    ...
    let messages: Vec<ChatCompletionRequestMessage> =
        args.prompt.clone()...;
    ...
    let conversation_metadata = args.previous_call_id.map(...);
    ...
    let llm_call = LlmCall {
        ...
        request: Request {
            ...,
            prompt: Prompt::Chat(ChatPrompt {
                messages: args.prompt,
            }),
        },
        ...
    };
    ...
}

#[tauri::command(async)]
#[specta]
pub async fn chat(
    api_keys: State<'_, ZammApiKeys>,
    database: State<'_, ZammDatabase>,
    args: ChatArgs,
) -> ZammResult<LightweightLlmCall> {
    ...
    chat_helper(&api_keys, &database, args, client_with_middleware).await
}

#[cfg(test)]
mod tests {
    ...

    #[derive(...)]
    struct ChatRequest {
        args: ChatArgs,
    }

    ...

    impl SampleCallTestCase<ChatRequest, ZammResult<LightweightLlmCall>> for ChatTestCase {
        ...

        async fn make_request(
            &mut self,
            args: &Option<ChatRequest>,
            ...
        ) -> ZammResult<LightweightLlmCall> {
            let actual_request = args.as_ref().unwrap().clone();
            ...

            chat_helper(
                ...,
                actual_request.args,
                ...
            )
            .await
        }
    
        ...
    }

    ...
}
```

Now test cases finally fail, so we edit `src-tauri/api/sample-calls/chat-start-conversation.yaml` to be the same as before but wrapped in an `args` key:

```yaml
request:
  - chat
  - >
    {
      "args": {
        "provider": "OpenAI",
        ...
      }
    }
response:
  ...
...
```

We edit `src-tauri/api/sample-calls/chat-continue-conversation.yaml` similarly:

```yaml
request:
  - chat
  - >
    {
      "args": {
        ...
      }
    }
...
```

Now we can commit with all backend tests passing. We edit `src-tauri/api/sample-calls/chat-continue-conversation.yaml` to actually make use of the new functionality:

```yaml
request:
  - chat
  - >
    {
      "args": {
        ...,
        "previous_call_id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
        "prompt": ...
      }
    }
...
```

Predictably, tests start failing now. We find out that the generated database dump is not quite what we want, because the "next_call_ids" is ending up in the wrong place. We edit `src-tauri/src/test_helpers/database_contents.rs` to fix the filter condition, and to dump the new table:

```rs
pub async fn get_database_contents(
    ...
) -> ZammResult<DatabaseContents> {
    ...
    let llm_calls_result: ZammResult<Vec<LlmCall>> = llm_call_left_joins
        .into_iter()
        .map(|lf| {
            let (llm_call_row, previous_call_id) = lf;
            let next_calls_result = llm_call_continuations::table
                .select(llm_call_continuations::next_call_id)
                .filter(llm_call_continuations::previous_call_id.eq(&llm_call_row.id))
                .load::<EntityId>(db)?;
            Ok(((llm_call_row, previous_call_id), next_calls_result).into())
        })
        .collect();
    ...
}

...

pub fn dump_sqlite_database(...) {
    let dump_output = std::process::Command::new("sqlite3")
        ...
        .arg(".dump api_keys llm_calls llm_call_continuations")
        ...;
    ...
}
```

We update `src-tauri/api/sample-database-writes/conversation-continued/dump.yaml`:

```yaml
api_keys: []
llm_calls:
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
  ...
  conversation:
    next_call_ids:
    - c13c1e67-2de3-48de-a34c-a32079c03316
- id: c13c1e67-2de3-48de-a34c-a32079c03316
  ...
  conversation:
    previous_call_id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
```

and update `src-tauri/api/sample-database-writes/conversation-continued/dump.sql` with the line

```sql
INSERT INTO llm_call_continuations VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a74','c13c1e67-2de3-48de-a34c-a32079c03316');
```

Now other tests fail with

```
---- commands::llms::get_api_calls::tests::test_get_api_calls_less_than_10 stdout ----
Test will use temp directory at /tmp/zamm/tests/get_api_calls/test_get_api_calls_less_than_10
thread 'commands::llms::get_api_calls::tests::test_get_api_calls_less_than_10' panicked at src/test_helpers/api_testing.rs:335:26:
called `Result::unwrap()` on an `Err` value: Serde { source: Yaml { source: Error("llm_calls[1].conversation: missing field `next_call_ids`", line: 57, column: 5) } }
```

It [turns out](https://github.com/dtolnay/serde-yaml/issues/32#issuecomment-257743613) we can use serde's `default` attribute to mark how a field should be deserialized if it isn't present. So, we edit `src-tauri/src/models/llm_calls/various.rs`:

```rs
#[derive(...)]
pub struct ConversationMetadata {
    ...,
    #[serde(skip_serializing_if = "Vec::is_empty", default)]
    pub next_call_ids: Vec<EntityId>,
}
```

Finally, we edit `src-tauri/api/sample-calls/get_api_call-start-conversation.yaml`:

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
      ...,
      "conversation": {
        "next_call_ids": [
          "c13c1e67-2de3-48de-a34c-a32079c03316"
        ]
      }
    }
...
```

and `src-tauri/api/sample-calls/get_api_call-continue-conversation.yaml`:

```yaml
request:
  - get_api_call
  - >
    {
      "id": "c13c1e67-2de3-48de-a34c-a32079c03316"
    }
response:
  message: >
    {
      ...,
      "conversation": {
        "previous_call_id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74"
      }
    }
...
```

Now all the backend tests pass.

Now that we know this is a possibility, we can edit `src-tauri/src/models/llm_calls/various.rs` again to simplify things:

```rs
#[derive(..., Default, ...)]
pub struct ConversationMetadata {
    ...
}

impl ConversationMetadata {
    pub fn is_default(&self) -> bool {
        self.previous_call_id.is_none() && self.next_call_ids.is_empty()
    }
}

```

Then, we edit `src-tauri\src\models\llm_calls\llm_call.rs` to use `default` instead and avoid nested Options:

```rs
#[derive(...)]
pub struct LlmCall {
    ...,
    #[serde(skip_serializing_if = "ConversationMetadata::is_default", default)]
    pub conversation: ConversationMetadata,
}

impl LlmCall {
    ...

    pub fn as_continuation(&self) -> Option<NewLlmCallContinuation> {
        self.conversation.previous_call_id
            .as_ref()
            .map(|id| NewLlmCallContinuation {
                ...
            })
    }
}

...

impl From<LlmCallQueryResults> for LlmCall {
    fn from(query_results: LlmCallQueryResults) -> Self {
        ...
        let conversation_metadata = ConversationMetadata {
            previous_call_id,
            next_call_ids,
        };
        ...
    }
}
```

Finally, we update `src-tauri/src/commands/llms/chat.rs` to use the updated schema:

```rs
async fn chat_helper(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...
    let previous_call_id = args.previous_call_id.map(|id| EntityId { uuid: id });
    let conversation_metadata = ConversationMetadata {
        previous_call_id,
        next_call_ids: [].to_vec(),
    };
    ...
    if let Some(conn) = db.as_mut() {
        ...

        if llm_call.conversation.previous_call_id.is_some()
        {
            ...
        }
    }

    ...
}
```

### Built-in Diesel associations

We try to see if we can use Diesel's built-in [associations](https://docs.rs/diesel/latest/diesel/associations/index.html) functionality here. We edit `src-tauri\src\models\llm_calls\row.rs` to manually implement `Identifiable` for `LlmCallRow` because at first it was giving us errors:

```rs
...
use diesel::associations::HasTable;
...

impl HasTable for LlmCallRow {
    type Table = llm_calls::table;

    fn table() -> Self::Table {
        llm_calls::table
    }
}

impl Identifiable for LlmCallRow {
    type Id = EntityId;

    fn id(self) -> Self::Id {
        self.id
    }
}
```

However, we later find that the errors are not reproduceable anymore if we automatically derive the class:

```rs
#[derive(..., Identifiable)]
#[diesel(table_name = llm_calls)]
pub struct LlmCallRow {
    ...
}
```

and we edit `src-tauri\src\models\llm_calls\entity_id.rs` to fit the requirements for identifiability:

```rs
#[derive(
    ..., PartialEq, Eq, Hash, ...
)]
...
pub struct EntityId {
    pub uuid: Uuid,
}
```

We find out that what we're trying to do [is](https://github.com/diesel-rs/diesel/issues/2613) not [possible](https://github.com/diesel-rs/diesel/issues/2142) with Diesel's built-in functionality. If we want to, this is as far as we can possibly take this by editing `src-tauri\src\models\llm_calls\linkage.rs`:

```rs
...
use crate::models::llm_calls::llm_call::LlmCall;
use diesel::associations::Associations;
...

#[derive(Associations)]
#[diesel(belongs_to(LlmCall, foreign_key = previous_call_id))]
#[diesel(table_name = llm_call_continuations)]
pub struct ConversationMetadata {
    pub previous_call_id: EntityId,
    pub next_call_id: EntityId,
}

...
```

### Returning display data

We realize that returning just the ID of the linked calls won't be sufficient for displaying links to them in the frontend, so we have to do extra joins. It appears that selecting from joins on [subqueries](https://www.reddit.com/r/rust/comments/17qgdki/dieselrs_join_with_subquery/) is not supported at this point in time. However, it appears that there is a [view trick](https://deterministic.space/diesel-view-table-trick.html) we can use. We modify `src-tauri/migrations/2024-05-13-135101_add_api_call_continuations/up.sql`, adding a semicolon to the previous statement:

```sql
CREATE TABLE llm_call_continuations (
  ...
);

CREATE VIEW llm_call_named_continuations AS
  SELECT
    llm_call_continuations.previous_call_id AS previous_call_id,
    previous_call.completion AS previous_call_completion,
    llm_call_continuations.next_call_id AS next_call_id,
    next_call.completion AS next_call_completion
  FROM
    llm_call_continuations
    JOIN llm_calls AS previous_call ON llm_call_continuations.previous_call_id = previous_call.id
    JOIN llm_calls AS next_call ON llm_call_continuations.next_call_id = next_call.id;

```

and we edit `src-tauri/migrations/2024-05-13-135101_add_api_call_continuations/down.sql` as well:

```sql
DROP TABLE llm_call_continuations;
DROP VIEW llm_call_named_continuations;

```

Then we can manually create `src-tauri/src/views.rs`:

```rs
use crate::schema::llm_calls;

diesel::table! {
    llm_call_named_continuations (previous_call_id, next_call_id) {
        previous_call_id -> Text,
        previous_call_completion -> Text,
        next_call_id -> Text,
        next_call_completion -> Text,
    }
}

diesel::allow_tables_to_appear_in_same_query!(llm_call_named_continuations, llm_calls);

```

and note it in `src-tauri/src/main.rs`:

```rs
...
mod views;
...
```

Next, we make use of this new schema in `src-tauri\src\models\llm_calls\various.rs`, where we set the snippet to be a truncated form of the message. We may want to let the frontend handle this logic in the future, but right now it seems best to avoid having too much redundancy in the exported YAML files:

```rs
#[derive(Debug, Clone, Serialize, Deserialize, specta::Type)]
pub struct LlmCallReference {
    pub id: EntityId,
    pub snippet: String,
}

impl From<(EntityId, ChatMessage)> for LlmCallReference {
    fn from((id, message): (EntityId, ChatMessage)) -> Self {
        let text = match message {
            ChatMessage::System { text } => text,
            ChatMessage::Human { text } => text,
            ChatMessage::AI { text } => text,
        };
        let truncated_text = text
            .split_whitespace()
            .take(10)
            .collect::<Vec<&str>>()
            .join(" ");
        let snippet = if text == truncated_text {
            text
        } else {
            format!("{}...", truncated_text)
        };
        Self { id, snippet }
    }
}

#[derive(...)]
pub struct ConversationMetadata {
    #[serde(...)]
    pub previous_call: Option<LlmCallReference>,
    #[serde(...)]
    pub next_calls: Vec<LlmCallReference>,
}

impl ConversationMetadata {
    pub fn is_default(&self) -> bool {
        self.previous_call.is_none() && self.next_calls.is_empty()
    }
}

```

We then edit `src-tauri\src\models\llm_calls\llm_call.rs`, marking some things as test-only because they're no longer accessed from regular code paths:

```rs
use crate::models::llm_calls::chat_message::ChatMessage;
...
#[cfg(test)]
use crate::models::llm_calls::{NewLlmCallContinuation, NewLlmCallRow};
...

#[cfg(test)]
impl LlmCall {
    ...

    pub fn as_continuation(&self) -> Option<NewLlmCallContinuation> {
        self.conversation
            .previous_call
            .as_ref()
            .map(|call| NewLlmCallContinuation {
                previous_call_id: &call.id,
                next_call_id: &self.id,
            })
    }
}

pub type LlmCallLeftJoinResult = (LlmCallRow, Option<EntityId>, Option<ChatMessage>);
pub type LlmCallQueryResults = (LlmCallLeftJoinResult, Vec<(EntityId, ChatMessage)>);

impl From<LlmCallQueryResults> for LlmCall {
    fn from(query_results: LlmCallQueryResults) -> Self {
        let ((llm_call_row, previous_call_id, previous_call_completion), next_calls) =
            query_results;
        
        ...

        let previous_call: Option<LlmCallReference> =
            if let (Some(id), Some(completion)) =
                (previous_call_id, previous_call_completion)
            {
                Some((id, completion).into())
            } else {
                None
            };
        let conversation_metadata = ConversationMetadata {
            previous_call,
            next_calls: next_calls
                .into_iter()
                .map(|(id, completion)| (id, completion).into())
                .collect(),
        };
        ...
    }
}
```

We continue using the new models in `src-tauri\src\commands\llms\chat.rs`, where we avoid constructing a `conversation_metadata` and therefore an LlmCall object because it requires a database read in order to retrieve the snippet, which is unnecessary because it isn't returned to the frontend anyways:

```rs
...
use crate::models::llm_calls::{
    ..., NewLlmCallContinuation, NewLlmCallRow,
};
...

async fn chat_helper(
    ...
) -> ZammResult<LightweightLlmCall> {
    ...

    let config = match &args.provider {
        ...
    };

    ... (no more conversation_metadata) ...

    let new_id = EntityId {
        uuid: Uuid::new_v4(),
    };
    let timestamp = chrono::Utc::now().naive_utc();
    let completion: ChatMessage = sole_choice.try_into()?;

    if let Some(conn) = db.as_mut() {
        diesel::insert_into(llm_calls::table)
            .values(NewLlmCallRow {
                id: &new_id,
                timestamp: &timestamp,
                provider: &args.provider,
                llm_requested: &requested_model,
                llm: &response.model,
                temperature: &requested_temperature,
                prompt_tokens: token_metadata.prompt.as_ref(),
                response_tokens: token_metadata.response.as_ref(),
                total_tokens: token_metadata.total.as_ref(),
                prompt: &Prompt::Chat(ChatPrompt {
                    messages: args.prompt,
                }),
                completion: &completion,
            })
            .execute(conn)?;

        if let Some(previous_id) = previous_call_id {
            diesel::insert_into(llm_call_continuations::table)
                .values(NewLlmCallContinuation {
                    previous_call_id: &previous_id,
                    next_call_id: &new_id,
                })
                .execute(conn)?;
        }
    } // todo: warn users if DB write unsuccessful

    Ok(LightweightLlmCall {
        id: new_id,
        timestamp,
        response_message: completion,
    })
}
```

Next, we edit `src-tauri\src\commands\llms\get_api_call.rs`:

```rs
...
use crate::schema::llm_calls;
use crate::views::llm_call_named_continuations;
...

async fn get_api_call_helper(
    ...
) -> ZammResult<LlmCall> {
    ...

    let previous_call_result: LlmCallLeftJoinResult = llm_calls::table
        .left_join(
            llm_call_named_continuations::dsl::llm_call_named_continuations
                .on(llm_calls::id.eq(llm_call_named_continuations::next_call_id)),
        )
        .select((
            llm_calls::all_columns,
            llm_call_named_continuations::previous_call_id.nullable(),
            llm_call_named_continuations::previous_call_completion.nullable(),
        ))
        ...;
    let next_calls_result = llm_call_named_continuations::table
        .select((
            llm_call_named_continuations::next_call_id,
            llm_call_named_continuations::next_call_completion,
        ))
        .filter(llm_call_named_continuations::previous_call_id.eq(parsed_uuid))
        .load::<(EntityId, ChatMessage)>(conn)?;
    ...
}
```

Finally, we edit the reading and writing from a database in `src-tauri/src/test_helpers/database_contents.rs`:

```rs
...
use crate::views::llm_call_named_continuations;
...

pub async fn get_database_contents(
    ...
) -> ZammResult<DatabaseContents> {
    ...
    let llm_call_left_joins = llm_calls::table
        .left_join(
            llm_call_named_continuations::dsl::llm_call_named_continuations
                .on(llm_calls::id.eq(llm_call_named_continuations::next_call_id)),
        )
        .select((
            llm_calls::all_columns,
            llm_call_named_continuations::previous_call_id.nullable(),
            llm_call_named_continuations::previous_call_completion.nullable(),
        ))
        .get_results::<LlmCallLeftJoinResult>(db)?;
    let llm_calls_result: ZammResult<Vec<LlmCall>> = llm_call_left_joins
        .into_iter()
        .map(|lf| {
            let (llm_call_row, previous_call_id, previous_call_completion) = lf;
            let next_calls_result = llm_call_named_continuations::table
                .select((
                    llm_call_named_continuations::next_call_id,
                    llm_call_named_continuations::next_call_completion,
                ))
                .filter(
                    llm_call_named_continuations::previous_call_id.eq(&llm_call_row.id),
                )
                .load::<(EntityId, ChatMessage)>(db)?;
            Ok((
                (llm_call_row, previous_call_id, previous_call_completion),
                next_calls_result,
            )
                .into())
        })
        .collect();
    ...
}
```

We update the database dump file at `src-tauri\api\sample-database-writes\conversation-continued\dump.yaml` with the new format for LLM call references:

```yaml
api_keys: []
llm_calls:
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
  ...
  conversation:
    next_calls:
    - id: c13c1e67-2de3-48de-a34c-a32079c03316
      snippet: 'Sure, here''s a joke for you: Why don''t scientists trust...'
- id: c13c1e67-2de3-48de-a34c-a32079c03316
  ...
  conversation:
    previous_call:
      id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
      snippet: Yes, it works. How can I assist you today?

```

and the actual call at `src-tauri/api/sample-calls/get_api_call-start-conversation.yaml`:

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
      ...
      "conversation": {
        "next_calls": [
          {
            "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
            "snippet": "Sure, here's a joke for you: Why don't scientists trust..."
          }
        ]
      }
    }
...
```

and `src-tauri/api/sample-calls/get_api_call-continue-conversation.yaml`:

```yaml
request:
  - get_api_call
  - >
    {
      "id": "c13c1e67-2de3-48de-a34c-a32079c03316"
    }
response:
  message: >
    {
      ...
      "conversation": {
        "previous_call": {
          "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
          "snippet": "Yes, it works. How can I assist you today?"
        }
      }
    }
...
```

Interestingly enough, all the tests pass and the app builds -- but `yarn tauri dev` specifically (and only that command) fails on Linux with:

```
error: linking with `cc` failed: exit status: 1
  |
  = note: LC_ALL="C" PATH="/root/.asdf/installs/rust/1.76.0/toolchains/1.76.0-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/bin:/root/.asdf/installs/rust/1.76.0/bin:/tmp/yarn--1716057597543-0.7563829...
  ...
  = note: /usr/bin/ld: /root/zamm/src-tauri/target/debug/deps/zamm-df7516eb9b93bda7.1xyldlavtwx51y4t.rcgu.o: in function `<diesel::expression::bound::Bound<T,U> as diesel::query_builder::QueryFragment<DB>>::walk_ast':
          /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/expression/bound.rs:40: undefined reference to `diesel::query_builder::ast_pass::AstPass<DB>::push_bind_param'
          /usr/bin/ld: /root/zamm/src-tauri/target/debug/deps/zamm-df7516eb9b93bda7.1xyldlavtwx51y4t.rcgu.o: in function `<diesel::expression::bound::Bound<T,U> as diesel::query_builder::QueryFragment<DB>>::walk_ast':
          /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/expression/bound.rs:40: undefined reference to `diesel::query_builder::ast_pass::AstPass<DB>::push_bind_param'
          /usr/bin/ld: /root/zamm/src-tauri/target/debug/deps/zamm-df7516eb9b93bda7.1xyldlavtwx51y4t.rcgu.o: in function `<diesel::expression::bound::Bound<T,U> as diesel::query_builder::QueryFragment<DB>>::walk_ast':
          /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/expression/bound.rs:40: undefined reference to `diesel::query_builder::ast_pass::AstPass<DB>::push_bind_param'
          /usr/bin/ld: /root/zamm/src-tauri/target/debug/deps/zamm-df7516eb9b93bda7.1xyldlavtwx51y4t.rcgu.o: in function `<diesel::expression::bound::Bound<T,U> as diesel::query_builder::QueryFragment<DB>>::walk_ast':
          /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.1.4/src/expression/bound.rs:40: undefined reference to `diesel::query_builder::ast_pass::AstPass<DB>::push_bind_param'
          /usr/bin/ld: /root/zamm/src-tauri/target/debug/deps/zamm-df7516eb9b93bda7: hidden symbol `_ZN6diesel13query_builder8ast_pass17AstPass$LT$DB$GT$15push_bind_param17h044cc0ebf0cc9718E' isn't defined
          /usr/bin/ld: final link failed: bad value
          collect2: error: ld returned 1 exit status
          
  = note: some `extern` functions couldn't be found; some native libraries may need to be installed or have their path specified
  = note: use the `-l` flag to specify native libraries to link
  = note: use the `cargo:rustc-link-lib` directive to specify the native libraries to link with Cargo (see https://doc.rust-lang.org/cargo/reference/build-scripts.html#cargorustc-link-libkindname)

error: could not compile `zamm` (bin "zamm") due to 1 previous error
```

This [question](https://stackoverflow.com/questions/70910000/rust-diesel-linking-with-cc-failed) seems related, but then again we already have `libsqlite3-dev` installed. This is failing on the `main` branch as well, so we try `make clean` and building everything from scratch. Sure enough, things work fine now.

#### Renaming to "follow up"

The name `llm_call_continuations` sounds more vague than `llm_call_follow_ups`. So we rename:

- `llm_call_continuations` to `llm_call_follow_ups`
- `llm_call_named_continuations` to `llm_call_named_follow_ups`
- `NewLlmCallContinuation` to `NewLlmCallFollowUp`
- `as_continuation` to `as_follow_up_row` (for additional clarity and consistency with `as_sql_row`)
- `insertable_call_continuations` to `insertable_call_follow_ups`
- rename the folder `src-tauri\migrations\2024-05-13-135101_add_api_call_continuations` to `src-tauri\migrations\2024-05-13-135101_add_api_call_follow_ups`

We are essentially relabeling what we used to call (in natural language, for this project) a "continuation" of an LLM call, to instead a "follow-up" to an LLM call.

We commit, and then check out the previous commit before running

```bash
$ diesel migration revert --database-url /root/.local/share/zamm/zamm.sqlite3 -n 1
```

because otherwise the `down.sql` script won't run correctly.

#### Discover existing follow-ups

We can go through the database and discover follow-up relations between existing API calls. We only want to do this the first time the user upgrades to zamm v0.1.4, but there doesn't appear to be attach Rust code to be triggered when a Diesel migration runs. As such, we settle for manually detecting the upgrade in the backend instead.

##### Detecting ZAMM version upgrades

We do

```bash
$ cargo add version-compare
```

We expose some functionality from the existing preference reading/writing code to other parts of the project. We edit `src-tauri\src\commands\preferences\write.rs` just to expose this function:

```ts
pub fn set_preferences_helper(
    ...
) -> ZammResult<()> {
    ...
}
```

We then do a little bit of a refactor in `src-tauri\src\commands\preferences\read.rs`, and expose the refactored function as well:

```rs
pub fn get_preferences_file_contents(
    ...
) -> ZammResult<Preferences> {
    ...
    if preferences_path.exists() {
        ...
        Ok(preferences)
    } else {
        ...
        Ok(Preferences::default())
    }
}

fn get_preferences_happy_path(
    maybe_preferences_dir: &Option<PathBuf>,
) -> ZammResult<Preferences> {
    #[allow(unused_mut)]
    let mut found_preferences = get_preferences_file_contents(maybe_preferences_dir)?;
    #[cfg(target_os = "windows")]
    if found_preferences.transparency_on.is_none() {
        found_preferences.transparency_on = Some(true);
    }
    Ok(found_preferences)
}

pub fn get_preferences_helper(...) -> Preferences {
    ...
}

```

Here `get_preferences_file_contents` is basically what `get_preferences_happy_path` was before, minus setting `transparency_on` to true for Windows. This is because we don't want to add settings to the local setttings file if the user hasn't explicitly changed them. Modifying preferences in this way with `get_preferences_happy_path` will result in losing information about what was set by the user and what was set by default.

We expose these new functions in `src-tauri\src\commands\preferences\mod.rs` as well:

```rs
pub use read::{..., get_preferences_file_contents};
pub use write::{..., set_preferences_helper};
```

and also expose that module for direct access in `src-tauri/src/commands/mod.rs`:

```rs
...
pub mod preferences;
...
```

We add a new field to `src-tauri/src/commands/preferences/models.rs`:

```rs
#[derive(...)]
pub struct Preferences {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub version: Option<String>,
    ...
}
```

We wish to keep the app upgrade and versioning logic contained to the app startup, so we avoid making any further changes here, such as setting a default for the version to the current Cargo version. This way, none of the tests for the other preference-setting API calls change, nor will they affect the version once the app is up and running and the user toggles some preference changes.

We create `src-tauri\src\upgrades.rs` to make use of these newly exposed functions, and overwrite the version if it's old:

```rs
use std::env;
use std::path::PathBuf;

use crate::commands::{
    errors::ZammResult,
    preferences::{get_preferences_file_contents, set_preferences_helper},
};

pub fn handle_app_upgrades(preferences_dir: &Option<PathBuf>) -> ZammResult<()> {
    let current_version = env!("CARGO_PKG_VERSION");
    let mut preferences = get_preferences_file_contents(preferences_dir)?;
    let last_touched_by_previous_zamm = match preferences.version {
        None => true,
        Some(pref_version) => version_compare::compare_to(
            pref_version,
            current_version,
            version_compare::Cmp::Lt,
        )
        .unwrap_or(false),
    };

    if last_touched_by_previous_zamm {
        preferences.version = Some(current_version.to_string());
        set_preferences_helper(preferences_dir, &preferences)?;
    }

    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sample_call::SampleCall;
    use crate::test_helpers::api_testing::standard_test_subdir;
    use crate::test_helpers::{
        SampleCallTestCase, SideEffectsHelpers, ZammResultReturn,
    };
    use stdext::function_name;

    struct HandleAppUpgradesTestCase {
        test_fn_name: &'static str,
    }

    impl SampleCallTestCase<(), ZammResult<()>> for HandleAppUpgradesTestCase {
        const EXPECTED_API_CALL: &'static str = "upgrade";
        const CALL_HAS_ARGS: bool = false;

        fn temp_test_subdirectory(&self) -> String {
            standard_test_subdir(Self::EXPECTED_API_CALL, self.test_fn_name)
        }

        async fn make_request(
            &mut self,
            _: &Option<()>,
            side_effects: &SideEffectsHelpers,
        ) -> ZammResult<()> {
            handle_app_upgrades(&side_effects.disk)
        }

        fn serialize_result(
            &self,
            sample: &SampleCall,
            result: &ZammResult<()>,
        ) -> String {
            ZammResultReturn::serialize_result(self, sample, result)
        }

        async fn check_result(
            &self,
            sample: &SampleCall,
            args: Option<&()>,
            result: &ZammResult<()>,
        ) {
            ZammResultReturn::check_result(self, sample, args, result).await
        }
    }

    impl ZammResultReturn<(), ()> for HandleAppUpgradesTestCase {}

    async fn check_app_upgrades(test_fn_name: &'static str, file_prefix: &str) {
        let mut test_case = HandleAppUpgradesTestCase { test_fn_name };
        test_case.check_sample_call(file_prefix).await;
    }

    #[tokio::test]
    async fn test_upgrade_first_init() {
        check_app_upgrades(
            function_name!(),
            "./api/sample-calls/upgrade-first-init.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_upgrade_from_v_0_1_3() {
        check_app_upgrades(
            function_name!(),
            "./api/sample-calls/upgrade-from-v0.1.3.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_upgrade_from_future_version() {
        check_app_upgrades(
            function_name!(),
            "./api/sample-calls/upgrade-from-future-version.yaml",
        )
        .await;
    }
}

```

Note that we're reusing the testing logic for API calls even though this is not an API call. Ideally, we should refactor API sample calls into the API request and response pair, and the backend interactions that make the request-response pair optional. But for now, we simply create `src-tauri/api/sample-calls/upgrade-first-init.yaml` to represent what happens on first app startup with the new version:

```yaml
request:
  - upgrade
response:
  message: "null"
sideEffects:
  disk:
    endStateDirectory: preferences/version-init

```

where `src-tauri\api\sample-disk-writes\preferences\version-init\preferences.toml` looks like:

```toml
version = "0.1.4"

```

and `src-tauri/api/sample-calls/upgrade-from-v0.1.3.yaml` to represent what happens when the user upgrades from v0.1.3 or before:

```yaml
request:
  - upgrade
response:
  message: "null"
sideEffects:
  disk:
    startStateDirectory: preferences/v0.1.3
    endStateDirectory: preferences/version-update
```

where `src-tauri\api\sample-disk-writes\preferences\v0.1.3\preferences.toml` looks like this (with an additional option added to simulate the user having changed some settings):

```toml
version = "0.1.3"
sound_on = false

```

and `src-tauri\api\sample-disk-writes\preferences\version-update\preferences.toml` looks like this (with the previous setting preserved):

```toml
sound_on = false
version = "0.1.4"

```

Finally, we create `src-tauri/api/sample-calls/upgrade-from-future-version.yaml`, which demonstrates that files should be untouched if it's a future version:

```yaml
request:
  - upgrade
response:
  message: "null"
sideEffects:
  disk:
    startStateDirectory: preferences/future-version
    endStateDirectory: preferences/future-version

```

where `src-tauri\api\sample-disk-writes\preferences\future-version\preferences.toml` looks like this (valid for our purposes until ZAMM gets past version 1.9.9):

```toml
version = "1.9.9"

```

We then edit `src-tauri\src\main.rs` based on the examples [here](https://github.com/tauri-apps/tauri/discussions/5230) and [here](https://github.com/tauri-apps/tauri/discussions/6583) to call this new function:

```rs
...
mod upgrades;
...

use upgrades::handle_app_upgrades;
...

fn main() {
    ...
            tauri::Builder::default()
                .setup(|app| {
                    let config_dir = app.handle().path_resolver().app_config_dir();
                    handle_app_upgrades(&config_dir)?;
                    Ok(())
                })
                ...;
    ...
}
```

##### Figuring out follow-up API calls

We already have a database of API calls from previous version. We'd like to add the missing links in now. Our first attempt passing the database in directly gives us this problem:

```
error[E0373]: closure may outlive the current function, but it borrows `possible_db`, which is owned by the current function
  --> src\main.rs:64:24
   |
64 |                 .setup(|app| {
   |                        ^^^^^ may outlive borrowed value `possible_db`
65 |                     let config_dir = app.hand...
66 |                     match possible_db.as_mut() {
   |                           ----------- `possible_db` is borrowed here
   |
note: function requires argument type to outlive `'static`
  --> src\main.rs:63:13
   |
63 | / ...   tauri::Builder::default()
64 | | ...       .setup(|app| {
65 | | ...           let config_dir = app.handle().path_resolver... 
66 | | ...           match possible_db.as_mut() {
...  |
77 | | ...           Ok(())
78 | | ...       })
   | |____________^
help: to force the closure to take ownership of `possible_db` (and any other referenced variables), use the `move` keyword
   |
64 |                 .setup(move |app| {
   |                        ++++

error[E0505]: cannot move out of `possible_db` because it is borrowed
  --> src\main.rs:79:49
   |
63 | /             tauri::Builder::default()
64 | |                 .setup(|app| {
   | |                        ----- borrow of `possible_db` occurs here
65 | |                     let config_dir = app.handle().path_resolve...
66 | |                     match possible_db.as_mut() {
   | |                           ----------- borrow occurs due to use in closure
...  |
77 | |                     Ok(())
78 | |                 })
   | |__________________- argument requires that `possible_db` is borrowed for `'static`
79 |                   .manage(ZammDatabase(Mutex::new(possible_db...
   |                                                   ^^^^^^^^^^^ move out of `possible_db` occurs here
```

Instead, we try getting the app state as shown [here](https://github.com/tauri-apps/tauri/discussions/4202#discussioncomment-2813507), but then we find that we need `async` due to needing to wait for the DB to lock, so we try the solution [here](https://github.com/tauri-apps/tauri/discussions/7596#discussioncomment-6718895). But then we get

```
error[E0521]: borrowed data escapes outside of closure
  --> src\main.rs:69:21
   |
65 |                   .setup(|app| {
   |                           ---
   |                           |
   |                           `app` is a reference that is only valid in the closure body
   |                           has type `&'1 mut tauri::App`        
...
69 | /                     tauri::async_runtime::spawn(async move { 
70 | |                         match db.0.lock().await.as_mut() {   
71 | |                             None => {
72 | |                                 eprintln!("Couldn't conne... 
...  |
80 | |                         }
81 | |                     });
   | |                      ^
   | |                      |
   | |______________________`app` escapes the closure body here     
   |                        argument requires that `'1` must outlive `'static`
```

It's unclear if it's possible for the `'_` lifetime of the state to live on like this, so instead we try a simpler way to [block on](https://stackoverflow.com/a/52521592) async tasks, turning them into sync tasks. We end up with a `src-tauri\src\main.rs` that looks like:

```rs
...
use futures::executor;
...
use tauri::Manager;
...

fn main() {
    ...
            tauri::Builder::default()
                .setup(|app| {
                    let config_dir = app.handle().path_resolver().app_config_dir();
                    let zamm_db = app.state::<ZammDatabase>();

                    executor::block_on(async {
                        handle_app_upgrades(&config_dir, &zamm_db)
                            .await
                            .unwrap_or_else(|e| {
                                eprintln!("Couldn't run custom data migrations: {e}");
                                eprintln!("Continuing with unchanged data");
                            });
                    });

                    Ok(())
                })
                .manage(ZammDatabase(Mutex::new(possible_db)))
                .manage(ZammApiKeys(Mutex::new(api_keys)))
                ...;
}
```

Then our `src-tauri\src\upgrades.rs` look like:

```rs
...
use anyhow::anyhow;
use chrono::NaiveDateTime;
use diesel::dsl::not;
use diesel::prelude::*;

use crate::models::llm_calls::ChatPrompt;
use crate::models::llm_calls::EntityId;
use crate::models::llm_calls::Prompt;
use crate::schema::{llm_call_follow_ups, llm_calls};
use crate::ZammDatabase;

async fn upgrade_to_v_0_1_4(zamm_db: &ZammDatabase) -> ZammResult<()> {
    let mut db = zamm_db.0.lock().await;
    let conn = db.as_mut().ok_or(anyhow!("Failed to lock database"))?;

    let llm_calls_without_followups: Vec<(EntityId, NaiveDateTime, Prompt)> =
        llm_calls::table
            .select((llm_calls::id, llm_calls::timestamp, llm_calls::prompt))
            .filter(not(llm_calls::id.eq_any(
                llm_call_follow_ups::table.select(llm_call_follow_ups::next_call_id),
            )))
            .load::<(EntityId, NaiveDateTime, Prompt)>(conn)?;
    let non_initial_calls: Vec<&(EntityId, NaiveDateTime, Prompt)> =
        llm_calls_without_followups
            .iter()
            .filter(|(_, _, prompt)| {
                match prompt {
                    // if it's just system message + initial human message, then this
                    // must be an initial prompt rather than any previous one.
                    // Otherwise, it should have at least 4 messages: the initial system
                    // message, the initial human message, the initial AI reply, and
                    // the human follow-up
                    Prompt::Chat(chat_prompt) => chat_prompt.messages.len() >= 4,
                }
            })
            .collect();

    for (id, timestamp, prompt) in non_initial_calls {
        let (search_prompt, search_completion) = match prompt {
            Prompt::Chat(chat_prompt) => {
                let length = chat_prompt.messages.len();
                let previous_messages = chat_prompt.messages[..length - 2].to_vec();
                let previous_prompt = Prompt::Chat(ChatPrompt {
                    messages: previous_messages,
                });
                let previous_completion = &chat_prompt.messages[length - 2];
                (previous_prompt, previous_completion)
            }
        };

        let prior_api_call: Option<EntityId> = llm_calls::table
            .select(llm_calls::id)
            .filter(llm_calls::timestamp.lt(timestamp))
            .filter(llm_calls::prompt.eq(search_prompt))
            .filter(llm_calls::completion.eq(search_completion))
            .first::<EntityId>(conn)
            .optional()?;

        if let Some(prior_call_id) = prior_api_call {
            diesel::insert_into(llm_call_follow_ups::table)
                .values((
                    llm_call_follow_ups::previous_call_id.eq(prior_call_id),
                    llm_call_follow_ups::next_call_id.eq(id),
                ))
                .execute(conn)?;
        }
    }

    Ok(())
}

fn version_before(a: &Option<String>, b: &str) -> bool {
    match a {
        None => true,
        Some(version) => {
            version_compare::compare_to(version, b, version_compare::Cmp::Lt)
                .unwrap_or(false)
        }
    }
}

pub async fn handle_app_upgrades(
    preferences_dir: &Option<PathBuf>,
    zamm_db: &ZammDatabase,
) -> ZammResult<()> {
    ...

    if version_before(&preferences.version, "0.1.4") {
        upgrade_to_v_0_1_4(zamm_db).await?;
    }

    if version_before(&preferences.version, current_version) {
        preferences.version = ...;
        ...
    }

    Ok(())
}

#[cfg(test)]
mod tests {
    ...

    impl SampleCallTestCase<(), ZammResult<()>> for HandleAppUpgradesTestCase {
        ...

        async fn make_request(
            ...
        ) -> ZammResult<()> {
            handle_app_upgrades(&side_effects.disk, side_effects.db.as_ref().unwrap())
                .await
        }

        ...
    }

    ...

    #[tokio::test]
    async fn test_upgrade_to_v_0_1_4_onwards() {
        check_app_upgrades(
            function_name!(),
            "./api/sample-calls/upgrade-to-v0.1.4.yaml",
        )
        .await;
    }

    ...
}

```

where `src-tauri/api/sample-calls/upgrade-to-v0.1.4.yaml` (renamed from `src-tauri/api/sample-calls/upgrade-from-v0.1.3.yaml`) now looks like

```yaml
request:
  - upgrade
response:
  message: "null"
sideEffects:
  disk:
    startStateDirectory: preferences/sound-override
    endStateDirectory: preferences/version-update
  database:
    startStateDump: sample-v0.1.3-db
    endStateDump: sample-v0.1.4-db

```

As for `src-tauri\api\sample-database-writes\sample-v0.1.3-db\dump.yaml`, we start it off with our usual duo, stripped of any links:

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

Then we add a mock AI call:

```yaml
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a00
  timestamp: 2024-01-16T08:00:50.738093890
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
```

Then mock call #2 is a follow-up to the first mock call:

```yaml
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a02
  timestamp: 2024-01-18T08:02:50.738093890
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
      - role: AI
        text: Mocking number 0.
      - role: Human
        text: This is a mock response.
    temperature: 1.0
  response:
    completion:
      role: AI
      text: Mocking number 2.
  tokens:
    prompt: 15
    response: 3
    total: 18
```

and then mock call #4 is one that doesn't match any prior history. This shouldn't happen, but in case it does, the program shouldn't crash:

```yaml
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a04
  timestamp: 2024-01-18T08:04:50.738093890
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
        text: This is a non-existent fluke with no history.
      - role: AI
        text: Mocking number 0.
      - role: Human
        text: This is another mock response.
    temperature: 1.0
  response:
    completion:
      role: AI
      text: Mocking number 4.
  tokens:
    prompt: 15
    response: 3
    total: 18
```

Then, mock call #6 is a *second* follow-up to call #2, but one that is finally non-sequential (for example, if someone restores a conversation and starts chatting from there):

```yaml
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a06
  timestamp: 2024-01-18T08:06:50.738093890
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
      - role: AI
        text: Mocking number 0.
      - role: Human
        text: This is another mock response.
    temperature: 1.0
  response:
    completion:
      role: AI
      text: Mocking number 6.
  tokens:
    prompt: 15
    response: 3
    total: 18
```

The corresponding SQL at `src-tauri\api\sample-database-writes\sample-v0.1.3-db\dump.sql`:

```sql
INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a74','2024-01-16 08:50:19.738093890','open_ai','gpt-4','gpt-4-0613',1.0,32,12,44,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"Hello, does this work?"}]}','{"role":"AI","text":"Yes, it works. How can I assist you today?"}');
INSERT INTO llm_calls VALUES('c13c1e67-2de3-48de-a34c-a32079c03316','2024-01-16 09:50:19.738093890','open_ai','gpt-4','gpt-4-0613',1.0,57,22,79,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"Hello, does this work?"},{"role":"AI","text":"Yes, it works. How can I assist you today?"},{"role":"Human","text":"Tell me something funny."}]}','{"role":"AI","text":"Sure, here''s a joke for you: Why don''t scientists trust atoms? Because they make up everything!"}');
INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a00','2024-01-18 08:00:50.738093890','open_ai','gpt-4','gpt-4-0613',1.0,15,3,18,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"This is a mock conversation."}]}','{"role":"AI","text":"Mocking number 0."}');
INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a02','2024-01-18 08:02:50.738093890','open_ai','gpt-4','gpt-4-0613',1.0,15,3,18,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"This is a mock conversation."},{"role":"AI","text":"Mocking number 0."},{"role":"Human","text":"This is a mock response."}]}','{"role":"AI","text":"Mocking number 2."}');
INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a04','2024-01-18 08:04:50.738093890','open_ai','gpt-4','gpt-4-0613',1.0,15,3,18,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"This is a non-existent fluke with no history."},{"role":"AI","text":"Mocking number 0."},{"role":"Human","text":"This is another mock response."}]}','{"role":"AI","text":"Mocking number 4."}');
INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a06','2024-01-18 08:06:50.738093890','open_ai','gpt-4','gpt-4-0613',1.0,15,3,18,'{"type":"Chat","messages":[{"role":"System","text":"You are ZAMM, a chat program. Respond in first person."},{"role":"Human","text":"This is a mock conversation."},{"role":"AI","text":"Mocking number 0."},{"role":"Human","text":"This is another mock response."}]}','{"role":"AI","text":"Mocking number 6."}');
```

gives us confidence that Serde serialization is consistent enough for us to simply query for the serialized prompt and completion text in SQL itself without needing to resort to higher-level parsing in Rust.

`src-tauri\api\sample-database-writes\sample-v0.1.4-db\dump.sql` looks similar, except as expected with the link rows add at the end:

```sql
INSERT INTO llm_calls VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a74', ...);
...
INSERT INTO llm_call_follow_ups VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a74','c13c1e67-2de3-48de-a34c-a32079c03316');
INSERT INTO llm_call_follow_ups VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a00','d5ad1e49-f57f-4481-84fb-4d70ba8a7a02');
INSERT INTO llm_call_follow_ups VALUES('d5ad1e49-f57f-4481-84fb-4d70ba8a7a00','d5ad1e49-f57f-4481-84fb-4d70ba8a7a06');
```

Similarly, with `src-tauri\api\sample-database-writes\sample-v0.1.4-db\dump.yaml` we recover the `continue-conversation` dump, plus the other mock calls we made:

```yaml
api_keys: []
llm_calls:
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
  ...
  conversation:
    next_calls:
    - id: c13c1e67-2de3-48de-a34c-a32079c03316
      snippet: 'Sure, here''s a joke for you: Why don''t scientists trust...'
- id: c13c1e67-2de3-48de-a34c-a32079c03316
  ...
  conversation:
    previous_call:
      id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a74
      snippet: Yes, it works. How can I assist you today?
- id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a00
  ...
  conversation:
    next_calls:
    - id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a02
      snippet: Mocking number 2.
    - id: d5ad1e49-f57f-4481-84fb-4d70ba8a7a06
      snippet: Mocking number 6.
...
```

Because the test function takes in a database now, we edit `src-tauri/api/sample-calls/upgrade-first-init.yaml` to ensure that this does not interfere with a fresh install of ZAMM:

```yaml
...
sideEffects:
  ...
  database:
    startStateDump: empty
    endStateDump: empty

```

Similarly with `src-tauri\api\sample-calls\upgrade-from-future-version.yaml`, we ensure that this data migration is not run if we encounter a *newer* version of the database (for example, future versions of ZAMM might enable manually deleting such links):

```yaml
...
sideEffects:
  ...
  database:
    startStateDump: sample-v0.1.3-db
    endStateDump: sample-v0.1.3-db

```

Finally, as a bit of extra information, we edit `src-tauri\src\upgrades.rs` to print out any results of this migration:

```rs
async fn upgrade_to_v_0_1_4(...) -> ZammResult<()> {
    ...
    let mut num_links_added = 0;
    for (id, timestamp, prompt) in non_initial_calls {
        ...

        if let Some(prior_call_id) = prior_api_call {
            diesel::insert_into(...)
                ...;
            num_links_added += 1;
        }
    }

    if num_links_added > 0 {
        println!(
            "v0.1.4 data migration: Linked {} LLM API calls with their follow-ups",
            num_links_added
        );
    }

    Ok(())
}
```

We can use the `--show-output` option of `cargo test` to check that this output works.

#### Always returning follow-ups in the same order

We notice one fluke failure in our Rust tests on CI because the database dump returned the `next_calls` in this order:

```yaml
    next_calls:
    - id: 63b5c02e-b864-4efe-a286-fbef48b152ef
      snippet: 'Sure, here is a simple Rust program that prints out the joke: ```rust fn main() { println!("Why don''t scientists trust...'
    - id: 0e6bcadf-2b41-43d9-b4cf-81008d4f4771
      snippet: 'Sure, here is a simple Python script that will print out the joke: ```python print("Why don''t scientists trust atoms? Because...'
```

To prevent this, we do

```bash
$ diesel migration generate ordered_names                                 
Creating migrations\2024-05-29-071520_ordered_names\up.sql
Creating migrations\2024-05-29-071520_ordered_names\down.sql
```

Our `src-tauri\migrations\2024-05-29-071520_ordered_names\down.sql` will just restore the view to its previous version:

```sql
DROP VIEW llm_call_named_follow_ups;
CREATE VIEW llm_call_named_follow_ups AS
  SELECT
    llm_call_follow_ups.previous_call_id AS previous_call_id,
    previous_call.completion AS previous_call_completion,
    llm_call_follow_ups.next_call_id AS next_call_id,
    next_call.completion AS next_call_completion
  FROM
    llm_call_follow_ups
    JOIN llm_calls AS previous_call ON llm_call_follow_ups.previous_call_id = previous_call.id
    JOIN llm_calls AS next_call ON llm_call_follow_ups.next_call_id = next_call.id;

```

whereas our `src-tauri\migrations\2024-05-29-071520_ordered_names\up.sql` will add an extra `ORDER BY` clause to remove such uncertainties:

```sql
DROP VIEW llm_call_named_follow_ups;
CREATE VIEW llm_call_named_follow_ups AS
  SELECT
    ...
  FROM
    ...
  ORDER BY next_call.timestamp ASC;

```

### Frontend

We update `src-svelte/src/lib/bindings.ts` automatically with Specta, and then start fixing failing tests. We edit `src-svelte/src/routes/chat/Chat.svelte` to pass in the last message ID to the backend whenever possible, and put all the arguments to `chat` inside a proper object:

```svelte
<script lang="ts" context="module">
  ...

  export const lastMessageId = writable<string | undefined>(undefined);
  ...

  export function resetConversation() {
    lastMessageId.set(undefined);
    ...
  }
</script>

<script lang="ts">
  ...

  async function sendChatMessage(message: string) {
    ...
    try {
      let llmCall = await chat({
        provider: "OpenAI",
        llm: "gpt-4",
        temperature: null,
        previous_call_id: $lastMessageId,
        prompt: $conversation,
      });
      lastMessageId.set(llmCall.id);
      ...
    } ...
  }
  
  ...
</script>

...
```

We edit `src-svelte/src/routes/chat/Chat.test.ts`, where we can now cast directly to `ChatArgs` instead of `ChatMessage`. We need to copy the changes a second time to the `"won't send multiple messages at once"` test because it is based on the above with modifications.

```ts
...
import Chat, {
  ...,
  lastMessageId,
  ...,
} from "./Chat.svelte";
...
import type { ChatArgs, ... } from "$lib/bindings";

...
  async function sendChatMessage(
    ...
  ) {
    ...
    const nextExpectedCallArgs = nextExpectedApiCall.request[1] as unknown as {
      args: ChatArgs;
    };
    const nextExpectedMessage =
      nextExpectedCallArgs.args["prompt"].slice(-1)[0];
    ...
  }

  ...

  test("won't send multiple messages at once", async () => {
    ...
    const nextExpectedCallArgs = nextExpectedApiCall.request[1] as unknown as {
      args: ChatArgs;
    };
    const nextExpectedMessage =
      nextExpectedCallArgs.args["prompt"].slice(-1)[0];
    ...
  });

  ...

  test("listens for updates to conversation store", async () => {
    ...
    lastMessageId.set("d5ad1e49-f57f-4481-84fb-4d70ba8a7a74");
    ...
  });
```

We need to manually edit `src-svelte/src/lib/bindings.ts` here because it generates

```ts
export type EntityId = { uuid: string }
```

which then produces the Svelte check error

```
c:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\src-svelte\src\routes\chat\Chat.svelte:90:25
Error: Argument of type 'EntityId' is not assignable to parameter of type 'string'. (ts)
      });
      lastMessageId.set(llmCall.id);
      appendMessage(llmCall.response_message);
```

As such, we manually edit that line right now to be

```ts
export type EntityId = string
```

and create [this bug](https://github.com/oscartbeaumont/tauri-specta/issues/93) to notify the maintainer of this issue in case they aren't already aware.

The API call tests are failing too now, so we edit `src-svelte/src/routes/api-calls/[slug]/Actions.svelte`:

```ts
  ...
  import { lastMessageId, conversation } from "../../chat/Chat.svelte";
  ...

  function restoreConversation() {
    ...

    lastMessageId.set(apiCall.id);
    ...
  }

```

We also fix the tests at `src-svelte\src\routes\api-calls\[slug]\ApiCall.test.ts`, to check that the "Restore Conversation" button properly restores everything about a conversation:

```ts
...
import {
  ...,
  lastMessageId,
} from "../../chat/Chat.svelte";
...

  test("can restore chat conversation", async () => {
    ...
    expect(get(conversation)).toEqual(...);
    expect(get(lastMessageId)).toBeUndefined();
    ...

    await waitFor(() => {
      expect(get(conversation)).toEqual(...);
    });
    expect(get(lastMessageId)).toEqual("c13c1e67-2de3-48de-a34c-a32079c03316");
    ...
```

While fixing the tests, we encountered the error `"message.includes" is not a function`, which [appears](https://stackoverflow.com/a/78389207) to be thrown when we try to handle an `Error` instead of an error message with the snackbar. As such, we also edit `src-svelte/src/lib/snackbar/Snackbar.svelte` to set the `msg` to a string before anything else happens:

```svelte
  export function snackbarError(error: string | Error) {
    const msg = error instanceof Error ? error.message : error;

    ...
  }
```

#### Manual specta overwrite

Due to having to manually override Specta, we will edit `main.rs` to not output the bindings unless we tell it to. We use `clap` to handle commandline arguments:

```bash
$ cargo add clap --features derive
```

We create `src-tauri/src/cli.rs` with a very barebones configuration, adding documentation because command descriptions in the help messages are [based on](https://github.com/clap-rs/clap/discussions/1619) code documentation:

```rs
use clap::{Parser, Subcommand};

#[derive(Subcommand)]
pub enum Commands {
    /// Run the GUI. This is the default command.
    Gui {},
    /// Export Specta bindings for development purposes
    #[cfg(debug_assertions)]
    ExportBindings {},
}

/// Zen and the Automation of Metaprogramming for the Masses
/// 
/// This is an experimental tool meant for automating programming-related activities,
/// although none have been implemented yet. Blog posts on progress can be found at
/// https://zamm.dev/
#[derive(Parser)]
#[command(name = "zamm")]
#[command(version)]
#[command(
    about = "Zen and the Automation of Metaprogramming for the Masses"
)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Option<Commands>,
}

```

We modify `src-tauri\src\main.rs` to allow even Windows to export bindings, but only ever if it's explicitly requested:

```rs
...
mod cli;
...
use clap::Parser;
...
#[cfg(debug_assertions)]
use specta::collect_types;
#[cfg(debug_assertions)]
use tauri_specta::ts;
...
use cli::{Cli, Commands};
...

fn main() {
    let cli = Cli::parse();

    match &cli.command {
        #[cfg(debug_assertions)]
        Some(Commands::ExportBindings {}) => {
            ts::export(
                collect_types![
                    get_api_keys,
                    ...
                ],
                "../src-svelte/src/lib/bindings.ts",
            )
            .expect("Failed to export Specta bindings");
            println!("Specta bindings should be exported to ../src-svelte/src/lib/bindings.ts");
        },
        Some(Commands::Gui {}) | None => {
            let mut possible_db = setup::get_db();
            let api_keys = setup_api_keys(&mut possible_db);

            tauri::Builder::default()
                .manage(ZammDatabase(Mutex::new(possible_db)))
                .manage(ZammApiKeys(Mutex::new(api_keys)))
                .invoke_handler(tauri::generate_handler![
                    get_api_keys,
                    ...
                ])
                .run(tauri::generate_context!())
                .expect("error while running tauri application");
        }
    }
}

```

Now we can remove `src-svelte/src/lib/bindings.ts` from `.prettierignore`. In fact, since that's the only entry there, we can remove the file altogether.

##### Specta upgrade

We try upgrading to Specta 2, which should fix this issue, but is still in beta. We edit `src-tauri/Cargo.toml` to pin to a specific version of Specta:

```toml
specta = { version = "=2.0.0-rc.12", features = [...] }
```

Next, we edit `src-tauri/src/main.rs`, which must now use `specta::collect_functions` instead of `specta::collect_types`. Now we run into the issue

```
error[E0277]: the trait bound `tauri::State<'_, ZammDatabase>: Type` is not satisfied
   --> src/commands/llms/get_api_calls.rs:30:1
    |
30  |   #[specta]
    |   ^^^^^^^^^ the trait `Type` is not implemented for `tauri::State<'_, ZammDatabase>`
    |
   ::: src/main.rs:40:17
    |
40  | /                 collect_functions![
41  | |                     get_api_keys,
42  | |                     set_api_key,
43  | |                     play_sound,
...   |
49  | |                     get_api_calls,
50  | |                 ],
    | |_________________- in this macro invocation
    |
    = help: the following other types implement trait `Type`:
              bool
              char
              isize
              i8
              i16
              i32
              i64
              i128
            and 118 others
    = note: required for `tauri::State<'_, ZammDatabase>` to implement `FunctionArg`
    = note: required for `fn(State<'_, ZammDatabase>, i32) -> ...` to implement `specta::function::Function<_>`
    = note: the full type name has been written to '/root/zamm/src-tauri/target/debug/deps/zamm-3960311a1ba30926.long-type-9008942556723932530.txt'
note: required by a bound in `get_fn_datatype`
   --> /root/.asdf/installs/rust/1.76.0/registry/src/index.crates.io-6f17d22bba15001f/specta-2.0.0-rc.12/src/internal.rs:232:40
    |
232 |     pub fn get_fn_datatype<TMarker, T: Function<TMarker>>(
    |                                        ^^^^^^^^^^^^^^^^^ required by this bound in `get_fn_datatype`
    = note: this error originates in the macro `__specta__fn__get_api_calls` which comes from the expansion of the macro `collect_functions` (in Nightly builds, run with -Z macro-backtrace for more info)

Some errors have detailed explanations: E0277, E0308.
For more information about an error, try `rustc --explain E0277`.
```

Perhaps we need tauri-specta v2 to work with specta v2, but if we specify

```toml
tauri-specta = { version = "1.0.2", features = [...] }
```

then we have

```
error: failed to select a version for `webkit2gtk-sys`.
    ... required by package `webkit2gtk v0.18.2`
    ... which satisfies dependency `webkit2gtk = "^0.18.2"` of package `tauri v1.4.0`
    ... which satisfies dependency `tauri = "^1.4"` of package `zamm v0.1.4 (/root/zamm/src-tauri)`
versions that meet the requirements `^0.18` are: 0.18.0

the package `webkit2gtk-sys` links to the native library `web_kit2`, but it conflicts with a previous package which links to `web_kit2` as well:
package `webkit2gtk-sys v2.0.1`
    ... which satisfies dependency `ffi = "^2.0.1"` of package `webkit2gtk v2.0.1`
    ... which satisfies dependency `webkit2gtk = "=2.0.1"` of package `tauri v2.0.0-beta.19`
    ... which satisfies dependency `tauri = "=2.0.0-beta.19"` of package `tauri-specta v2.0.0-rc.10`
    ... which satisfies dependency `tauri-specta = "=2.0.0-rc.10"` of package `zamm v0.1.4 (/root/zamm/src-tauri)`
Only one package in the dependency graph may specify the same links value. This helps ensure that only one copy of a native library is linked in the final binary. Try to adjust your dependencies so that only one package uses the `links = "web_kit2"` value. For more information, see https://doc.rust-lang.org/cargo/reference/resolver.html#links.

failed to select a version for `webkit2gtk-sys` which could resolve this conflict
```

As such, we'll stick with Specta 1 and the manual override for now.

#### Displaying the new data

##### Getting test data

When we eventually evalute the visualization of this data, we'd like to have a visual test case where the API call both is a follow-up, and has more than one follow-up to itself. To make sure that we are testing with realistic data, we add some extra tests to `src-tauri\src\commands\llms\chat.rs`:

```rs
  #[tokio::test]
    async fn test_fork_conversation_step_1() {
        test_llm_api_call(
            function_name!(),
            "api/sample-calls/chat-fork-conversation-python.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_fork_conversation_step_2() {
        test_llm_api_call(
            function_name!(),
            "api/sample-calls/chat-fork-conversation-rust.yaml",
        )
        .await;
    }
```

where `src-tauri/api/sample-calls/chat-fork-conversation-python.yaml` looks like

```yaml
request:
  - chat
  - >
    {
      "args": {
        "provider": "OpenAI",
        "llm": "gpt-4",
        "temperature": null,
        "previous_call_id": "c13c1e67-2de3-48de-a34c-a32079c03316",
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
          },
          {
            "role": "AI",
            "text": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
          },
          {
            "role": "Human",
            "text": "Write me a Python script that prints that joke out."
          }
        ]
      }
    }
response:
  message: >
    {
      "id": "0e6bcadf-2b41-43d9-b4cf-81008d4f4771",
      "timestamp": "2024-05-23T09:30:37.854241700",
      "response_message": {
        "role": "AI",
        "text": "Sure, here is a simple Python script that will print out the joke:\n\n```python\nprint(\"Why don't scientists trust atoms? Because they make up everything!\")\n```\n\nJust run this script and it will display the joke."
      }
    }
sideEffects:
  database:
    startStateDump: conversation-continued
    endStateDump: conversation-forked-step-1
  network:
    recordingFile: fork-conversation-python.json

```

and `src-tauri/api/sample-calls/chat-fork-conversation-rust.yaml` is the same thing but continuing on from the previous database state, and with a different user response to `c13c1e67-2de3-48de-a34c-a32079c03316`:

```yaml
request:
  - chat
  - >
    {
      "args": {
        "provider": "OpenAI",
        "llm": "gpt-4",
        "temperature": null,
        "previous_call_id": "c13c1e67-2de3-48de-a34c-a32079c03316",
        "prompt": [
          {
            "role": "System",
            "text": "You are ZAMM, a chat program. Respond in first person."
          },
          ...,
          {
            "role": "AI",
            "text": "Sure, here's a joke for you: Why don't scientists trust atoms? Because they make up everything!"
          },
          {
            "role": "Human",
            "text": "Write me a Rust script that prints that joke out."
          }
        ]
      }
    }
response:
  message: >
    {
      "id": "63b5c02e-b864-4efe-a286-fbef48b152ef",
      "timestamp": "2024-05-23T09:34:38.572764500",
      "response_message": {
        "role": "AI",
        "text": "Sure, here is a simple Rust program that prints out the joke:\n\n```rust\nfn main() {\n    println!(\"Why don't scientists trust atoms? Because they make up everything!\");\n}\n```\nTo run this program, you'd simply compile and run the Rust file containing this code."
      }
    }
sideEffects:
  database:
    startStateDump: conversation-forked-step-1
    endStateDump: conversation-forked-step-2
  network:
    recordingFile: fork-conversation-rust.json

```

The response messages are copied over from the output of the failing tests.

Since the backend testing infrastructure does not yet support simulating multiple API calls at once and testing their effects, this is the easiest way to get realistic test data on both calls.

The contents of `src-tauri/api/sample-database-writes/conversation-forked-step-1/` and `src-tauri/api/sample-database-writes/conversation-forked-step-2` are simply copied from the temporary test directories when the tests failed, with a manual sanity check to see that the results look sane. `src-tauri/api/sample-network-requests/fork-conversation-python.json` and `src-tauri/api/sample-network-requests/fork-conversation-rust.json` are automatically captured. In the future, it would be a nice TODO to have the backend test infrastructure also automatically capture empty database directories.

We realize that we actually still need to update `src-tauri\api\sample-calls\upgrade-from-future-version.yaml` to use the new database dump, and to update the call accordingly:

```yaml
...
response:
  message: >
    {
      "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
      ...
      "conversation": {
        "previous_call": {
          "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
          ...
        },
        "next_calls": [
          {
            "id": "0e6bcadf-2b41-43d9-b4cf-81008d4f4771",
            "snippet": "Sure, here is a simple Python script that will print..."
          },
          {
            "id": "63b5c02e-b864-4efe-a286-fbef48b152ef",
            "snippet": "Sure, here is a simple Rust program that prints out..."
          }
        ]
      }
    }
sideEffects:
  database:
    startStateDump: conversation-forked-step-2
    endStateDump: conversation-forked-step-2

```

We edit `src-tauri\api\sample-calls\get_api_call-start-conversation.yaml` to also use `conversation-forked-step-2`, but this of course results in no changes to that particular API call because the only new follow up links don't affect the API call in question. We also realize that there needs to be an example *without* any conversation links, so we create `src-tauri\api\sample-calls\get_api_call-no-links.yaml` using an earlier sample database snapshot:

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
    startStateDump: conversation-started
    endStateDump: conversation-started

```

and add it as a test to `src-tauri\src\commands\llms\get_api_call.rs`:

```rs
    #[tokio::test]
    async fn test_get_api_call_no_links() {
        check_get_api_call_sample(
            function_name!(),
            "./api/sample-calls/get_api_call-no-links.yaml",
        )
        .await;
    }
```

We actually need to also copy-paste the contents of `src-tauri\api\sample-calls\get_api_call-continue-conversation.yaml` into the `CONTINUE_CONVERSATION_CALL` variable in `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.stories.ts`. Ideally this could either be automated, or the stories could be split between ones that make actual API calls and ones that just have them defined.

##### Displaying the test data

We find that at first, the SVGs for the left and right arrows are differently sized for some reason. This gets fixed when we wrap the SVGs in their own divs first.

We also try to clamp the text to two lines at most. This is when we [find out](https://css-tricks.com/almanac/properties/l/line-clamp/) that the way to do so in CSS is by exactly repeating these four lines:

```css
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;  
  overflow: hidden;
```

As such, we finally end up with a `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.svelte` that looks like this:

```svelte
<script lang="ts">
  ...
  import IconLeftArrow from "~icons/mingcute/left-fill";
  import IconRightArrow from "~icons/mingcute/right-fill";
  ...

  $: previousCall = apiCall?.conversation?.previous_call;
  $: nextCalls = apiCall?.conversation?.next_calls ?? [];
</script>

<InfoBox title="API Call">
  {#if apiCall}
    ...
    {#if apiCall?.conversation !== undefined}
      <SubInfoBox subheading="Conversation">
        <div class="conversation-links composite-reveal">
          <div class="conversation previous-links composite-reveal">
            {#if previousCall !== undefined}
              <a
                class="conversation link previous atomic-reveal"
                href={`/api-calls/${previousCall?.id}`}
              >
                <div class="arrow-icon"><IconLeftArrow /></div>
                <div class="snippet">{previousCall?.snippet}</div>
              </a>
            {/if}
          </div>

          <div class="conversation next-links composite-reveal">
            {#each nextCalls as nextCall}
              <a
                class="conversation link next atomic-reveal"
                href={`/api-calls/${nextCall.id}`}
              >
                <div class="snippet">{nextCall.snippet}</div>
                <div class="arrow-icon"><IconRightArrow /></div>
              </a>
            {/each}
          </div>
        </div>
      </SubInfoBox>
    {/if}
  {:else}
    ...
  {/if}
</InfoBox>

<style>
  ...

  .conversation-links {
    display: flex;
    flex-direction: row;
    gap: 1rem;
  }

  .conversation.previous-links,
  .conversation.next-links {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
  }

  .conversation.link {
    border: 1px solid var(--color-border);
    padding: 0.75rem;
    border-radius: var(--corner-roundness);
    color: black;
    display: flex;
    flex-direction: row;
    gap: 0.75rem;
    align-items: center;
  }

  .conversation.link .arrow-icon {
    display: block;
    margin-top: 0.3rem;
  }

  .conversation.link .arrow-icon :global(svg) {
    transform: scale(1.5);
    color: var(--color-faded);
  }

  .conversation.link .snippet {
    text-align: start;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    line-clamp: 2;
    text-overflow: ellipsis;
    overflow: hidden;
  }
</style>
```

We edit `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.stories.ts` to include a new example that showcases links that would span more than one line:

```ts
...

const LOTS_OF_CODE_CALL = {
  id: "9857257b-8e17-4203-91eb-c10bef8ff4e6",
  timestamp: "2024-05-18T03:21:39.637157100",
  llm: {
    name: "gpt-4-0613",
    provider: "OpenAI",
    requested: "gpt-4",
  },
  request: {
    prompt: {
      type: "Chat",
      messages: [
        {
          role: "System",
          text: "You are ZAMM, a chat program. Respond in first person.",
        },
        {
          role: "Human",
          // eslint-disable-next-line max-len
          text: "I'm not sure that this diesel code is doing quite what I want:\n\n```rs\nlet llm_calls_display = diesel::alias!(llm_calls as display);\n\n    let previous_call_result: LlmCallLeftJoinResult = llm_calls::table\n        .left_join(\n            llm_call_continuations::dsl::llm_call_continuations\n                .on(llm_calls::id.eq(llm_call_continuations::next_call_id)),\n        )\n        .inner_join(llm_calls_display.on(llm_call_continuations::previous_call_id.eq(llm_calls_display.field(llm_calls::id))))\n        .select((\n            llm_calls::all_columns,\n            llm_call_continuations::previous_call_id.nullable(),\n            llm_calls_display.fields(llm_calls::completion).nullable(),\n        ))\n        .filter(llm_calls::id.eq(&parsed_uuid))\n        .first::<LlmCallLeftJoinResult>(conn)?;\n```\n\nHow do I get Diesel to first do an inner join on llm_call_continuations with llm_calls_display, and then do a left join with llm_calls?",
        },
        {
          role: "AI",
          // eslint-disable-next-line max-len
          text: "With Diesel ORM in Rust, you can't change arbitrarily the order of `JOIN` operations through the query builder because it always follows a left to right order in a straight-line query. \n\nHowever, in your situation, there is a possible workaround. Although this method is a bit verbose and more complicated, Diesel allows you to nest `JOIN` statements, thus enabling more flexibility. You could first perform the inner join and then use the result of that to perform the left join.\n\nHere's an example of how you could modify your code:\n\n```rs\nlet subquery = llm_call_continuations::dsl::llm_call_continuations\n          .inner_join(llm_calls_display::table)\n          .filter(llm_call_continuations::previous_call_id.eq(llm_calls_display.field(llm_calls::id)))\n          .select((\n            llm_call_continuations::previous_call_id,\n            llm_calls_display.fields(llm_calls::completion)\n            ))\n          .into_boxed();\n\nlet previous_call_result: LlmCallLeftJoinResult = llm_calls::table\n    .left_join(\n        subquery.on(diesel::dsl::exists(\n            subquery.filter(subquery_columns::id.eq(llm_calls::id)).limit(1),\n        )),\n    )\n    .select((llm_calls::all_columns, subquery_columns::all_columns.nullable()))\n    .filter(llm_calls::id.eq(&parsed_uuid))\n    .first::<LlmCallLeftJoinResult>(conn)?;\n```\n\nNotice that the inner join between `llm_call_continuations` and `llm_calls_display` is now separated into a subquery which is then used in the left join with `llm_calls`. This would achieve the desired effect of first performing the inner join and then the left join. Please adjust this skeleton code based on your column names and requirements. \n\nAlso please remember to handle the nullable fields as necessary from the subquery. \n\nMake sure also that your actual ORM model matches the SQL query, specifically the nullability of your fields.",
        },
        {
          role: "Human",
          text: "The variable `subquery_columns` is not defined anywhere.",
        },
      ],
    },
    temperature: 1,
  },
  response: {
    completion: {
      role: "AI",
      // eslint-disable-next-line max-len
      text: "I apologize for the confusion. The `subquery_columns` used in the previous example was intended as a placeholder for the actual columns you're selecting from your subquery. Diesel doesn't have direct support for aliasing a subquery so you can't directly access those columns.\n\nInstead, you'll have to use individual .select() calls for each column in your subquery that you want to use in the outer query.\n\nHere is a corrected version of the code where `subquery_columns` is replaced by appropriate column names:\n\n```rs\nlet subquery = llm_call_continuations::dsl::llm_call_continuations\n    .inner_join(llm_calls_display::table)\n    .filter(llm_call_continuations::previous_call_id.eq(llm_calls_display.field(llm_calls::id)))\n    .select((\n        llm_call_continuations::previous_call_id,\n        llm_calls_display.fields(llm_calls::completion)\n    ))\n    .into_boxed();\n\nlet previous_call_result: LlmCallLeftJoinResult = llm_calls::table\n    .left_join(\n         diesel::dsl::exists(\n            subquery.filter(llm_call_continuations::previous_call_id.eq(llm_calls::id))\n        )\n    )\n    .select((\n        llm_calls::all_columns, \n        subquery.select(llm_call_continuations::previous_call_id).nullable(), \n        subquery.select(llm_calls_display.fields(llm_calls::completion)).nullable()\n    ))\n    .filter(llm_calls::id.eq(&parsed_uuid))\n    .first::<LlmCallLeftJoinResult>(conn)?;\n```\n\nIn this code, both `llm_call_continuations::previous_call_id` and `llm_calls_display.fields(llm_calls::completion)` are used in the .select() calls for `subquery` in the outer query. Please adjust to your actual column names and select items. \n\nYou might also need to import `IntoBoxed` trait from diesel::query_dsl::boxed::IntoBoxed to use `into_boxed()` method on your query.\n\nAgain, with Diesel ORM you have to match your ORM and query structure exactly with your SQL query structure, so you will need to ensure that this modified query still aligns with your underlying data model.",
    },
  },
  tokens: {
    prompt: 688,
    response: 462,
    total: 1150,
  },
  conversation: {
    next_calls: [
      {
        id: "a0f13902-8ae9-4fce-9317-5189636bb058",
        snippet:
          "`HasTable` trait is implemented for every struct that derives `Table`....",
      },
    ],
    previous_call: {
      id: "93c89a67-7423-42dd-9869-4fc155c2f477",
      snippet: "With Diesel ORM in Rust, you can't change arbitrarily the...",
    },
  },
};

...

export const LotsOfCode: StoryObj = Template.bind({}) as any;
LotsOfCode.args = {
  apiCall: LOTS_OF_CODE_CALL,
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
LotsOfCode.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
};
```

This new test case also shows us the need to do better word wrapping on the `pre` text. It turns out all we need to change is to add these lines to `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.svelte`:

```css
  pre {
    ...
    word-wrap: break-word;
    word-break: break-word;
    ...
  }
```

We add the new test case to `src-svelte\src\routes\storybook.test.ts`, and also set the `wide` case to require a window resize, because the new links mean that they no longer fit:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "individual"],
    variants: [
      ...,
      {
        name: "wide",
        resizeWindow: true,
      },
      ...,
      {
        name: "lots-of-code",
        resizeWindow: true,
      }
    ],
    ...
  },
  ...
];
```

This looks fine for a large enough screen, but the links get too smushed on small screens. We edit `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.svelte`, toning down the gap to conform with the 0.5rem gap within each group, but then realizing that the smaller gap works well on bigger screens too:

```css
  .conversation-links {
    ...
    gap: 0.5rem;
  }

  ...

  @media (max-width: 46rem) {
    .conversation-links {
      flex-direction: column;
    }

    .conversation.previous-links,
    .conversation.next-links {
      flex: none;
    }
  }
```

Unfortunately, we find that the screenshots taken now aren't very good. This is because the screen contents now extend past the height of the device that we've set on Storybook. Clearly we need to get rid of that viewport option. However, resizing the entire window to be extremely narrow causes its own set of complications with Storybook's UI changing to fit, and also doesn't let us easily view the Storybook story unless we resize the window ourselves. On the other hand, not resizing the entire window means that the CSS media queries for smaller screens won't be triggered. We end up resizing the entire window before even loading it, by editing `src-svelte/src/routes/storybook.test.ts` as such:

```ts
...

const NARROW_WINDOW_SIZE = 675;

...

interface VariantConfig {
  ...
  narrowWindow?: boolean;
  ...
}

const components: ComponentTestConfig[] = [
  ...
  {
    path: ["screens", "llm-call", "individual"],
    variants: [
      {
        name: "narrow",
        narrowWindow: true,
        resizeWindow: true,
      },
      ...
    ],
    screenshotEntireBody: true,
  },
  ...
];

...

  const makeWindowNarrow = async (page: Page) => {
    const currentViewport = page.viewportSize();
    if (currentViewport === null) {
      throw new Error("Viewport is null");
    }

    await page.setViewportSize({
      width: NARROW_WINDOW_SIZE,
      height: currentViewport.height,
    });
  };

  for (const config of components) {
    ...

    test(
        `${testName} should render the same`,
        async ({ expect }) => {
          const page =
            ...;
          const variantPrefix = ...;

          if (variantConfig.narrowWindow) {
            await makeWindowNarrow(page);
          }

          await page.goto(
            ...
          );

          ...
        },
        ...
    );
```

We then rename `resizeWindow` to `tallWindow` to make the naming consistent with `narrowWindow`.

Resizing the window also makes us realize that we should edit `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.svelte` to make the snippet longer when it fits completely within less than one line in the UI:

```css
  .conversation.link .snippet {
    flex: 1;
    ...
  }
```

Finally, we realize that we should get the backend to send perhaps double the amount of text it is currently sending. We refactor and edit `src-tauri/src/models/llm_calls/various.rs` to set the number of words taken for a snippet to 20:

```rs
...

const NUM_WORDS_TO_SNIPPET: usize = 20;

...

impl From<(EntityId, ChatMessage)> for LlmCallReference {
    fn from((id, message): (EntityId, ChatMessage)) -> Self {
        ...
        let truncated_text = text
            ...
            .take(NUM_WORDS_TO_SNIPPET)
            ...;
        ...
    }
}
```

and then we update the test files. Some test files still show the relevant truncation, for example `src-tauri\api\sample-calls\get_api_call-continue-conversation.yaml`:

```yaml
...
response:
  message: >
    {
      "id": "c13c1e67-2de3-48de-a34c-a32079c03316",
      ...,
      "conversation": {
        "previous_call": {
          "id": "d5ad1e49-f57f-4481-84fb-4d70ba8a7a74",
          "snippet": "Yes, it works. How can I assist you today?"
        },
        "next_calls": [
          {
            "id": "0e6bcadf-2b41-43d9-b4cf-81008d4f4771",
            "snippet": "Sure, here is a simple Python script that will print out the joke: ```python print(\"Why don't scientists trust atoms? Because..."
          },
          {
            "id": "63b5c02e-b864-4efe-a286-fbef48b152ef",
            "snippet": "Sure, here is a simple Rust program that prints out the joke: ```rust fn main() { println!(\"Why don't scientists trust..."
          }
        ]
      }
    }
...
```

We update `CONTINUE_CONVERSATION_CALL` and `LOTS_OF_CODE_CALL` in `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.stories.ts` as well to coincide with the updated gold API call and the actual outputs from the app, respectively.

We also tweak `src-svelte\src\routes\api-calls\[slug]\ApiCallDisplay.svelte` to get the table text looking better when the view is narrow:

```css
  td {
    text-align: left;
  }
```

We also update the credits page at `src-svelte\src\routes\credits\Credits.svelte`:

```svelte
              <Creditor
                name="MingCute Icon"
                logo="mingcute.svg"
                url="https://www.mingcute.com/"
              />
```

The Storybook screenshot tests work as expected once the screenshots are updated. However, the end-to-end tests are now failing because the sidebar icons are no longer showing inset shadows. Since we haven't touched the sidebar at all here, and since the Storybook sidebar screenshots see no change, we see if the e2e tests fail on main as well. Sure enough, they do despite having passed with no problems just a week earlier. We verify that the inset shadows still render correctly on our own Linux machine with the built AppImage from CI, and then update the e2e screenshots separately.

## Persisting and resuming conversations

We create a new migration to introduce the concept of *conversations* that LLM calls are related to:

```bash
$ diesel migration generate create_conversation
Creating migrations/2024-01-31-051556_create_conversation/up.sql
Creating migrations/2024-01-31-051556_create_conversation/down.sql
```

As usual, we edit

and then run

```bash
$ diesel migration run --database-url /root/.local/share/zamm/zamm.sqlite3
```
