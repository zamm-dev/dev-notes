# Setting up Storybook with SvelteKit

Follow the instructions [here](https://storybook.js.org/docs/svelte/get-started/install):

```bash
$ npx storybook@latest init
Need to install the following packages:
  storybook@7.4.0
Ok to proceed? (y) y

 storybook init - the simplest way to add a Storybook to your project. 

 • Detecting project type. ✓
 • Preparing to install dependencies. ✓


yarn install v1.22.19
[1/4] Resolving packages...
success Already up-to-date.
Done in 0.36s.
. ✓
 • Adding Storybook support to your "SvelteKit" app
  ✔ Getting the correct version of 10 packages
  ✔ Installing Storybook dependencies
. ✓
 • Preparing to install dependencies. ✓


yarn install v1.22.19
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
warning " > @typescript-eslint/eslint-plugin@5.62.0" has incorrect peer dependency "@typescript-eslint/parser@^5.0.0".
warning "workspace-aggregator-6111915a-9d82-4040-9d33-56d36c9385f4 > webdriver > ts-node@10.9.1" has unmet peer dependency "@types/node@*".
warning Workspaces can only be enabled in private projects.
warning Workspaces can only be enabled in private projects.
[4/4] Building fresh packages...
Done in 3.82s.
. ✓
╭─────────────────────────────────────────────────────────────────────────╮│                                                                         ││   Storybook was successfully installed in your project! 🎉              ││   To run Storybook manually, run yarn storybook. CTRL+C to stop.        ││                                                                         ││   Wanna know more about Storybook? Check out                            ││   https://storybook.js.org/                                             ││   Having trouble or want to chat? Join us at                            ││   https://discord.gg/storybook/                                         ││                                                                         │╰─────────────────────────────────────────────────────────────────────────╯

Running Storybook
yarn run v1.22.19
$ storybook dev -p 6006 --quiet
@storybook/cli v7.4.0

info => Starting manager..
The following Vite config options will be overridden by SvelteKit:
  - base

Error: We've detected a SvelteKit project using the @storybook/svelte-vite framework, which is not supported in Storybook 7.0
Please use the @storybook/sveltekit framework instead.
You can migrate automatically by running

npx storybook@latest automigrate

See https://github.com/storybookjs/storybook/blob/next/MIGRATION.md#sveltekit-needs-the-storybooksveltekit-framework
    at handleSvelteKit (/root/zamm/node_modules/@storybook/svelte-vite/dist/preset.js:1:1635)
    at async viteFinal (/root/zamm/node_modules/@storybook/svelte-vite/dist/preset.js:9:2478)
    at async Object.viteFinal (/root/zamm/node_modules/@storybook/sveltekit/dist/preset.js:1:1319)
    at async createViteServer (/root/zamm/node_modules/@storybook/builder-vite/dist/index.js:159:10530)
    at async Module.start (/root/zamm/node_modules/@storybook/builder-vite/dist/index.js:159:12527)
    at async storybookDevServer (/root/zamm/node_modules/@storybook/core-server/dist/index.js:101:8374)
    at async buildDevStandalone (/root/zamm/node_modules/@storybook/core-server/dist/index.js:116:3013)
    at async withTelemetry (/root/zamm/node_modules/@storybook/core-server/dist/index.js:101:4155)
    at async dev (/root/zamm/node_modules/@storybook/cli/dist/generate.js:502:401)
    at async Command.<anonymous> (/root/zamm/node_modules/@storybook/cli/dist/generate.js:504:225)

WARN Broken build, fix the error above.
WARN You may need to refresh the browser.

error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

Let's listen to the error and do this:

```bash
$ npx storybook@latest automigrate
🔎 checking possible migrations..

🔎 found a 'github-flavored-markdown-mdx' migration:
╭ Automigration detected ─────────────────────────────────────────────────╮│                                                                         ││   In MDX1 you had the option of using GitHub flavored markdown.         ││                                                                         ││   Storybook 7.0 uses MDX2 for compiling MDX, and thus no longer         ││   supports GFM out of the box.                                          ││   Because of this you need to explicitly add the GFM plugin in the      ││   addon-docs options:                                                   ││   https://storybook.js.org/docs/react/writing-docs/mdx#lack-of-github   ││   -flavored-markdown-gfm                                                ││                                                                         ││   We recommend you follow the guide on the link above, however we can   ││   add a temporary storybook addon that helps make this migration        ││   easier.                                                               ││   We'll install the addon and add it to your storybook config.          ││                                                                         │╰─────────────────────────────────────────────────────────────────────────╯
✔ Do you want to run the 'github-flavored-markdown-mdx' migration on your project? … yes
✅ Adding "@storybook/addon-mdx-gfm" addon
✅ ran github-flavored-markdown-mdx migration

🔎 found a 'wrap-require' migration:
╭ Automigration detected ─────────────────────────────────────────────────╮│                                                                         ││   We have detected that you're using Storybook 7.4.0 in a monorepo      ││   project.                                                              ││   For Storybook to work correctly, some fields in your main config      ││   must be updated. We can do this for you automatically.                ││                                                                         ││   More info: https://storybook.js.org/docs/react/faq#how-do-i-fix-mod   ││   ule-resolution-in-special-environments                                ││                                                                         │╰─────────────────────────────────────────────────────────────────────────╯
✔ Do you want to run the 'wrap-require' migration on your project? … yes
✅ ran wrap-require migration

╭ Migration check ran successfully ───────────────────────────────────────╮│                                                                         ││   Successful migrations:                                                ││                                                                         ││   github-flavored-markdown-mdx, wrap-require                            ││                                                                         ││   ─────────────────────────────────────────────────                     ││                                                                         ││   If you'd like to run the migrations again, you can do so by running   ││   'npx storybook@next automigrate'                                      ││                                                                         ││   The automigrations try to migrate common patterns in your project,    ││   but might not contain everything needed to migrate to the latest      ││   version of Storybook.                                                 ││                                                                         ││   Please check the changelog and migration guide for manual             ││   migrations and more information:                                      ││   https://storybook.js.org/migration-guides/7.0                         ││   And reach out on Discord if you need help:                            ││   https://discord.gg/storybook                                          ││                                                                         │╰─────────────────────────────────────────────────────────────────────────╯

```

It still doesn't work. We see that there is [an issue](https://github.com/storybookjs/storybook/issues/23777). Follow the instructions there and edit `src-svelte/.storybook/main.ts` to refer to:

```ts
const config = {
  ...
  framework: {
    name: '@storybook/sveltekit',
    options: {},
  },
  ...
};
export default config;
```

(This problem crops up again after we update Storybook from 7.4.0 to 7.5.1, so it appears we have to do this after every Storybook update.)

Next, follow the instructions [here](https://www.thisdot.co/blog/integrating-storybook-with-sveltekit-typescript-and-scss/). Create a file `src-svelte/src/routes/api_keys_display.stories.ts`:

```ts
import ApiKeysDisplay from './api_keys_display.svelte';

export default {
    component: ApiKeysDisplay,
    title: 'Example/API Keys Display',
    argTypes: {},
};

const Template = ({ ...args }) => ({
    Component: ApiKeysDisplay,
    props: args,
});

export const Default = Template.bind({});

```

Then we go to our storybook and find `API Keys Display > Default`. We see the error

```
invoke() is not a function

getApiKeys@http://localhost:6006/src/lib/bindings.ts:6:18
instance@http://localhost:6006/src/routes/api_keys_display.svelte:359:17
init@http://localhost:6006/node_modules/.cache/sb-vite/deps/chunk-SW64H2IU.js?v=507a655b:2166:23
...
```

Mock the function in the story:

```ts
import type { ApiKeys } from '$lib/bindings';

...

const mock_invoke: (arg: string) => Promise<ApiKeys> = () => Promise.resolve({
  openai: null,
});
window.__TAURI_INVOKE__ = mock_invoke as any;

```

We see that the text now shows up but is completely unstyled. We edit `src-svelte/.storybook/preview.ts`:

```ts
import '../src/routes/styles.css';

...
```

Now the color is there, but the fonts are not. Edit the config again:

```ts
const config = {
  ...
  staticDirs: ['../static'],
  ...
};
```

**Restart** the storybook server. Now it should look just as we expected.

Delete the default components folder because we aren't using any of those.

To avoid the error

```
/root/zamm/src-svelte/.storybook/main.ts
  0:0  warning  File ignored by default.  Use a negated ignore pattern (like "--ignore-pattern '!<relative/path/to/filename>'") to override
```

which is due to [eslint ignoring hidden directories by default](https://stackoverflow.com/a/71829427), edit `src-svelte/.eslintrc.yaml`:

```yaml
...
ignorePatterns:
  ...
  - "!.storybook"
  ...
```

We want to visualize this component in multiple states, but we can only mock one return value for the `mock_invoke` function at a time. We could either:

- try to explicitly pass in the function to the component so that it can be mocked, or
- try to create a separate `*.stories.ts` file for each different version of the function, using [single-story hoisting](https://storybook.js.org/docs/react/writing-stories/naming-components-and-hierarchy#single-story-hoisting) to make sure the tree stays clean

We try the first option, but there are issues with the function getting passed in, and it also annoyingly changes the existing code without testing it as thoroughly as the Vitest.

We try the second option. Rename `src-svelte/src/routes/api_keys_display.stories.ts` to `src-svelte/src/routes/api_keys_display.unknown.stories.ts` and make the export the same name to take advantage of the hoisting:

```ts
...

export default {
  component: ApiKeysDisplay,
  title: "Settings/API Keys Display/Unknown",
  argTypes: {},
};

...

export const Unknown = Template.bind({});
```

Then create `src-svelte/src/routes/api_keys_display.loading.stories.ts` with a long wait:

```ts
import ApiKeysDisplay from "./api_keys_display.svelte";
import type { ApiKeys } from "$lib/bindings";

export default {
  component: ApiKeysDisplay,
  title: "Settings/API Keys Display/Loading",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: ApiKeysDisplay,
  props: args,
});

