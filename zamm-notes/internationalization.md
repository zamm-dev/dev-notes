# Internationalization

## Khmer fonts

We browse through [this list](https://fonts.google.com/?subset=khmer&noto.script=Khmr) of Khmer fonts on Google fonts, and decide to add Nokora:

```bash
$ yarn add @fontsource/nokora
```

We edit `src-svelte\src\routes\styles.css`:

```css
...
@import "@fontsource/nokora";
...

:root {
  --font-body: "Good Timing", "Nokora", sans-serif;
  ...
}
```

and make sure to try it out in `src-svelte\src\routes\chat\Message.stories.ts`:

```ts
export const MixedKhmerAndEnglish: StoryObj = Template.bind({}) as any;
MixedKhmerAndEnglish.args = {
  message: {
    role: "Human",
    text: "Hello, សួស្ដី, what languages do you speak? ចេះខ្មែរអត់?",
  },
};
MixedKhmerAndEnglish.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

```

We register this in `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "chat", "message"],
    variants: [
      ...,
      "mixed-khmer-and-english",
    ],
  },
  ...
];
```

Upon committing, we get a lot of formatting errors due to `prettier` no longer putting trailing commas. This shouldn't be because all lints are passing in CI, so we do

```bash
$ yarn install --frozen-lockfile
```

again and find that the problem is now fixed.

Next, we try to apply this for fixed-width text in the API call screen too. The Nokora font doesn't fit as well as Kdam Thmor Pro, which has a blockier look that better matches the look of the existing fixed-width font. So, we add that

```bash
$ yarn add @fontsource/kdam-thmor-pro
```

and then edit `src-svelte\src\routes\styles.css`:

```css
...
@import "@fontsource/kdam-thmor-pro";
...

:root {
  ...
  --font-mono: "Jetbrains Mono", "Kdam Thmor Pro", monospace;
  ...
}
```

We create a sample call file at `src-tauri/api/sample-calls/get_api_call-khmer.yaml` -- we don't need the database side effects here because this won't be tested on the backend:

```yaml
request:
  - get_api_call
  - >
    {
      "id": "92665f19-be8c-48f2-b483-07f1d9b97370"
    }
response:
  message: >
    {
      "id": "92665f19-be8c-48f2-b483-07f1d9b97370",
      "timestamp": "2024-04-10T07:22:12.752276900",
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
              "text": "Hello, សួស្ដី, what languages do you speak? ចេះខ្មែរអត់?"
            }
          ]
        },
        "temperature": 1.0
      },
      "response": {
        "completion": {
          "role": "AI",
          "text": "Hello! I am capable of understanding and responding in many languages, including Khmer. So, yes, ខ្មែរបាទ/ចាស, I understand Khmer. How can I assist you today?"
        }
      },
      "tokens": {
        "prompt": 68,
        "response": 52,
        "total": 120
      }
    }

```

We add a story in `src-svelte\src\routes\api-calls\[slug]\ApiCall.stories.ts`:

```ts
export const Khmer: StoryObj = Template.bind({}) as any;
Khmer.args = {
  id: "92665f19-be8c-48f2-b483-07f1d9b97370",
  dateTimeLocale: "en-GB",
  timeZone: "Asia/Phnom_Penh",
};
Khmer.parameters = {
  viewport: {
    defaultViewport: "smallTablet",
  },
  sampleCallFiles: [
    "/api/sample-calls/get_api_call-khmer.yaml",
  ],
};
```

And register this in `src-svelte\src\routes\storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "llm-call", "individual"],
    variants: [
      ...,
      "khmer",
    ],
  },
  ...
];
```

### Further customizing the look of Khmer text

We edit `src-svelte\src\routes\chat\MessageUI.svelte` to make Khmer text larger while keeping English text the same size:

```svelte
<script lang="ts" context="module">
  export function styleKhmer(text: string) {
    var newText = '';
    var isKhmer = false;
    for (let i = 0; i < text.length; i++) {
        var ch = text.charAt(i);
        if (ch >= '\u1780' && ch <= '\u17FF') {
            if (!isKhmer) {
                newText += '<span class="khmer">';
            }
            isKhmer = true;
        } else {
            if (isKhmer) {
                newText += '</span>';
            }
            isKhmer = false;
        }
        newText += ch;
    }
    if (isKhmer) {
        newText += '</span>';
    }
    return newText;
  }
</script>

<script lang="ts">
  ...

  onMount(() => {
    for (const element of textElement?.querySelectorAll("p") ?? []) {
      element.innerHTML = styleKhmer(element.innerHTML);
    }

    ...
  });
</script>

...

<style>
  ...

  .message .text-container :global(p span.khmer) {
    font-size: 1.2rem;
  }

  ...
</style>
```

This looks great in Storybook. The main body of Khmer text now lines up in height with the tallest English alphabet elements, such as `?`.

To be sure that the span logic is working properly, we test it in `src-svelte\src\routes\chat\MessageUI.test.ts`:

```ts
import { styleKhmer } from './MessageUI.svelte';

describe('Khmer styling', () => {
  it("should leave text without Khmer unchanged", () => {
    expect(styleKhmer("Hello, how are you?")).toEqual("Hello, how are you?");
  });

  it("should select the entire text if it is all Khmer", () => {
    expect(styleKhmer("ខ្ញុំសុខសប្បាយ")).toEqual('<span class="khmer">ខ្ញុំសុខសប្បាយ</span>');
  });

  it("should pick out Khmer text from within other text", () => {
    expect(styleKhmer("Hello, សួស្ដី, what languages do you speak? ចេះខ្មែរអត់?")).toEqual('Hello, <span class="khmer">សួស្ដី</span>, what languages do you speak? <span class="khmer">ចេះខ្មែរអត់</span>?');
  });
});

```

Because the text size effect is sublte, we shouldn't rely solely on the screenshot tests. We create `src-svelte\src\routes\chat\Message.test.ts` to test the rendered HTML. At first, we get

```
 FAIL  src/routes/chat/Message.test.ts > Text messages > should render without Khmer spans when no Khmer is present
Error: Invalid Chai property: toHaveTextContent
 ❯ src/routes/chat/Message.test.ts:12:40
     10|       },
     11|     });
     12|     expect(screen.getByRole("listitem")).toHaveTextContent("Hello, how are you?");
       |                                        ^
     13|   });
     14| 
```

Even importing

```ts
import { expect, vi } from "vitest";
```

does not work. It turns out this is due to the need to import

```ts
import "@testing-library/jest-dom";
```

The final file contents look like:

```ts
import Message from "./Message.svelte";
import "@testing-library/jest-dom";
import { render, screen } from "@testing-library/svelte";
import { expect } from "vitest";

describe('Text messages', () => {
  it("should render without Khmer spans when no Khmer is present", () => {
    render(Message, {
      message: {
        role: "Human",
        text: "Hello, how are you?",
      },
    });
    expect(screen.getByRole("listitem")).toHaveTextContent("Hello, how are you?");
  });

  it("should pick out Khmer text embedded in other text", () => {
    const { container } = render(Message, {
      message: {
        role: "Human",
        text: "Hello, សួស្ដី, what languages do you speak? ចេះខ្មែរអត់?",
      },
    });
    expect(screen.getByRole("listitem")).toHaveTextContent("Hello, សួស្ដី, what languages do you speak? ចេះខ្មែរអត់?");

    const khmerSpans = container.getElementsByClassName("khmer");
    const khmerSpanText = Array.from(khmerSpans).map((span) => span.textContent);
    expect(khmerSpanText).toEqual(["សួស្ដី", "ចេះខ្មែរអត់"]);
  });
});

```

Doing the same to `src-svelte/src/routes/api-calls/[slug]/ApiCall.svelte`:

```svelte
<script lang="ts>
  ...

  let apiCallPromise = getApiCall(id)
    .then((apiCall) => {
      ...

      setTimeout(() => {
        for (const preElement of document.querySelectorAll("pre")) {
          preElement.innerHTML = styleKhmer(preElement.innerHTML);
        }
      }, 0);

      return apiCall;
    })
    ...;
</script>

...

<style>
  ...

  pre :global(span.khmer) {
    font-weight: 100;
  }
</style>
```

doesn't really work because the [Kdam Thmor Pro font](https://fonts.google.com/specimen/Kdam+Thmor+Pro) turns out to only have a single style available.
