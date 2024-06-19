# Nasalized button

## Adding disabled feature

We edit `src-svelte/src/lib/controls/Button.svelte` to add a new disabled option:

```svelte
<script lang="ts">
  ...
  export let disabled = false;
  ...
</script>

{#if unwrapped}
  <button
    class="cut-corners inner"
    ...
    class:disabled
    ...
  >
    <slot />
  </button>
{:else}
  <button
    class="cut-corners outer"
    ...
    class:disabled
    ...
  >
    <div
      class="cut-corners inner"
      ...
    >
      <slot />
    </div>
  </button>
{/if}

<style>
  ...

  .inner.disabled,
  .disabled .inner {
    filter: grayscale(1);
    color: var(--color-faded);
    pointer-events: none;
  }
  .inner.disabled:hover,
  .disabled .inner:hover {
    filter: grayscale(1);
  }
  .inner.disabled:active,
  .disabled .inner:active {
    transform: none;
  }

  ...
</style>
```

We edit `src-svelte/src/lib/controls/Button.stories.ts` to showcase this new option:

```ts
export const Disabled: StoryObj = Template.bind({}) as any;
Disabled.args = {
  text: "Simulate",
  disabled: true,
};
```

We find that we have to edit `src-svelte/src/lib/controls/ButtonView.svelte` as well to pass the new argument through to the underlying component:

```svelte
<Button {...$$restProps}>...</Button>
```

We now make use of this in `src-svelte/src/routes/api-calls/new/ApiCallEditor.svelte`:

```svelte
<script lang="ts">
  ...

  export let expectingResponse = false;
  ...
</script>

<InfoBox title="New API Call">
  ...
  <div class="action">
    <Button disabled={expectingResponse} ...>Submit</Button
    >
  </div>
</InfoBox>
```

We add a story to `src-svelte/src/routes/api-calls/new/ApiCallEditor.stories.ts` as well. The parameters are copied verbatim from the `EditContinuedConversation` story:

```ts
export const Busy: StoryObj = Template.bind({}) as any;
Busy.args = {
  expectingResponse: true,
};
Busy.parameters = {
  stores: {
    ...
  },
};
```

and add the new stories to `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["reusable", "button"],
    variants: [..., "disabled"],
  },
  ...,
  {
    path: ["screens", "llm-call", "new"],
    variants: [..., "busy"],
    screenshotEntireBody: true,
  },
  ...
];
```

Finally, we edit `src-svelte/src/routes/chat/Form.svelte` as well:

```svelte
<form
  ...
>
  ...
  <Button ariaLabel="Send" disabled={chatBusy} ...
    >...</Button
  >
</form>
```