const mock_invoke: (arg: string) => Promise<ApiKeys> = () =>
  new Promise((resolve) => {
    setTimeout(() => {
      resolve({
        openai: null,
      });
    }, 1_000_000);
  });
window.__TAURI_INVOKE__ = mock_invoke as any;

export const Loading = Template.bind({});
```

It works... until you switch to the other page and `window.__TAURI_INVOKE__` gets overwritten once again.

We look at [this answer](https://stackoverflow.com/a/70115093) instead and find that Storybook does offer a built-in way after all to mock functions. We look at the documentation for [decorators](https://storybook.js.org/docs/react/writing-stories/decorators) and parameters, and create a mock invoke function at `src-svelte/src/lib/__mocks__/invoke.ts`: 

```ts
import type {StoryFn, Decorator, StoryContext} from "@storybook/svelte";

let nextResolution: any;
let nextShouldWait: boolean = false;

window.__TAURI_INVOKE__ = () => {
  return new Promise((resolve) => {
    if (nextShouldWait) {
      setTimeout(() => {
        resolve(nextResolution);
      }, 0); // the re-render never happens, so any timeout is fine
    } else {
      resolve(nextResolution);
    }
  });
}

interface TauriInvokeArgs {
  resolution: any;
  shouldWait?: boolean | undefined;
  [key: string]: any;
}

