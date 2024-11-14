# Upgrading to Svelte 5

From [the migration guide](https://svelte.dev/docs/svelte/v5-migration-guide#Migration-script), we see that there is a migration script available. As such, we run the upgrade script for Svelte 5:

```bash
$ npx sv migrate svelte-5
Need to install the following packages:
  sv@0.6.1
Ok to proceed? (y) y
This migration is experimental — please report any bugs to https://github.com/sveltejs/svelte/issues

added 83 packages, and audited 84 packages in 32s

15 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

This will update files in the current directory
If you're inside a monorepo, don't run this in the root directory, rather run it in all projects independently.

✔ Continue? … yes
✔ Which folders should be migrated? › build, screenshots, src, static
Updated svelte to ^5.0.0
Updated svelte-check to ^4.0.0
Updated svelte-preprocess to ^6.0.0
Updated @sveltejs/kit to ^2.5.27
Updated @sveltejs/vite-plugin-svelte to ^4.0.0
Updated prettier-plugin-svelte to ^3.2.6
Updated eslint-plugin-svelte to ^2.45.1
Updated svelte-eslint-parser to ^0.42.0
Updated vite to ^5.4.4
One or more `@migration-task` comments were added to `src/routes/BackgroundUI.svelte`, please check them and complete the migration manually.
Error while migrating Svelte code [InternalCompileError: `<tr>` is invalid inside `<table>`] {
  code: 'node_invalid_placement',
  filename: 'src/routes/components/Metadata.svelte',
  position: [ 796, 850 ],
  start: { line: 32, column: 8, character: 796 },
  end: { line: 34, column: 13, character: 850 },
  frame: '30:     {:then systemInfo}\n' +
    '31:       <table>\n' +
    '32:         <tr>\n' +
    '                ^\n' +
    '33:           <th colspan="2">ZAMM</th>\n' +
    '34:         </tr>'
}
One or more `@migration-task` comments were added to `src/routes/components/Metadata.svelte`, please check them and complete the migration manually.
Error while migrating Svelte code [InternalCompileError: `<tr>` is invalid inside `<table>`] {
  code: 'node_invalid_placement',
  filename: 'src/routes/database/api-calls/[slug]/ApiCallDisplay.svelte',
  position: [ 2170, 2250 ],
  start: { line: 78, column: 6, character: 2170 },
  end: { line: 81, column: 11, character: 2250 },
  frame: '76:   {#if apiCall}\n' +
    '77:     <table class="composite-reveal">\n' +
    '78:       <tr>\n' +
    '              ^\n' +
    '79:         <td>ID</td>\n' +
    '80:         <td>{apiCall?.id ?? "Unknown"}</td>'
}
One or more `@migration-task` comments were added to `src/routes/database/api-calls/[slug]/ApiCallDisplay.svelte`, please check them and complete the migration manually.
✔ Your project has been migrated

Recommended next steps:

  1: install the updated dependencies ('npm i' / 'pnpm i' / etc) (note that there may be peer dependency issues when not all your libraries officially support Svelte 5 yet. In this case try installing with the --force option)
  2: git commit -m "migration to Svelte 5"
  3: Review the breaking changes at https://svelte-5-preview.vercel.app/docs/breaking-changes

Run git diff to review changes.
```

Just to be safe, we fix the error by editing `src-svelte/src/routes/database/api-calls/[slug]/ApiCallDisplay.svelte` accordingly to fit all the table rows inside of a `<tbody>`. We then run the command again, and find that we should edit `src-svelte/src/routes/components/Metadata.svelte` as well in the same manner. It finally succeeds:

```
$ npx sv migrate svelte-5
...
Updated vite to ^5.4.4
One or more `@migration-task` comments were added to `src/routes/BackgroundUI.svelte`, please check them and complete the migration manually.
✔ Your project has been migrated

Recommended next steps:

  1: install the updated dependencies ('npm i' / 'pnpm i' / etc) (note that there may be peer dependency issues when not all your libraries officially support Svelte 5 yet. In this case try installing with the --force option)
  2: git commit -m "migration to Svelte 5"
  3: Review the breaking changes at https://svelte-5-preview.vercel.app/docs/breaking-changes

Run git diff to review changes.
```

We do

```bash
$ yarn install
```

inside of `src-svelte/`.

When we try to commit now, we find that there are 82 pre-commit errors. We look at one of them:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/+layout.svelte
   5:9   error  'data' is assigned a value but never used. Allowed unused vars must match /^_/u  @typescript-eslint/no-unused-vars
  11:24  error  'currentRoute' is defined but never used. Allowed unused args must match /^_/u   @typescript-eslint/no-unused-vars
```

The corresponding file at `src-svelte/src/routes/+layout.svelte` now looks like this:

```svelte
<script lang="ts">
  import AppLayout from "./AppLayout.svelte";
  import "./styles.css";

  let { data, children } = $props();

  const children_render = $derived(children);
</script>

<AppLayout>
  {#snippet children({ currentRoute })}
    {@render children_render?.()}
  {/snippet}
</AppLayout>

```

We see that [the children prop](https://svelte.dev/docs/svelte/v5-migration-guide#Snippets-instead-of-slots-Default-content) is now used instead of a slot. We check our old code to see what it used to do:

```svelte
<script>
  import AppLayout from "./AppLayout.svelte";
  import "./styles.css";

  export let data;
</script>

<AppLayout currentRoute={data.url}>
  <slot />
</AppLayout>
```

We try to simplify the new code a bit:

```svelte
<script lang="ts">
  ...

  let { data, children } = $props();
</script>

<AppLayout currentRoute={data.url}>
  {@render children?.()}
</AppLayout>
```

That error appears to go away for now.

Next, we see that there is an eslint error:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/BackgroundUI.svelte
  1:1  error  This line has a length of 173. Maximum allowed is 88  max-len
```

We check `src-svelte/src/routes/BackgroundUI.svelte` and find this comment at the top:

```svelte
<!-- @migration-task Error while migrating Svelte code: can't migrate `$: animateIntervalMs = $standardDuration / 2;` to `$derived` because there's a variable named derived.
     Rename the variable and try again or migrate by hand. -->
```

We remove that comment, and try to instead migrate the file manually, because it appears the script is scared of the import of `derived` from Svelte store.

```ts
  interface Props {
    animated?: boolean;
  }

  let { animated = false }: Props = $props();
  const rng = ...;
  const animateIntervalMs = derived(standardDuration, ($sd) => $sd / 2);
  $effect(() => updateAnimationSpeed(animated, $animateIntervalMs));

  ...

  function updateAnimationSpeed(_animated: boolean, _newSpeed: number) {
    stopAnimating();
    startAnimating();
  }
```

All we want is for `updateAnimationSpeed` to be called when animation enablement or speed changes, but since the underlying functions don't take in those variables, it seems we must manually specify them. Previously, Svelte did not typecheck the "effect" function calls, but now it does.

VS Code still shows us

```
'props' is not defined  svelte(missing-declaration)
```

We'll tackle something else first. (In the end, it turns out we just need to update the Svelte extension for VS Code.) We try building with our `make` command, and run into:

```
error during build:
[vite-plugin-svelte] [plugin vite-plugin-svelte] src/routes/PageTransition.svelte (10:9): src/routes/PageTransition.svelte:10:9 TypeScript language features like enums are not natively supported, and their use is generally discouraged. Outside of `<script>` tags, these features are not supported. For use within `<script>` tags, you will need to use a preprocessor to convert it to JavaScript before it gets passed to the Svelte compiler. If you are using `vitePreprocess`, make sure to specifically enable preprocessing script tags (`vitePreprocess({ script: true })`)
file: /Users/amos/Documents/zamm/src-svelte/src/routes/PageTransition.svelte:10:9

  8 |    export const pageTransition = writable<PageTransitionContext | null>(null);
  9 |
 10 |    export enum TransitionType {
          ^
 11 |      // user is going leftward -- meaning both incoming and outgoing are moving right
 12 |      Left,
```

As such, we edit `src-svelte/svelte.config.js`:

```js
const config = {
  ...
  preprocess: vitePreprocess({ script: true }),

  ...
};
```

Everything now builds. We continue to try to fix the remaining 80 pre-commit errors.

```

/Users/amos/Documents/zamm/src-svelte/src/routes/settings/SettingsSwitch.test.ts:14:34
Error: No overload matches this call.
  Overload 1 of 2, '(component: Constructor<SvelteComponent<Record<string, any>, any, any>>, componentOptions?: SvelteComponentOptions<SvelteComponent<Record<string, any>, any, any>> | undefined, renderOptions?: Omit<...> | undefined): RenderResult<...>', gave the following error.
    Argument of type 'Component<Props, {}, "toggledOn">' is not assignable to parameter of type 'Constructor<SvelteComponent<Record<string, any>, any, any>>'.
      Type 'Component<Props, {}, "toggledOn">' provides no match for the signature 'new (...args: any[]): SvelteComponent<Record<string, any>, any, any>'.
  Overload 2 of 2, '(component: Constructor<SvelteComponent<Record<string, any>, any, any>>, componentOptions?: SvelteComponentOptions<SvelteComponent<Record<string, any>, any, any>> | undefined, renderOptions?: RenderOptions<...> | undefined): RenderResult<...>', gave the following error.
    Argument of type 'Component<Props, {}, "toggledOn">' is not assignable to parameter of type 'Constructor<SvelteComponent<Record<string, any>, any, any>>'.
  test("can be toggled on from clicking the container", async () => {
    const { container } = render(SettingsSwitch, { label: "Test" });
```

We see that we can import `mount` from Svelte as documented [here](https://svelte.dev/docs/svelte/testing#Unit-and-integration-testing-using-Vitest-Component-testing). However, it doesn't seem right to just use Svelte functions without involving testing-library at all. (Also, it appears that we may eventually want to make other updates mentioned in that file for testing.) We try copying [this comment](https://github.com/testing-library/svelte-testing-library/issues/284#issuecomment-2457034427):

```ts
const { container } = render(SettingsSwitch, { props: { label: "Test" }});
```

but it still complains. We see that there is a [newer version](https://www.npmjs.com/package/@testing-library/svelte) of the library for Svelte available, so we try updating `src-svelte/package.json`:

```json
{
  ...,
  "devDependencies": {
    ...,
    "@testing-library/svelte": "^5.2.4",
    ...
  }
}
```

and run `yarn install` again to update the lockfile.

It still fails. We see from the [`testing-library` docs](https://testing-library.com/docs/svelte-testing-library/setup) that there are now more steps to set up Svelte for testing. We run through them again:

```bash
$ yarn add --dev @testing-library/svelte @testing-library/jest-dom @sveltejs/vite-plugin-svelte vitest jsdom @vitest/ui
```

We create `src-svelte/vitest-setup.js` as recommended:

```js
import '@testing-library/jest-dom/vitest'
```

We try editing `src-svelte/vitest.config.ts` as recommended:

```ts
...
import {sveltekit} from '@sveltejs/kit/vite'
import {svelteTesting} from '@testing-library/svelte/vite';
...

export default defineConfig({
  plugins: [
    sveltekit(),
    svelteTesting(),
    ...
  ],
  ...
});
```

In VS Code, we get

```
Cannot find module '@testing-library/svelte/vite' or its corresponding type declarations. ts(2307)
```

We try to at least get things compiling first.

```

yarn run v1.22.22
$ /Users/amos/Documents/zamm/node_modules/.bin/eslint --fix --max-warnings=0 src-svelte/src/routes/database/api-calls/ApiCallsTable.svelte src-svelte/src/lib/__mocks__/MockPageTransitions.svelte src-svelte/src/routes/components/api-keys/Service.svelte src-svelte/src/routes/settings/SettingsSlider.svelte src-svelte/src/routes/database/terminal-sessions/TerminalSessionBlurb.svelte src-svelte/src/lib/Scrollable.svelte src-svelte/src/routes/BackgroundUI.svelte src-svelte/src/routes/chat/Message.svelte src-svelte/src/routes/components/Metadata.svelte src-svelte/src/routes/database/terminal-sessions/TerminalSession.svelte

/Users/amos/Documents/zamm/src-svelte/src/routes/BackgroundUI.svelte
    1:19  error  `context="module"` is deprecated, use the `module` attribute instead(script_context_deprecated)  svelte/valid-compile
  141:12  error  'updateAnimationState' is defined but never used. Allowed unused vars must match /^_/u           @typescript-eslint/no-unused-vars
```

Once again, we edit `src-svelte/src/routes/BackgroundUI.svelte`:

```svelte
<script module lang="ts">
  ...
</script>

<script lang="ts>
  ...
  $effect(() => updateAnimationState(animated));
  $effect(() => updateAnimationSpeed($animateIntervalMs));
  ...

  function updateAnimationState(nowAnimating: boolean) {
    ...
  }

  function updateAnimationSpeed(_newSpeed: number) {
    ...
  }
</script>
```

It turns out we had erroneously mistook both effects to use `updateAnimationSpeed` at first, when that's actually not the case. We restore the original logic.

Next, we get

```

[error] src-svelte/src/lib/controls/SendInputForm.svelte: Error: unknown node type: Script
[error]     at Object.print (/Users/amos/Documents/zamm/node_modules/prettier-plugin-svelte/plugin.js:1474:11)
[error]     at callPluginPrintFunction (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8601:26)
[error]     at mainPrintInternal (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8550:22)
[error]     at mainPrint (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8537:18)
[error]     at AstPath.call (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8359:24)
[error]     at printTopLevelParts (/Users/amos/Documents/zamm/node_modules/prettier-plugin-svelte/plugin.js:1515:33)
[error]     at Object.print (/Users/amos/Documents/zamm/node_modules/prettier-plugin-svelte/plugin.js:929:16)
[error]     at callPluginPrintFunction (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8601:26)
[error]     at mainPrintInternal (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8550:22)
[error]     at mainPrint (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8537:18)
src-svelte/src/routes/database/api-calls/[slug]/Prompt.svelte{
    "type": "Script",
    "start": 0,
    "end": 1647,
    "context": "default",
    "content": {
        "type": "Program",
        "start": 1636,
        "end": 1638,
        "loc": {
            "start": {
                "line": 1,
                "column": 0
            },
            "end": {
                "line": 1,
                "column": 1638
            }
        },
        "body": [
            {
                "type": "BlockStatement",
                "start": 1636,
                "end": 1638,
                "loc": {
                    "start": {
                        "line": 1,
                        "column": 1636
                    },
                    "end": {
                        "line": 1,
                        "column": 1638
                    }
                },
                "body": []
            }
        ],
        "sourceType": "module"
    }
}

[error] src-svelte/src/routes/database/api-calls/[slug]/Prompt.svelte: Error: unknown node type: Script
[error]     at Object.print (/Users/amos/Documents/zamm/node_modules/prettier-plugin-svelte/plugin.js:1474:11)
[error]     at callPluginPrintFunction (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8601:26)
[error]     at mainPrintInternal (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8550:22)
[error]     at mainPrint (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8537:18)
[error]     at AstPath.call (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8359:24)
[error]     at printTopLevelParts (/Users/amos/Documents/zamm/node_modules/prettier-plugin-svelte/plugin.js:1515:33)
[error]     at Object.print (/Users/amos/Documents/zamm/node_modules/prettier-plugin-svelte/plugin.js:929:16)
[error]     at callPluginPrintFunction (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8601:26)
[error]     at mainPrintInternal (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8550:22)
[error]     at mainPrint (/Users/amos/Documents/zamm/node_modules/prettier/index.js:8537:18)
src-svelte/vitest.config.ts 87ms
error Command failed with exit code 2.
```

We try

```bash
$ yarn add --dev prettier-plugin-svelte
```

and that error goes away.

We now have 31 errors remaining:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/settings/Settings.stories.ts:39:3
Error: Type '(story: StoryFn) => { Component: Component<Props, {}, "">; slot: AnnotatedStoryFn<SvelteRenderer<SvelteComponentTyped<Record<string, any>, any, any>>, Args>; }' is not assignable to type 'DecoratorFunction<SvelteRenderer<SvelteComponent<Record<string, any>, any, any>>, { [x: string]: any; }>'.
  Call signature return types '{ Component: Component<Props, {}, "">; slot: AnnotatedStoryFn<SvelteRenderer<SvelteComponentTyped<Record<string, any>, any, any>>, Args>; }' and 'SvelteStoryResult<any, any>' are incompatible.
    The types of 'Component' are incompatible between these types.
      Type 'Component<Props, {}, "">' is not assignable to type 'ComponentType<any, any>'.
FullPage.decorators = [
  (story: StoryFn) => {
    return {
```

We try

```bash
$ yarn add --dev @storybook/svelte
```

Now we have 24 errors:

```

/Users/amos/Documents/zamm/src-svelte/src/routes/settings/SettingsSwitch.svelte:17:7
Error: Type '({ $on?(type: string, callback: (e: any) => void): () => void; $set?(props: Partial<Props>): void; } & { toggle: () => void; }) | undefined' is not assignable to type '{ $on?(type: string, callback: (e: any) => void): () => void; $set?(props: Partial<Props>): void; } & { toggle: () => void; }'.
  Type 'undefined' is not assignable to type '{ $on?(type: string, callback: (e: any) => void): () => void; $set?(props: Partial<Props>): void; } & { toggle: () => void; }'.
    Type 'undefined' is not assignable to type '{ $on?(type: string, callback: (e: any) => void): () => void; $set?(props: Partial<Props>): void; }'. (ts)
  }: Props = $props();
  let switchChild: Switch = $state();
</script>
```

We change the offending line to simply

```
let switchChild: Switch;
```

because we don't actually want anything to be triggered once `switchChild` is bound. We get this eslint error instead:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/settings/SettingsSwitch.svelte
  17:7  error  `switchChild` is updated, but is not declared with `$state(...)`. Changing its value will not correctly trigger updates(non_reactive_update)  svelte/valid-compile
```

It appears from [this comment](https://github.com/sveltejs/svelte/issues/11818#issuecomment-2150891877) that direct usage of the variable in the template will trigger this warning. We fix this, along with the use of the deprecate `preventDefault`:

```svelte
<script lang="ts>
  ...

  let switchChild: Switch | undefined;

  function toggleSwitchChild(e: Event) {
    e.preventDefault();
    switchChild?.toggle();
  }
</script>

<div
  ...
  onclick={toggleSwitchChild}
  ...
>
  ...
</div>
```

Now with 23 errors remaining:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/database/api-calls/[slug]/ApiCall.svelte:16:29
Error: Cannot use 'bind:' with this property. It is declared as non-bindable inside the component.
To mark a property as bindable: 'let { apiCall = $bindable() } = $props()' (ts)
<div class="container">
  <ApiCallDisplay {...rest} bind:apiCall />
  {#if showActions}
```

For this one, it appears that it shouldn't even be bindable. This is the sort of judgment call that might be hard to make for an LLM.

First, through a global search, we find and edit `src-svelte/src/routes/database/api-calls/[slug]/+page.svelte` to import the file as `UnloadedApiCall` instead of `ApiCall`, for clarity:

```svelte
<script>
  ...
  import UnloadedApiCall from "./UnloadedApiCall.svelte";
</script>

<UnloadedApiCall ... />

```

Now there is only one call to `ApiCall`, and that is in `src-svelte/src/routes/database/api-calls/[slug]/UnloadedApiCall.svelte`. We head on to `src-svelte/src/routes/database/api-calls/[slug]/ApiCall.svelte`:

```svelte
<script lang="ts">
  ...

  let { apiCall, showActions = true, ...rest }: Props = $props();
</script>

<div class="container">
  <ApiCallDisplay {apiCall} {...rest} />
  ...
</div>
```

We should refactor `src-svelte/src/routes/database/api-calls/[slug]/ApiCallDisplay.svelte` later to assume that `ApiCall` always exists, but for now we'll just get things compiling first.

```
/Users/amos/Documents/zamm/src-svelte/src/routes/database/terminal-sessions/TerminalSessionsTable.svelte:22:3
Error: Type 'Component<Props, {}, "">' is not assignable to type 'ConstructorOfATypedSvelteComponent'.
  Type 'Component<Props, {}, "">' provides no match for the signature 'new (args: { target: any; props?: any; }): ATypedSvelteComponent'. (ts)
  itemUrl={terminalSessionUrl}
  renderItem={TerminalSessionBlurb}
  {...rest}
```

After reading [this comment](https://www.reddit.com/r/sveltejs/comments/1fjwdp4/comment/lnr9wxf/), the [migration documentation](https://svelte.dev/docs/svelte/v5-migration-guide#Components-are-no-longer-classes-Component-typing-changes), the [@const documentation](https://svelte.dev/docs/svelte/@const), we understand that we should change `src-svelte/src/lib/Table.svelte` like so:

```svelte
<script lang="ts" generics="Item extends { ... }">
  ...
  import { ..., type Component } from "svelte";

  interface Props {
    ...
    renderItem: Component<{ item: Item}>;
    ...
  }
</script>

...

        {#each items as item (...)}
          {@const Blurb = renderItem}
          <a href={itemUrl(item)}>
            ...
                  <Blurb {item} />
            ...
          </a>
        {/each}
```

where `ConstructorOfATypedSvelteComponent` is no longer used.

We're now down to 20 errors:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/SidebarUI.svelte:58:7
Error: Type 'number | undefined' is not assignable to type 'number'.
  Type 'undefined' is not assignable to type 'number'. (ts)
  let { currentRoute = $bindable(), dummyLinks = false }: Props = $props();
  let indicatorPosition: number = $state();
  let transitionDuration = $state(REGULAR_TRANSITION_DURATION);
```

We make it `let indicatorPosition: number = $state(0);` for a default uninitialized value. We could log the actual initial value and set it, but `getIndicatorPosition` sets it to 0 anyway by default. We'll have to refactor this file to do a proper reset back to a default value.

19 errors remaining:

```
/Users/amos/Documents/zamm/src-svelte/src/lib/Switch.test.ts:1:33
Error: Module '"vitest"' has no exported member 'SpyInstance'.
import { expect, test, vi, type SpyInstance, type Mock } from "vitest";
import "@testing-library/jest-dom";
```

We see from the [vitest migration guide](https://vitest.dev/guide/migration#deprecated-options-removed) that "`SpyInstance` is removed in favor of `MockInstance`". We change the import and `recordSoundDelaySpy` to match.

```

/Users/amos/Documents/zamm/src-svelte/src/lib/Switch.svelte:46:7
Error: Type 'HTMLElement | undefined' is not assignable to type 'HTMLElement'.
  Type 'undefined' is not assignable to type 'HTMLElement'. (ts)
  }: Props = $props();
  let toggleBound: HTMLElement = $state();
  let left = $state(0);
```

We edit `src-svelte/src/lib/Switch.svelte` to:

```ts
let toggleBound: HTMLElement | undefined = $state(undefined);
```

but now the bound for `() => toggleBound` is no longer correct as a possible member of `DragBounds`. So, we do:

```ts
  function getBounds() {
    if (!toggleBound) {
      throw new Error("Toggle bounds not set");
    }
    return toggleBound;
  }

  let toggleDragOptions: DragOptions = $state({
    ...
    bounds: getBounds,
    ...
  });
```

We do the same changes in `src-svelte/src/lib/Slider.svelte`:

```ts
  ...
  let track: HTMLDivElement | null = $state(null);
  let toggleBound: HTMLDivElement | null = $state(null);
  let toggleLabel: HTMLDivElement | null = $state(null);
  ...
```

With 14 errors remaining, we see

```
/Users/amos/Documents/zamm/src-svelte/src/routes/Background.svelte:13:15
Error: Cannot use 'bind:' with this property. It is declared as non-bindable inside the component.
To mark a property as bindable: 'let { animated = $bindable() } = $props()' (ts)

<BackgroundUI bind:animated />
```

We edit `src-svelte/src/routes/Background.svelte` to remove another `bind` that does not appear to matter:

```svelte
<script lang="ts">
  ...

  let animated: boolean = $state(false);
  $effect(() => {
    animated = $animationsOn && $backgroundAnimation;
  });
</script>

<BackgroundUI {animated} />

```

Next, we encounter a similar error. We edit `src-svelte/src/lib/controls/SendInputForm.svelte` to initialize `textareaInput` properly, and to also guard for `textareaInput` being null when it shouldn't be:

```ts
  let textareaInput: HTMLTextAreaElement | null = $state(null);

  onMount(() => {
    if (!textareaInput) {
      throw new Error("Textarea input not found");
    }

    autosize(textareaInput);
    textareaInput.addEventListener(...);

    return () => {
      if (!textareaInput) {
        throw new Error("Textarea input not found");
      }

      autosize.destroy(textareaInput);
    };
  });

  ...

  function submitInput() {
    ...
    if (message && !isBusy) {
      ...
      requestAnimationFrame(() => {
        if (!textareaInput) {
          throw new Error("Textarea input not found");
        }
        autosize.update(textareaInput);
      });
    }
  }
```

With 10 errors remaining:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/chat/Message.svelte:22:56
Error: Type 'Component<Props, {}, "">' is not assignable to type 'InstantiableSvelteComponentTyped<Partial<Omit<Tokens.Code, "type">>, any, any>'.
  Type 'Component<Props, {}, "">' provides no match for the signature 'new (...args: any[]): SvelteComponentTyped<Partial<Omit<Tokens.Code, "type">>, any, any>'. (ts)
  <div class="markdown">
    <SvelteMarkdown source={message.text} renderers={{ code: CodeRender }} />
  </div>
```

We have code like this in that file:

```svelte
<script lang="ts">
  ...
  let { message, ...rest }: Props = $props();
  let resizeBubbleBound: (chatWidthPx: number) => Promise<void> = $state();

  export function resizeBubble(chatWidthPx: number) {
    return resizeBubbleBound(chatWidthPx);
  }
</script>

<MessageUI role={message.role} bind:resizeBubble={resizeBubbleBound} {...rest}>
  ...
</MessageUI>
```

What is even happening here? It appears that `src-svelte/src/routes/chat/Chat.svelte` is not even using the `resizeBubble` variable here.

Looking at our notes from before, we see that the Chat component does call resize on each of the child messages.

```ts
  async function onScrollableResized(e: ResizedEvent) {
    conversationWidthPx = e.detail.width;
    const resizePromises = messageComponents.map((message) =>
      message.resizeBubble(e.detail.width),
    );
    ...
  }
```

The thing is that `src-svelte/src/routes/chat/MessageUI.svelte` doesn't even export that as a prop, but instead as a functin. We edit `src-svelte/src/routes/chat/Message.svelte` as such:

```svelte
<script lang="ts">
  ...

  let { message, ...rest }: Props = $props();
  let messageUI: MessageUI;

  export function resizeBubble(chatWidthPx: number) {
    return messageUI.resizeBubble(chatWidthPx);
  }
</script>

<MessageUI ... bind:this={messageUI} ...>
  ...
</MessageUI>
```

We see that there is yet another error in this file:

```
/Users/amos/Documents/zamm/src-svelte/src/routes/chat/Message.svelte:22:56
Error: Type 'Component<Props, {}, "">' is not assignable to type 'InstantiableSvelteComponentTyped<Partial<Omit<Tokens.Code, "type">>, any, any>'.
  Type 'Component<Props, {}, "">' provides no match for the signature 'new (...args: any[]): SvelteComponentTyped<Partial<Omit<Tokens.Code, "type">>, any, any>'. (ts)
  <div class="markdown">
    <SvelteMarkdown source={message.text} renderers={{ code: CodeRender }} />
  </div>
```

It seems Svelte markdown hasn't been updated to use the new Svelte 5 types, but it should still work according to [this issue](https://github.com/pablo-abc/svelte-markdown/pull/93). We update the issue with our typescript findings, and we work around it with

```ts
<script lang="ts">
  ...
  import ..., { type Renderers } from "svelte-markdown";

  ...
  const renderers: Partial<Renderers> = {
    code: CodeRender,
  } as unknown as Partial<Renderers>;

  ...
</script>

<MessageUI ...>
  ...
    <SvelteMarkdown ... {renderers} />
  ...
</MessageUI>

```

We go on to the next error, where we edit `src-svelte/src/routes/chat/MessageUI.svelte`:

```ts
  let textElement: HTMLDivElement | null = $state(null);
```

Then we have

```
/Users/amos/Documents/zamm/src-svelte/src/lib/InfoBox.svelte:389:11
Error: Type 'number' is not assignable to type 'string'. (ts)
          const opacity = easingFunction(tLocalFraction);
          anim.node.style.opacity = opacity;
```

It appears we have to do this instead, as confirmed by [this documentation](https://www.w3schools.com/cssref/css3_pr_opacity.php):

```ts
  anim.node.style.opacity = opacity.toString();
```

Next, we edit `src-svelte/src/lib/FixedScrollable.svelte` to fix familiar problems:

```ts
  ...
  let topIndicator: HTMLDivElement | undefined = $state();
  let bottomIndicator: HTMLDivElement | undefined = $state();
  let topShadow: HTMLDivElement | undefined = $state();
  let bottomShadow: HTMLDivElement | undefined = $state();

  ...

  function intersectionCallback(shadow: HTMLDivElement | undefined) {
    return ...;
  }

  ...

  onMount(() => {
    if (!topIndicator || !bottomIndicator) {
      throw new Error("Scrollable component not mounted");
    }
    ...
  });
```

It turns out `intersectionCallback` already handles `undefined`, so that function doesn't need to change at all.

We check to see if the markdown component really renders fine despite the type errors. Unfortunately, Storybook can't even start:

```
✘ [ERROR] Could not resolve "storybook/internal/preview-api"

    node_modules/@storybook/svelte/dist/index.mjs:3:163:
      3 │ ...Story as composeStory$1, composeStories as composeStories$1 } from 'storybook/internal/preview-api';
        ╵                                                                       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  You can mark the path "storybook/internal/preview-api" as external to exclude it from the bundle,
  which will remove this error and leave the unresolved path in the bundle.

✘ [ERROR] Could not resolve "storybook/internal/core-events"

    node_modules/@storybook/svelte/dist/chunk-2VFJ3RAK.mjs:2:33:
      2 │ import { RESET_STORY_ARGS } from 'storybook/internal/core-events';
        ╵                                  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  You can mark the path "storybook/internal/core-events" as external to exclude it from the bundle,
  which will remove this error and leave the unresolved path in the bundle.

✘ [ERROR] Could not resolve "storybook/internal/preview-api"

    node_modules/@storybook/svelte/dist/chunk-2VFJ3RAK.mjs:3:51:
      3 │ import { addons, sanitizeStoryContextUpdate } from 'storybook/internal/preview-api';
        ╵                                                   
```

It seems Storybook v8 [supports Svelte 5](https://github.com/storybookjs/storybook/issues/25178), so we'll commit the changes we have so far (skipping pre-commit verification due to the remaining type error) and try upgrading Storybook.

We replace all instances of `7.5.1` (the previous version of Storybook we were using) with `8.4.2` (the latest version of Storybook) in `src-svelte/package.json` and restart Storybook. Safari shows a blank page with 404 errors for resources such as `http://localhost:6006/sb-manager/chunk-2IXBUOFS.js`, but it displays just fine in Firefox. We [remove](https://usqassist.custhelp.com/app/answers/detail/a_id/3199/~/how-do-i-clear-my-cache%2C-cookies-and-history-in-safari%3F) the website data for `localhost` in Safari, and it works fine.

We try to run our frontend tests. We are met with

```

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Error ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
Error: Your application, or one of its dependencies, imported from 'svelte/internal', which was a private module used by Svelte 4 components that no longer exists in Svelte 5. It is not intended to be public API. If you're a library author and you used 'svelte/internal' deliberately, please raise an issue on https://github.com/sveltejs/svelte/issues detailing your use case.
 ❯ ../node_modules/svelte/src/internal/index.js:3:7

 ❯ ViteNodeRunner.runModule ../node_modules/vite-node/dist/client.mjs:399:11
 ❯ ViteNodeRunner.directRequest ../node_modules/vite-node/dist/client.mjs:381:16
 ❯ ViteNodeRunner.cachedRequest ../node_modules/vite-node/dist/client.mjs:206:14
 ❯ ViteNodeRunner.dependencyRequest ../node_modules/vite-node/dist/client.mjs:259:12
 ❯ src/lib/test-helpers.ts:3:31
 ❯ ViteNodeRunner.runModule ../node_modules/vite-node/dist/client.mjs:399:5
 ❯ ViteNodeRunner.directRequest ../node_modules/vite-node/dist/client.mjs:381:5
 ❯ ViteNodeRunner.cachedRequest ../node_modules/vite-node/dist/client.mjs:206:14
 ❯ ViteNodeRunner.dependencyRequest ../node_modules/vite-node/dist/client.mjs:259:12
```

This may be because our `src-svelte/vitest.config.ts` has the line

```ts
export default defineConfig({
  ...,
  test: {
    ...,
    alias: [{ find: /^svelte$/, replacement: "svelte/internal" }],
    ...
  },
});

```

We see [the original reason](/general-notes/setup/tauri/vitest.md) for this alias, and find that the [associated issue](https://github.com/vitest-dev/vitest/issues/2834) is still open. We try removing it anyways... and a whole lot of tests fail. Then the test runner hangs on the `beforeEach` part of `src-svelte/src/routes/PageTransition.test.ts`. We enforce a timeout on that function:

```ts
  beforeEach(() => {
    ...
  }, 1_000);
```

It still does not help when we're running all tests. We try to run this test file specifically, and it quickly errors out with

```

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 3 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on initial visit
 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on visit to new page
 FAIL  src/routes/PageTransition.test.ts > PageTransition > should unset first page load on visit to old page
TypeError: element.animate is not a function
 ❯ animate ../node_modules/svelte/src/internal/client/dom/elements/transitions.js:377:26
    375|   // for bidirectional transitions, we start from the current pos…
    376|   // rather than doing a full intro/outro
    377|   var t1 = counterpart?.t() ?? 1 - t2;
       |                          ^
    378|   counterpart?.abort();
    379|
 ❯ Object.in ../node_modules/svelte/src/internal/client/dom/elements/transitions.js:243:12
 ❯ ../node_modules/svelte/src/internal/client/dom/elements/transitions.js:298:54
 ❯ Module.untrack ../node_modules/svelte/src/internal/client/runtime.js:874:10
 ❯ ../node_modules/svelte/src/internal/client/dom/elements/transitions.js:298:27
 ❯ update_reaction ../node_modules/svelte/src/internal/client/runtime.js:329:56
 ❯ update_effect ../node_modules/svelte/src/internal/client/runtime.js:457:18
 ❯ flush_queued_effects ../node_modules/svelte/src/internal/client/runtime.js:547:4
 ❯ flush_queued_root_effects ../node_modules/svelte/src/internal/client/runtime.js:528:4
 ❯ flush_sync ../node_modules/svelte/src/internal/client/runtime.js:700:3
```

We find out from [this comment](https://github.com/jsdom/jsdom/issues/3429#issuecomment-1251051054) that there is `jsdom-testing-mocks`. We try installing it:

```bash
$ yarn add -D jsdom-testing-mocks
```

and follow [its documentation](https://github.com/trurl-master/jsdom-testing-mocks?tab=readme-ov-file#mock-web-animations-api). We need to enable globals, but that is already true in `src-svelte/vitest.config.ts`. Now it fails with this instead:

```

 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on initial visit
AssertionError: expected false to deeply equal true

- Expected
+ Received

- true
+ false

 ❯ src/routes/PageTransition.test.ts:55:32
     53|
     54|   it("should set first page load on initial visit", async () => {
     55|     expect(get(firstPageLoad)).toEqual(true);
       |                                ^
     56|   });
     57|
```

While debugging this, we realize that further down, we see

```

 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on initial visit
 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on visit to new page
 FAIL  src/routes/PageTransition.test.ts > PageTransition > should unset first page load on visit to old page
TypeError: Cannot access private method
 ❯ __accessCheck ../node_modules/jsdom-testing-mocks/dist/esm/index.js:5:11
      3|     throw TypeError("Cannot " + msg);
      4| };
      5| var __privateGet = (obj, member, getter) => {
       |           ^
      6|   __accessCheck(obj, member, "read from private field");
      7|   return getter ? getter.call(obj) : member.get(obj);
```

We try removing that package and doing the mock ourselves in `src-svelte/src/routes/PageTransition.test.ts`:

```ts
...
import { vi } from "vitest";
...

describe("PageTransition", () => {
  ...

  beforeAll(() => {
    HTMLElement.prototype.animate = vi.fn();
  });

  ...
});
```

Now we get

```

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 3 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on initial visit
 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on visit to new page
 FAIL  src/routes/PageTransition.test.ts > PageTransition > should unset first page load on visit to old page
TypeError: Cannot set properties of undefined (setting 'onfinish')
 ❯ animate ../node_modules/svelte/src/internal/client/dom/elements/transitions.js:379:21
    378|   counterpart?.abort();
    379|
    380|   var delta = t2 - t1;
       |                    ^
    381|   var duration = /** @type {number} */ (options.duration) * Math.…
    382|   var keyframes = [];
```

We edit it further

```ts
  beforeAll(() => {
    HTMLElement.prototype.animate = vi.fn().mockReturnValue({
      onfinish: null,
      cancel: vi.fn(),
    });
  });
```

and now our tests finally fail with just

```

 FAIL  src/routes/PageTransition.test.ts > PageTransition > should set first page load on initial visit
AssertionError: expected false to deeply equal true

- Expected
+ Received

- true
+ false

 ❯ src/routes/PageTransition.test.ts:58:32
     56|
     57|   it("should set first page load on initial visit", async () => {
     58|     expect(get(firstPageLoad)).toEqual(true);
       |                                ^
     59|   });
     60|
```

Surprisingly, `onMount` runs in `src-svelte/src/routes/PageTransition.svelte` -- a `console.log` statement there successfully prints. So it appears the old problem around `onMount` no longer applies. We try to update that file to have

```ts
  $effect(() => {
    checkFirstPageLoad(currentRoute);
    updateFlyDirection(currentRoute);
  });
```

instead of the automatically generated

```ts
  run(() => {
    checkFirstPageLoad(currentRoute);
  });
  run(() => {
    updateFlyDirection(currentRoute);
  });
```

Now all tests pass except for the very first one, where no navigation happens. It turns out this is because `checkFirstPageLoad` is invoked twice now: once in `onMount` and again as part of the first effect.

We try running the actual app, and this is where we see

```
Unhandled Promise Rejection: Error: Vitest failed to access its internal state.

One of the following is possible:
- "vitest" is imported directly without running "vitest" command
- "vitest" is imported inside "globalSetup" (to f...

Unhandled Promise Rejection: ReferenceError: Cannot access uninitialized variable.
```

It turns out this is because VS Code automatically (unhelpfully) imported `vi` for us in `src-svelte/src/routes/PageTransition.svelte`.

While checking our `package.json`, we realize that `Svelte` has somehow made it as a dev dependency. We move `Svelte` from `devDependencies` into regular dependencies in `src-svelte/package.json`.

Other tests still hang:

```

^Cmake: *** [test] Interrupt: 2

node:events:492
      throw er; // Unhandled 'error' event
      ^

Error: Worker exited unexpectedly
    at ChildProcess.<anonymous> (file:///Users/amos/Documents/zamm/node_modules/tinypool/dist/index.js:139:34)
    at ChildProcess.emit (node:events:526:35)
    at ChildProcess._handle.onexit (node:internal/child_process:294:12)
Emitted 'error' event on Tinypool instance at:
    at EventEmitterReferencingAsyncResource.runInAsyncScope (node:async_hooks:206:9)
    at Tinypool.emit (node:events:169:18)
    at file:///Users/amos/Documents/zamm/node_modules/tinypool/dist/index.js:746:30
    at ChildProcess.<anonymous> (file:///Users/amos/Documents/zamm/node_modules/tinypool/dist/index.js:206:16)
    at ChildProcess.emit (node:events:514:28)
    at ChildProcess.<anonymous> (file:///Users/amos/Documents/zamm/node_modules/tinypool/dist/index.js:139:20)
    at ChildProcess.emit (node:events:526:35)
    at ChildProcess._handle.onexit (node:internal/child_process:294:12)

Node.js v20.5.1
```

We try to manually test the files, starting with `src/lib/snackbar/Snackbar.test.ts`. We see the same `element.animate is not a function` error here. We try to refactor the element animation code to `src-svelte/src/lib/test-helpers.ts`, but then we get

```

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Error ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
Error: Vitest failed to access its internal state.

One of the following is possible:
- "vitest" is imported directly without running "vitest" command
- "vitest" is imported inside "globalSetup" (to fix this, use "setupFiles" instead, because "globalSetup" runs in a different context)
- Otherwise, it might be a Vitest bug. Please report it to https://github.com/vitest-dev/vitest/issues

 ❯ getWorkerState ../node_modules/vitest/dist/chunks/utils.C8RiOc4B.js:8:11
 ❯ getCurrentEnvironment ../node_modules/vitest/dist/chunks/utils.C8RiOc4B.js:22:17
 ❯ createExpect ../node_modules/vitest/dist/chunks/vi.JMQoNY_Z.js:439:20
 ❯ ../node_modules/vitest/dist/chunks/vi.JMQoNY_Z.js:484:22
 ❯ ModuleJob.run node:internal/modules/esm/module_job:192:25
 ❯ DefaultModuleLoader.import node:internal/modules/esm/loader:228:24
 ❯ ViteNodeRunner.interopedImport ../node_modules/vite-node/dist/client.mjs:421:28
 ❯ ViteNodeRunner.directRequest ../node_modules/vite-node/dist/client.mjs:280:24
 ❯ ViteNodeRunner.cachedRequest ../node_modules/vite-node/dist/client.mjs:206:14
 ❯ ViteNodeRunner.dependencyRequest ../node_modules/vite-node/dist/client.mjs:259:12
```

We undo and simply copy-paste the code instead to `src-svelte/src/lib/snackbar/Snackbar.test.ts`. Now we get:

```

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Errors ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

Vitest caught 1 unhandled error during the test run.
This might cause false positive tests. Resolve unhandled errors to make sure your tests are not affected.

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Uncaught Exception ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
TypeError: element.getAnimations is not a function
 ❯ Object.fix ../node_modules/svelte/src/internal/client/dom/elements/transitions.js:127:16
    125|
    126|    // It's important to destructure these to get fixed values - t…
    127|    // and changing the style to 'absolute' can for example influe…
       |                ^
    128|    var { position, width, height } = getComputedStyle(element);
    129|
 ❯ reconcile ../node_modules/svelte/src/internal/client/dom/blocks/each.js:423:23
 ❯ ../node_modules/svelte/src/internal/client/dom/blocks/each.js:192:4
 ❯ update_reaction ../node_modules/svelte/src/internal/client/runtime.js:329:56
```

So then we add

```
  beforeAll(() => {
    ...
    HTMLElement.prototype.getAnimations = vi.fn().mockReturnValue([]);
  });
```

But then the test fails with

```

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/lib/snackbar/Snackbar.test.ts > Snackbar > should hide a message if the dismiss button is clicked
Error: expect(element).not.toBeInTheDocument()

expected document not to contain element, found <div
  class="snackbar error svelte-1j4qjll"
  role="alertdialog"
>
  This is a test message
  <button
    class="svelte-1j4qjll"
    title="Dismiss"
  >
    <svg
      height="1.2em"
      viewBox="0 0 1024 1024"
      width="1.2em"
    >
      <path
        d="M195.2 195.2a64 64 0 0 1 90.496 0L512 421.504L738.304 195.2a64 64 0 0 1 90.496 90.496L602.496 512L828.8 738.304a64 64 0 0 1-90.496 90.496L512 602.496L285.696 828.8a64 64 0 0 1-90.496-90.496L421.504 512L195.2 285.696a64 64 0 0 1 0-90.496"
        fill="currentColor"
      />
      <!---->
    </svg>
    <!---->
  </button>
</div> instead
```

We see that [this issue](https://github.com/testing-library/svelte-testing-library/issues/206), which we previously referenced when creating [the dashboard](/zamm-notes/dashboard.md), has been closed. Unfortunately, it seems that it was closed by simply testing `on:introend`.

We try to log where the other function calls are being invoked from:

```
stderr | src/lib/snackbar/Snackbar.test.ts > Snackbar > should hide a message if the dismiss button is clicked
Trace:
    at HTMLDivElement.<anonymous> (/Users/amos/Documents/zamm/src-svelte/src/lib/snackbar/Snackbar.test.ts:13:15)
    at HTMLDivElement.mockCall (file:///Users/amos/Documents/zamm/node_modules/@vitest/spy/dist/index.js:61:17)
    at HTMLDivElement.spy [as animate] (file:///Users/amos/Documents/zamm/node_modules/tinyspy/dist/index.js:45:80)
    at animate (/Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/dom/elements/transitions.js:377:26)
    at Object.out (/Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/dom/elements/transitions.js:262:12)
    at Module.run_out_transitions (/Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/reactivity/effects.js:521:15)
    at pause_effects (/Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/dom/blocks/each.js:74:24)
    at reconcile (/Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/dom/blocks/each.js:427:4)
    at /Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/dom/blocks/each.js:192:4
    at update_reaction (/Users/amos/Documents/zamm/node_modules/svelte/src/internal/client/runtime.js:329:56)
```

We eventually find [someone else](https://github.com/testing-library/svelte-testing-library/issues/284#issuecomment-2434162767) with the same problem. Unfortunately the provided solution does not work -- but then we realize that the Storybook story isn't quite working either.

We try to commit the previous changes first before experimenting further. Now we run into

```

/Users/amos/Documents/zamm/src-svelte/src/routes/settings/Settings.stories.ts:39:3
Error: Type '(story: StoryFn) => { Component: Component<Props, {}, "">; slot: AnnotatedStoryFn<SvelteRenderer<SvelteComponent<Record<string, any>, any, any>>, Args>; }' is not assignable to type 'DecoratorFunction<SvelteRenderer<SvelteComponent<Record<string, any>, any, any>>, { [x: string]: any; }>'.
  Call signature return types '{ Component: Component<Props, {}, "">; slot: AnnotatedStoryFn<SvelteRenderer<SvelteComponent<Record<string, any>, any, any>>, Args>; }' and 'SvelteStoryResult<any, any>' are incompatible.
    The types of 'Component' are incompatible between these types.
      Type 'Component<Props, {}, "">' is not assignable to type 'ComponentType<any, any>'.
FullPage.decorators = [
  (story: StoryFn) => {
    return {
```

We can't find much on this online. However, we did find that we should've perhaps looked at the [Storybook 8 migration guide](https://storybook.js.org/docs/migration-guide) first.

```
$ npx storybook@latest upgrade
╭ Migration check ran successfully ───────────────────────────────────────╮
│                                                                         │
│   Successful migrations:                                                │
│                                                                         │
│   remove-jest-testing-library, wrap-require, autodocs-tags              │
│                                                                         │
│   Manual migrations:                                                    │
│                                                                         │
│   remove-argtypes-regex, remove-react-dependency                        │
│                                                                         │
│   Skipped migrations:                                                   │
│                                                                         │
│   visual-tests-addon                                                    │
│                                                                         │
│   ─────────────────────────────────────────────────                     │
│                                                                         │
│   If you'd like to run the migrations again, you can do so by running   │
│   'npx storybook automigrate'                                           │
│                                                                         │
│   The automigrations try to migrate common patterns in your project,    │
│   but might not contain everything needed to migrate to the latest      │
│   version of Storybook.                                                 │
│                                                                         │
│   Please check the changelog and migration guide for manual             │
│   migrations and more information:                                      │
│   https://storybook.js.org/docs/migration-guide                         │
│   And reach out on Discord if you need help:                            │
│   https://discord.gg/storybook                                          │
│                                                                         │
╰─────────────────────────────────────────────────────────────────────────╯
```

Unfortunately, it seems this doesn't do too much in the way of useful things. A lot of dependencies are added that we want in our top-level `package.json` instead, such as `prettier` and `eslint`, and it also adds it to regular dependencies instead of `devDependencies`.

The [main documentation](https://storybook.js.org/docs/writing-stories) on stories for Svelte now seems to only mention `@storybook/addon-svelte-csf` as a way to include decorators for specific stories. We try to add

```bash
$ yarn add -D @storybook/addon-svelte-csf
```

We follow the README documentation for that add-on and edit `src-svelte/.storybook/main.ts`:

```ts
const config: StorybookConfig = {
  stories: ["../src/**/*.stories.@(...|svelte)"],
  addons: [
    getAbsolutePath("@storybook/addon-svelte-csf"),
    ...
  ],
  ...
};
```

We get

```

SB_CORE-SERVER_0002 (CriticalPresetLoadError): Storybook failed to load the following preset: ./.storybook/main.ts.

Please check whether your setup is correct, the Storybook dependencies (and their peer dependencies) are installed correctly and there are no package version clashes.

If you believe this is a bug, please open an issue on Github.

SB_CORE-SERVER_0002 (CriticalPresetLoadError): Storybook failed to load the following preset: /Users/amos/Documents/zamm/node_modules/@storybook/addon-svelte-csf/dist/index.js.

Please check whether your setup is correct, the Storybook dependencies (and their peer dependencies) are installed correctly and there are no package version clashes.

If you believe this is a bug, please open an issue on Github.

/Users/amos/Documents/zamm/node_modules/@storybook/addon-svelte-csf/dist/components/Meta.svelte:1
<script>
^

SyntaxError: Unexpected token '<'
    at internalCompileFunction (node:internal/vm:73:18)
    at wrapSafe (node:internal/modules/cjs/loader:1153:20)
    at Module._compile (node:internal/modules/cjs/loader:1197:27)
    at Module._extensions..js (node:internal/modules/cjs/loader:1287:10)
    at Object.newLoader (/Users/amos/Documents/zamm/node_modules/esbuild-register/dist/node.js:2262:9)
    at extensions..js (/Users/amos/Documents/zamm/node_modules/esbuild-register/dist/node.js:4838:24)
    at Module.load (node:internal/modules/cjs/loader:1091:32)
    at Module._load (node:internal/modules/cjs/loader:938:12)
    at Module.require (node:internal/modules/cjs/loader:1115:19)
    at require (node:internal/modules/helpers:130:18)

More info:

    at loadPreset (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/common/index.cjs:16477:13)

More info:

    at loadPreset (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/common/index.cjs:16477:13)
    at async Promise.all (index 1)
    at async loadPresets (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/common/index.cjs:16487:55)
    at async getPresets (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/common/index.cjs:16520:11)
    at async buildDevStandalone (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:37145:11)
    at async withTelemetry (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:35757:12)
    at async dev (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/cli/bin/index.cjs:2591:3)
    at async s.<anonymous> (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/cli/bin/index.cjs:2643:74)

WARN Broken build, fix the error above.
WARN You may need to refresh the browser.
```

We try to instead use version `5.0.0-next.11` of the add-on, since it is the latest version available on their GitHub page.

We try editing `src-svelte/src/routes/settings/Settings.stories.svelte` as such:

```svelte
<script>
  import Settings from "./Settings.svelte";
  import MockPageTransitions from "$lib/__mocks__/MockPageTransitions.svelte";
  import { Template, Story } from '@storybook/addon-svelte-csf';

  export const meta = {
    title: "Screens/Settings",
    component: Settings,
  }
</script>

<Story name="TinyPhoneScreen">
  <Settings />
</Story>

```

But on the new version, we follow [the in-progress migration guide](https://github.com/storybookjs/addon-svelte-csf/blob/73be7827ec3e3e30e9fc0b95d0cdf6f4c70319e7/MIGRATION.md) to do this:

```
Unable to index ./src/routes/settings/Settings.stories.svelte:
  Error: Invariant failed: No matching indexer found for /Users/amos/Documents/zamm/src-svelte/src/routes/settings/Settings.stories.svelte
    at invariant (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34548:11)
    at $n.extractStories (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34748:5)
    at /Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34693:53
    at /Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34669:30
    at Array.map (<anonymous>)
    at /Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34666:26
    at Array.map (<anonymous>)
    at $n.updateExtracted (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34658:23)
    at $n.ensureExtracted (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34692:16)
    at $n.getIndexAndStats (/Users/amos/Documents/zamm/node_modules/@storybook/core/dist/core-server/index.cjs:34914:108)

If you are in development, this likely indicates a problem with your Storybook process,
check the terminal for errors.

If you are in a deployed Storybook, there may have been an issue deploying the full Storybook
build.
```

We notice in the terminal log output:

```
WARN Could not resolve addon "/Users/amos/Documents/zamm/node_modules/@storybook/addon-svelte-csf", skipping. Is it installed?
```

A `yarn install` fails to fix the problem. We abandon this approach since it seems this add-on is not ready for Storybook 8 yet. We try updating `src-svelte/src/routes/settings/Settings.stories.ts` for the new way of exporting a default of type `Meta`, but even doing

```ts
const meta: Meta<typeof SettingsComponent> = {
  title: "Screens/Settings",
  component: SettingsComponent,
};
```

gives us the error

```
/Users/amos/Documents/zamm/src-svelte/src/routes/settings/Settings.stories.ts:11:3
Error: Type '__sveltets_2_IsomorphicComponent<Record<string, never>, { [evt: string]: CustomEvent<any>; }, {}, {}, string>' is not assignable to type 'ComponentType<__sveltets_2_IsomorphicComponent<Record<string, never>, { [evt: string]: CustomEvent<any>; }, {}, {}, string>, any>'.
  Types of parameters 'options' and 'options' are incompatible.
    Type 'ComponentConstructorOptions<__sveltets_2_IsomorphicComponent<Record<string, never>, { [evt: string]: CustomEvent<any>; }, {}, {}, string>>' is not assignable to type 'ComponentConstructorOptions<Record<string, never>>'.
      Type '__sveltets_2_IsomorphicComponent<Record<string, never>, { [evt: string]: CustomEvent<any>; }, {}, {}, string>' is not assignable to type 'Record<string, never>'.
        Index signature for type 'string' is missing in type '__sveltets_2_IsomorphicComponent<Record<string, never>, { [evt: string]: CustomEvent<any>; }, {}, {}, string>'.
  title: 'Screens/Settings',
  component: SettingsComponent,
};
```

We simply export the meta the usual way. The only way to fix the errors appears to be creating `src-svelte/src/lib/__mocks__/decorators.ts` to force TypeScript to recognize the decorator as a `DecoratorFunction<SvelteRenderer>`:

```ts
import type { DecoratorFunction, PartialStoryFn } from "storybook/internal/types";
import MockPageTransitions from "./MockPageTransitions.svelte";
import type { SvelteRenderer } from "@storybook/svelte";

const mockPageTransitionFn = (story: PartialStoryFn) => {
  return {
    Component: MockPageTransitions,
    slot: story,
  };
};
export const MockPageTransitionsDecorator: DecoratorFunction<SvelteRenderer> = mockPageTransitionFn as unknown as DecoratorFunction<SvelteRenderer>;
```

Then we import that in `src-svelte/src/routes/settings/Settings.stories.ts`:

```ts
import { MockPageTransitionsDecorator } from "$lib/__mocks__/decorators";

...

export const FullPage: StoryObj = ...;
FullPage.decorators = [MockPageTransitionsDecorator];
```

We have to do the same thing for the `MountTransition` story in `src-svelte/src/routes/database/api-calls/new/import/ApiCallImport.stories.ts`, but for that one we need to edit `src-svelte/src/lib/__mocks__/decorators.ts` to add another decorator:

```ts
...
import MockTransitions from "./MockTransitions.svelte";
...

const mockTransitionFn = (story: PartialStoryFn) => {
  return {
    Component: MockTransitions,
    slot: story,
  };
};
export const MockTransitionsDecorator: DecoratorFunction<SvelteRenderer> =
  mockTransitionFn as unknown as DecoratorFunction<SvelteRenderer>;
```

We do the same for:

- `src-svelte/src/routes/credits/Credits.stories.ts`
- `src-svelte/src/lib/InfoBox.stories.ts`, which uses both types of transitions defined so far

We then look at the next error:

```
/Users/amos/Documents/zamm/src-svelte/src/lib/sample-call-testing.ts:61:28
Error: Parameter 'item' implicitly has an 'any' type.
  if (isArray(obj)) {
    const items = obj.map((item) => stringify(item));
    return `[${items.join(",")}]`;
```

and edit `src-svelte/src/lib/sample-call-testing.ts` to explicitly type `item`:

```ts
function stringify(...): string {
  if (...) {
    const items = obj.map((item: any) => ...);
    ...
  }
  ...
}
```

Then we see

```
/Users/amos/Documents/zamm/src-svelte/src/lib/sample-call-testing.ts:4:36
Error: Could not find a declaration file for module 'lodash'. '/Users/amos/Documents/zamm/node_modules/lodash/lodash.js' implicitly has an 'any' type.
  Try `npm i --save-dev @types/lodash` if it exists or add a new declaration (.d.ts) file containing `declare module 'lodash';`
import { Convert } from "./sample-call";
import { camelCase, isArray } from "lodash";
```

We do as instructed:

```bash
$ yarn add -D @types/lodash
```

Finally, all TypeScript errors are fixed. We notice [this issue](https://github.com/storybookjs/storybook/issues/27073) with Storybook 8: that viewports are not reset in between story navigation. We take the easy fix that wouldn't involve changing every other file, and edit `src-svelte/.storybook/preview.ts`:

```ts
const preview = {
  parameters: {
    ...,
    viewport: {
      defaultViewport: "reset",
      ...
    },
  },
  ...
};
```

We also notice that various InfoBox-related things are broken now, especially on Safari. We edit `src-svelte/src/lib/InfoBox.svelte` to first remove the `import { run } from "svelte/legacy";` that the migration script introduced:

```ts
  $effect(() => {
    forceUpdateTitleText(title);
  });
```

We notice that the title text flashes -- first the full text is displayed, and then it is cleared and the animation starts. This is because of our fix in [the terminal](/zamm-notes/terminal.md) to make it possible to change the title in the middle of an animation. We edit the code to only directly set the title text if the animation has already finished; otherwise, we just set the attribute from which the animation draws its data:

```ts
  let isAnimatingTitle = false;
  
  ...

  class TypewriterEffect extends SubAnimation<void> {
    constructor(...) {
      ...
      super({
        timing: anim.timing,
        tick: (tLocalFraction: number) => {
          isAnimatingTitle = true;
          ...

          if (tLocalFraction === 1) {
            isAnimatingTitle = false;
          }
        },
      });
    }
  }

  ...

  function forceUpdateTitleText(newTitle: string) {
    if (titleElement) {
      if (isAnimatingTitle) {
        titleElement.setAttribute("data-text", newTitle);
      } else {
        titleElement.textContent = newTitle;
      }
    }
  }
```

Next, we realize that we can fix the InfoBox border outline issue for once, because Safari sometimes fixes the border box render. We see that there is [this answer](https://stackoverflow.com/a/3485654) for forcing DOM rerenders, which links to [this resource](https://gist.github.com/paulirish/5d52fb081b3570c81e3a). One of the comments mention using `inline-block` to reduce flicker, so that is what we do. We set the node type to `HTMLDivElement` so that there is no error around the `style` attribute:

```ts
  function revealOutline(
    node: HTMLDivElement,
    ...
  ): TransitionConfig {
    ...

    return {
      ...
      tick: (tGlobalFraction: number) => {
        ...
        if (tGlobalFraction === 1) {
          node.style.display = "inline-block";
          setTimeout(() => {
            node.removeAttribute("style");
          }, 10);
        }
      },
    };
  }
```

For completeness sake, we clean up after ourselves with element opacity:

```ts
  class RevealContent extends SubAnimation<void> {
    constructor(...) {
      ...
      super({
        timing: anim.timing,
        tick: (tLocalFraction: number) => {
          ...
          if (...) {
            ...
          } else if (tLocalFraction >= 0.9) {
            ...

            if (tLocalFraction === 1) {
              anim.node.style.removeProperty("opacity");
            }
          }
        },
      });
    }
  }
```

We notice that there are problems with the database view on Safari -- all the table rows are visible before the animation completes. It looks fine in Firefox. We try editing `src-svelte/src/lib/Table.svelte` until we finally understand that Safari in particular somehow does not propagate `opacity: 0;` to the children of the `<a>` element. As such, we apply it to the `<div>` inside that tag.

```svelte
<div ...>
  <div class="blurb header atomic-reveal">
    ...
  </div>
  <div class="scrollable-container ... composite-reveal">
    <Scrollable ...>
      ...
          <a ... class="composite-reveal">
            <div class="blurb instance atomic-reveal">
```

We also edit `src-svelte/src/lib/controls/Select.svelte` so that the select element could fade in too:

```svelte
<div class="setting atomic-reveal">
  ...
</div>
```

We notice that there are still problems with content reveal on the API calls page, so we refactor the code to be a little simpler:

```ts
    const getNodeAnimations = (
      currentNode: Element,
      ...
    ): RevealContent[] => {
      ...

      let isAtomicNode = false;
      if (currentNode.classList.contains("atomic-reveal")) {
        isAtomicNode = true;
      } else if (currentNode.classList.contains("composite-reveal")) {
        isAtomicNode = false;
      } else if (currentNode.tagName === "TBODY") {
        isAtomicNode = false;
      } else {
        isAtomicNode = true;
      }

      ...
    }
```

### Page transitions

Page transition animations are also broken. We edit `src-svelte/src/routes/PageTransition.svelte` to properly introduce `TransitionType.Init` (which is only ever the transition state on app startup and never gets assigned again thereafter) instead of relying on `await tick();` as before to set the initial transition duration on app startup:

```svelte
  ...

  export enum TransitionType {
    // user just opened the app
    Init,
    ...
  }

  ...

  let transitions: Transitions = $state(getTransitions(TransitionType.Init));
  ...

  onMount(async () => {
    ready = true;
    setTimeout(() => {
      firstAppLoad.set(false);
    }, $standardDuration);

    pageTransition.set({
      ...
    });
  });

  ...

  function getTransitions(transitionType: TransitionType) {
    ...

    switch (transitionType) {
      ...
      case TransitionType.Init:
        return {
          out: {
            // doesn't matter for init
            x: "-20%",
            duration: 0,
            easing: cubicIn,
            delay: 0,
          },
          in: {
            x: "-20%",
            duration: $standardDuration,
            easing: backOut,
            delay: 0,
          },
        };
    }
  }

  function updateFlyDirection(route: string) {
    ...

    if (oldRoute === route) {
      return;
    }

    ...
  }

  $effect.pre(() => {
    checkFirstPageLoad(...);
    updateFlyDirection(...);
  });
```

For manual testing purposes, we also edit `src-svelte/src/lib/__mocks__/MockPageTransitions.svelte` to put the store updates in an `onMount` so that they get executed on Storybook's story reload:

```ts
  ...
  import { onMount } from "svelte";
  ...

  onMount(() => {
    firstAppLoad.set(true);
    firstPageLoad.set(true);
    animationSpeed.set(0.1);
    transparencyOn.set(true);
  });

  ...
```