const tauri_invoke_decorator: Decorator = (story: StoryFn, context: StoryContext) => {
  const { args, parameters } = context;
  const { resolution, shouldWait } = parameters as TauriInvokeArgs;
  nextResolution = resolution;
  nextShouldWait = shouldWait || false;
  return story(args, context);
}

export default tauri_invoke_decorator;
```

We make sure to add that decorator in `src-svelte/.storybook/preview.ts`:

```ts
import "../src/routes/styles.css";
import tauri_invoke_decorator from "../src/lib/__mocks__/invoke";

/** @type { import('@storybook/svelte').Preview } */
const preview = {
  ...
  decorators: [tauri_invoke_decorator],
};

export default preview;

```

Now in the original `src-svelte/src/routes/api_keys_display.stories.ts`, we set the new expected parameters:

```ts
import ApiKeysDisplay from "./api_keys_display.svelte";
import type { ApiKeys } from "$lib/bindings";
import type { StoryObj } from "@storybook/svelte";

export default {
  component: ApiKeysDisplay,
  title: "Settings/API Keys Display",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: ApiKeysDisplay,
  props: args,
});

const unknownKeys: ApiKeys = {
  openai: null,
};

const knownKeys: ApiKeys = {
  openai: {
    value: "sk-1234567890",
    source: "Environment",
  },
};

export const Loading: StoryObj = Template.bind({}) as any;
Loading.parameters = {
  resolution: unknownKeys,
  shouldWait: true,
};

export const Unknown: StoryObj = Template.bind({}) as any;
Unknown.parameters = {
  resolution: unknownKeys,
};

export const Known: StoryObj = Template.bind({}) as any;
Known.parameters = {
  resolution: knownKeys,
};

```

Now we can render all three different component states based on mocked return values.

## Per-story decorator

We can follow the example [here](https://storybook.js.org/docs/svelte/writing-stories/decorators#component-decorators) to move the above decorator into a single story. Edit `src-svelte/.storybook/preview.ts` to remove the `TauriInvokeDecorator`, then edit the story that needs it, `src-svelte/src/routes/ApiKeysDisplay.stories.ts`, to add the decorator:

```ts
import ApiKeysDisplay from "./ApiKeysDisplay.svelte";
import type { ApiKeys } from "$lib/bindings";
import type { StoryObj } from "@storybook/svelte";
import TauriInvokeDecorator from "$lib/__mocks__/invoke";

export default {
  component: ApiKeysDisplay,
  title: "Screens/Dashboard/API Keys Display",
  argTypes: {},
  decorators: [TauriInvokeDecorator],
};

...
```

## Mocking store values

Create `src-svelte/src/lib/__mocks__/stores.ts`:

```ts
import type { StoryFn, Decorator, StoryContext } from "@storybook/svelte";
import { unceasingAnimations } from "../../preferences";

interface Preferences {
  unceasingAnimations?: boolean;
}

interface StoreArgs {
  preferences?: Preferences;
  [key: string]: any;
}

const SvelteStoresDecorator: Decorator = (
  story: StoryFn,
  context: StoryContext,
) => {
  const { args, parameters } = context;
  const { preferences } = parameters as StoreArgs;
  if (preferences?.unceasingAnimations !== undefined) {
    unceasingAnimations.set(preferences.unceasingAnimations);
  }
  
  return story(args, context);
};

export default SvelteStoresDecorator;

```

Then use it like so in `src-svelte/src/routes/AppLayout.stories.ts`:

```ts
import AppLayout from "./AppLayout.svelte";
import type { StoryObj } from "@storybook/svelte";
import SvelteStoresDecorator from "$lib/__mocks__/stores";

export default {
  component: AppLayout,
  title: "Layout/App",
  argTypes: {},
  decorators: [SvelteStoresDecorator],
};

const Template = ({ ...args }) => ({
  Component: AppLayout,
  props: args,
});

export const Dynamic: StoryObj = Template.bind({}) as any;
Dynamic.parameters = {
  preferences: {
    unceasingAnimations: true,
  },
};

export const Static: StoryObj = Template.bind({}) as any;
Static.parameters = {
  preferences: {
    unceasingAnimations: false,
  },
};

```

## Custom background color for component

Follow the instructions [here](https://storybook.js.org/docs/react/writing-stories/parameters#component-parameters) and adapt to Svelte (the Svelte-specific docs don't mention the default option):

```ts
export default {
  component: SidebarUI,
  title: "Navigation/Sidebar",
  argTypes: {},
  parameters: {
    backgrounds: {
      default: 'ZAMM background',
      values: [
        { name: 'ZAMM background', value: '#f4f4f4' },
      ],
    },
  },
};
```

## Custom viewport for component

To make the component render at a specific size, we can use the `@storybook/addon-viewport` addon. Install it:

```bash
$ yarn add -D @storybook/addon-viewport
```

Then edit `src-svelte/.storybook/main.ts`:

```ts
...

const config: StorybookConfig = {
  ...
  addons: [
    ...
    getAbsolutePath("@storybook/addon-viewport"),
  ],
  ...
};
```

Then, as shown [here](https://stackoverflow.com/a/73028857) (note the typo in the answer), edit your story at, say, `src-svelte/src/routes/Metadata.stories.ts`,  to add the parameters:

```ts
...

export const Metadata: StoryObj = Template.bind({}) as any;
Metadata.parameters = {
  viewport: {
      defaultViewport: "mobile2"
  }
}
```

### Custom props for component

To create different stories that have different prop values by default, you can do this:

```ts
const Template = ({ ...args }) => ({
  Component: BackgroundComponent,
  props: args,
});

export const Static: StoryObj = Template.bind({}) as any;
Static.args = {
  animated: false,
};
export const Dynamic: StoryObj = Template.bind({}) as any;
Dynamic.args = {
  animated: true,
};
```

## Errors and warnings

### No story files

If you see this warning when starting Storybook up:

```
WARN No story files found for the specified pattern: src/**/*.mdx
```

remove `"../src/**/*.mdx"` from your `StorybookConfig` in `src-svelte/.storybook/main.ts`, from

```ts
...

const config: StorybookConfig = {
  stories: ["../src/**/*.mdx", "../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"],
  ...
};
export default config;
```

to

```ts
...

const config: StorybookConfig = {
  stories: ["../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"],
  ...
};
export default config;
```

### Blank page on Safari

If you see a blank page on Safari, even after reloading the page, you see no errors in the console, and it works fine on Firefox, it should work fine again on Safari after you quit and restart Safari.

There is [this issue](https://github.com/storybookjs/storybook/issues/23564) but it comes with a console error and does not seem to be applicable.

## Tips and tricks

### New browser window

To start Storybook without opening a new browser window (for example, if you already have it open and are merely restarting Storybook), run

```bash
$ yarn storybook --ci
```

Based on [this comment](https://github.com/storybookjs/storybook/issues/6201#issuecomment-812005603), we see that there is an alternative to disabling the auto-opening mechanism. However, it does not always work:

```
$ BROWSER=none yarn workspace gui storybook
...
/root/zamm/node_modules/@storybook/cli/bin/index.js:23
  throw error;
  ^

Error: spawn none ENOENT
    at __node_internal_captureLargerStackTrace (node:internal/errors:496:5)
    at __node_internal_errnoException (node:internal/errors:623:12)
    at ChildProcess._handle.onexit (node:internal/child_process:286:19)
    at onErrorNT (node:internal/child_process:484:16)
    at process.processTicksAndRejections (node:internal/process/task_queues:82:21)
Emitted 'error' event on ChildProcess instance at:
    at ChildProcess._handle.onexit (node:internal/child_process:292:12)
    at onErrorNT (node:internal/child_process:484:16)
    at process.processTicksAndRejections (node:internal/process/task_queues:82:21) {
  errno: -2,
  code: 'ENOENT',
  syscall: 'spawn none',
  path: 'none',
  spawnargs: [ 'http://localhost:6006/' ]
}

```

As such, we'll stick to `--ci`.

### Child process

Simply killing the spawned child in a NodeJS test might not work. Sa you log the child PID and it comes out as 193290:

```bash
$ ps aux | grep 193290      
root      193290  3.1  1.1 1262256 89044 pts/1   Sl+  03:32   0:00 /root/.asdf/installs/nodejs/20.5.1/bin/node /root/.asdf/installs/nodejs/20.5.1/bin/yarn storybook --ci
root      193740  0.0  0.0   6608  2276 pts/0    S+   03:32   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox 193290
```

But then you look at the actual process listening on port 6006, and it's a different one:

```bash
$ sudo lsof -t -i:6006
193341
$ ps aux | grep 193341      
root      193341  9.3  3.6 54489228 288556 pts/1 Sl   03:32   0:15 /root/.asdf/installs/nodejs/20.5.1/bin/node /root/zamm/src-svelte/node_modules/.bin/storybook dev -p 6006 --ci
root      194226  0.0  0.0   6608  2260 pts/0    S+   03:35   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox 193341

```

# Errors

## babel.js

If you get

```
10:37:55 AM [vite] error while updating dependencies:
Error: ENOENT: no such file or directory, open '/root/zamm/node_modules/prettier/plugins/babel.js'
```

then nuke all `node_module` directories in your project and restart Storybook.
