# Setting up Playwright component testing

## Using test components

Follow instructions [here](https://playwright.dev/docs/test-components):

```bash
$ yarn create playwright --ct
yarn create v1.22.19
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
success Installed "create-playwright@1.17.129" with binaries:
      - create-playwright
[#########] 9/9Getting started with writing end-to-end tests with Playwright:
Initializing project in '.'
✔ Which framework do you use? (experimental) · svelte
✔ Install Playwright browsers (can be done manually via 'yarn playwright install')? (Y/n) · true

✔ Install Playwright operating system dependencies (requires sudo / root - can be done manually via 'sudo yarn playwright install-deps')? (y/N) · false


Installing Playwright Component Testing (yarn add --dev @playwright/experimental-ct-svelte)…
yarn add v1.22.19
...
✔ Success! Created a Playwright Test project at /root/zamm/src-svelte

...

We suggest that you begin by typing:

  yarn run test-ct

Visit https://playwright.dev/docs/intro for more information. ✨

Happy hacking! 🎭
Done in 32.47s.
```

Then edit `src-svelte/playwright/index.ts`:

```ts
import '../src/routes/styles.css';
```

And `src-svelte/playwright-ct.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/experimental-ct-svelte';
import * as path from "path";

/**
 * See https://playwright.dev/docs/test-configuration.
 */
export default defineConfig({
  testDir: './',
  /* The base directory, relative to the config file, for snapshot files created with toMatchSnapshot and toHaveScreenshot. */
  snapshotDir: './snapshots',
  ...
  /* Shared settings for all the projects below. See https://playwright.dev/docs/api/class-testoptions. */
  use: {
    ...

    ctViteConfig: { 
      resolve: {
        alias: {
          $lib: path.resolve("src/lib"),
        },
      },
    },
  },
```

As it turns out, Playwright [cannot currently alias SvelteKit paths](https://github.com/microsoft/playwright/issues/19411). As such, we give up on this endeavor for now.

## Using Storybook

Follow the instructions at [`storybook.md`](./storybook.md). Then

```bash
$ yarn add -D @playwright/test playwright
$ yarn add -D jest-image-snapshot @types/jest-image-snapshot
```

and using the instructions [here](https://www.the-koi.com/projects/how-to-run-playwright-within-vitest/) and [here](https://greif-matthias.medium.com/how-to-setup-visual-testing-with-storybook-jest-and-puppeteer-c489b4f64c21), and the documentation for options [here](https://github.com/americanexpress/jest-image-snapshot), and the fact that Vitest is [already compatible](https://vitest.dev/guide/snapshot.html#image-snapshots) with `jest-image-snapshot`, we create a file `src-svelte/src/routes/storybook.test.ts`:

```ts
import { type Browser, chromium, expect, type Page } from "@playwright/test";
import { afterAll, beforeAll, describe, test } from "vitest";
import { toMatchImageSnapshot } from "jest-image-snapshot";

expect.extend({ toMatchImageSnapshot });

describe("Storybook visual tests", () => {
  let page: Page;
  let browser: Browser;

  const components = {
    "settings-api-keys-display": ["loading", "unknown", "known"],
  };

  beforeAll(async () => {
    browser = await chromium.launch({ headless: true });
    const context = await browser.newContext();
    page = await context.newPage();
  });

  afterAll(async () => {
    await browser.close();
  });

  for (const [component, variants] of Object.entries(components)) {
    describe(component, () => {
      for (const variant of variants) {
        const testName = variant ? variant : component;
        test(`${testName} should render the same`, async () => {
          const variantPrefix = variant ? `--${variant}` : "";

          await page.goto(
            `http://localhost:6006/?path=/story/${component}${variantPrefix}`,
          );

          const frame = page.frame({ name: "storybook-preview-iframe" });
          if (!frame) {
            throw new Error("Could not find Storybook iframe");
          }

          const rootElement = await frame.waitForSelector("#storybook-root");
          const screenshot = await rootElement.screenshot();

          // @ts-ignore
          expect(screenshot).toMatchImageSnapshot({
            diffDirection: "vertical",
            storeReceivedOnFailure: true,
            customSnapshotsDir: "screenshots/baseline",
            customSnapshotIdentifier: `${storybookPath}/${testName}`,
            customDiffDir: "screenshots/testing/diff",
            customReceivedDir: "screenshots/testing/actual",
            customReceivedPostfix: "",
          });
        });
      }
    });
  }
});
```

The `ts-ignore` is to avoid this error:

```
/root/zamm/src-svelte/src/routes/storybook.test.ts:103:30
Error: Property 'toMatchImageSnapshot' does not exist on type 'MakeMatchers<void, Buffer>'. Did you mean 'toMatchSnapshot'? 

          expect(screenshot).toMatchImageSnapshot({
            diffDirection: "vertical",
```

Unlike the example app [here](https://github.com/vitest-dev/vitest/blob/0c13c39/examples/image-snapshot/test/basic.test.ts), copying the module declaration here does not seem to get rid of the TypeScript error. The error is a spurious one because the test itself clearly executes successfully.

Make sure to add a `src-svelte/screenshots/.gitignore` to ignore the new output files. Because we've customized the output directories to be consistent with [WebdriverIO](/general-notes/setup/tauri/e2e-testing.md), we do

```
testing/
```

If you have instead kept the defaults, then yours should look like:

```
__diff_output__/
__received_output__/
```

Note that we'll have to edit the timeout in `src-svelte/src/lib/__mocks__/invoke.ts` after all for this test -- it appears Storybook behavior is subtly different on different platforms:

```
    if (nextShouldWait) {
      setTimeout(() => {
        resolve(nextResolution);
      }, 1_000_000); // the re-render never happens, so any timeout is fine
    } else {
      resolve(nextResolution);
    }
```

### Automatically starting Storybook before tests

Note that this requires `yarn storybook` to be running first before the tests run. We can automate this by installing

```bash
$ yarn add -D node-fetch
```

We observe that a regular Storybook startup looks like this:

```bash
$ yarn storybook
yarn run v1.22.19
$ storybook dev -p 6006
@storybook/cli v7.4.0

WARN The "@storybook/addon-mdx-gfm" addon is meant as a migration assistant for Storybook 7.0; and will likely be removed in a future version.
WARN It's recommended you read this document:
WARN https://storybook.js.org/docs/react/writing-docs/mdx#lack-of-github-flavored-markdown-gfm
WARN 
WARN Once you've made the necessary changes, you can remove the addon from your package.json and storybook config.
info => Serving static files from ././static at /
info => Starting manager..
WARN No story files found for the specified pattern: src/**/*.mdx
The following Vite config options will be overridden by SvelteKit:
  - base


╭─────────────────────────────────────────────────╮
│                                                 │
│   Storybook 7.4.0 for sveltekit started         │
│   267 ms for manager and 1.06 s for preview     │
│                                                 │
│    Local:            http://localhost:6006/     │
│    On your network:  http://5.78.91.93:6006/    │
│                                                 │
╰─────────────────────────────────────────────────╯
```

Next

```ts
...

import { spawn, ChildProcess } from "child_process";
import fetch from "node-fetch";

...

let storybookProcess: ChildProcess | null = null;

const startStorybook = (): Promise<void> => {
  return new Promise((resolve) => {
    storybookProcess = spawn("yarn", ["storybook", "--ci"]);
    if (!storybookProcess) {
      throw new Error("Could not start storybook process");
    } else if (!storybookProcess.stdout || !storybookProcess.stderr) {
      throw new Error("Could not get storybook output");
    }

    const storybookStartupMessage =
      /Storybook \d+\.\d+\.\d+ for sveltekit started/;

    storybookProcess.stdout.on("data", (data) => {
      const strippedData = data.toString().replace(/\\x1B\[\d+m/g, "");
      if (storybookStartupMessage.test(strippedData)) {
        resolve();
      }
    });

    storybookProcess.stderr.on("data", (data) => {
      console.error(`Storybook error: ${data}`);
    });
  });
};

const checkIfStorybookIsRunning = async (): Promise<boolean> => {
  try {
    await fetch("http://localhost:6006");
    return true;
  } catch {
    return false;
  }
};

describe("Storybook visual tests", () => {
  ...

  beforeAll(async () => {
    const isStorybookRunning = await checkIfStorybookIsRunning();
    if (!isStorybookRunning) {
      await startStorybook();
    }

    ...
  });

  afterAll(async () => {
    await browser.close();

    if (storybookProcess) {
      storybookProcess.kill();
    }
  });

  ...
}
```

The `--ci` option passed to Storybook prevents it from automatically popping a browser window open, which can be annoying when testing repeatedly during development.

Make sure you now update `.github/workflows/tests.yaml` as well. Screenshots should now be uploaded from our new directory:

```yaml
  svelte:
    ...
    steps:
      - name: Upload final app
        if: always() # run even if tests fail
        uses: actions/upload-artifact@v3
        with:
          name: storybook-screenshots
          path: |
            src-svelte/screenshots/testing/**/*.png
          retention-days: 1
```

#### Using Storybook across multiple tests

If we use Storybook across multiple tests, we may not want to stop it after it's started. This might be a potential cause of CI failures. Even locally, we can see

```
Error: browserContext.newPage: Target page, context or browser has been closed
 ❯ src/routes/storybook.test.ts:200:43
    198|   beforeEach<StorybookTestContext>(
    199|     async (context: TestContext & StorybookTestContext) => {
    200|       context.page = await browserContext.newPage();
       |                                           ^
    201|       context.expect.extend({ toMatchImageSnapshot });
    202|     },
```

What we want is some setup code that acts across test suites. We create `src-svelte/src/testSetup.ts` while looking at the documentation and example [here](https://vitest.dev/config/#globalsetup):

```ts
import { ensureStorybookRunning, killStorybook } from "$lib/test-helpers";

export default async function setup() {
  let storybookProcess = await ensureStorybookRunning();

  return () => killStorybook(storybookProcess);
}

```

We then refer to this in `src-svelte/vitest.config.ts`:

```ts
export default defineConfig({
  ...,
  test: {
    ...,
    globalSetup: "src/testSetup.ts",
  },
});

```

We then remove all references to Storybook process starting and killing from `src-svelte/src/lib/Slider.playwright.test.ts`, `src-svelte/src/lib/Switch.playwright.test.ts`, and `src-svelte/src/lib/Switch.playwright.test.ts`.

We get

```
Error: [birpc] timeout on calling "onTaskUpdate"
 ❯ Timeout._onTimeout ../node_modules/vitest/dist/vendor-index.b271ebe4.js:39:22
 ❯ listOnTimeout node:internal/timers:573:17
 ❯ processTimers node:internal/timers:514:7

This error originated in "src/lib/Switch.test.ts" test file. It doesn't mean the error was thrown inside the file itself, but while it was running.
The latest test that might've caused the error is "calls onToggle when toggled". It might mean one of the following:
- The error was thrown, while Vitest was running this test.
- This was the last recorded test before the error was thrown, if error originated after test finished its execution.
```

This appears to be a fluke that doesn't happen again, or else we would've tried [this workaround](https://github.com/vitest-dev/vitest/issues/1154#issuecomment-1138717832). After several more tries, we find that we're still getting the old error. We try to see if running the tests sequentially rather than in parallel would help. We find that there's `poolOptions`, but these don't appear to be available for the old version of Vitest we're using. We try to upgrade `vitest` to 1.2.2, but that also requires an upgrade for `vite` to 5.1.2. We edit `src-svelte/package.json`:

```json
{
  ...,
  "devDependencies": {
    ...
    "vite": "^5.1.3",
    "vitest": "^1.2.2"
  },
  ...
}
```

We remove the corresponding `resolutions` override for `vite` from `package.json` that we added in [`chat.md`](/zamm-notes/chat.md). We do a final

```bash
$ yarn install
```

Now we can finally edit `src-svelte/vitest.config.ts`:

```ts

export default defineConfig({
  ...,
  test: {
    ...,
    poolOptions: {
      threads: {
        singleThread: true,
      },
    },
  },
});

```

This seems to work locally, but when we commit, Svelte type-checking gives us:

```
/root/zamm/src-svelte/src/routes/storybook.test.ts:202:19
Error: Property 'task' does not exist on type 'TestContext & StorybookTestContext'.
    async (context: TestContext & StorybookTestContext) => {
      if (context.task.result?.state === "pass") {
        await context.page.close();
```

It actually just works in `src-svelte/src/routes/storybook.test.ts` if we avoid specifying the types ourselves:

```ts
  beforeEach<StorybookTestContext>(
    async (context) => {
      context.page = await browserContext.newPage();
      context.expect.extend({ toMatchImageSnapshot });
    },
  );

  afterEach<StorybookTestContext>(
    async (context) => {
      if (context.task.result?.state === "pass") {
        await context.page.close();
      }
    },
  );
```

We get the error

```
/root/zamm/src-svelte/src/routes/storybook.test.ts:240:26
Error: Property 'page' does not exist on type 'TaskContext<Test<{}>> & TestContext'. 
        `${testName} should render the same`,
        async ({ expect, page }) => {
          const variantPrefix = `--${variantConfig.name}`;
```

further down, so we do the same thing again:

```ts
      test<StorybookTestContext>(
        `${testName} should render the same`,
        async ({ expect, page }) => {
          ...
        }
        ...
      );
```

Of course, we'll need to update our Docker image base too, because otherwise we get the error:

```
yarn install --frozen-lockfile && yarn svelte-kit sync && yarn build
yarn install v1.22.21
[1/4] Resolving packages...
[2/4] Fetching packages...
warning svelte-preprocess@5.1.3: The engine "pnpm" appears to be invalid.
error vite@5.1.3: The engine "node" is incompatible with this module. Expected version "^18.0.0 || >=20.0.0". Got "16.20.2"
```

We could just build the frontend in a newer version of Ubuntu, and then build the backend (which will get linked against GLIBC) in an older version, but that is not worth the effort at this point. We upgrade the `Dockerfile` to use a later base image of Ubuntu 20.04 so that we can use a newer version of NodeJS:

```Dockerfile
FROM ubuntu:20.04
...

ARG TAURI_CLI_VERSION=1.5.9
...
RUN cargo install --locked tauri-cli@${TAURI_CLI_VERSION}

ARG NODEJS_VERSION=20.5.1
```

Note that we're also pinning `TAURI_CLI_VERSION` because the new Dockerfile fails to build with the latest Tauri CLI version that depends on a newer version of Rust.

We edit the `Makefile` to build a new Docker image for this version of the software:

```Makefile
...

BUILD_IMAGE = ghcr.io/amosjyng/zamm:v0.1.1-build
...
```

and we similarly edit the file `.github/workflows/tests.yaml`:

```yaml
jobs:
  build:
    name: Build entire program
    ...
    container:
      image: "ghcr.io/amosjyng/zamm:v0.1.0-build"
```

### Diff direction

In some cases, we want the diff to be vertical when the screenshot is wide, and we want the diff to be horizontal when the screenshot is tall. We install


```bash
$ yarn add -D image-size
```

and then do

```ts
import sizeOf from "image-size";

describe("Storybook visual tests", () => {
  ...
          const screenshotSize = sizeOf(screenshot);
          const diffDirection = (screenshotSize.width && screenshotSize.height && screenshotSize.width > screenshotSize.height) ? "vertical" : "horizontal";

          // @ts-ignore
          expect(screenshot).toMatchImageSnapshot({
            diffDirection,
            ...
          });
  ...
});
```

### Testing dynamic images

Let's say we want to also optionally check that static elements are static, whereas dynamic elements (e.g. animated ones) change. We wouldn't want to do exact screenshot tests because depending on the exact timing of the screenshot, the animation may be in a slightly different place each time. Instead, we take two screenshots and check that they're different. But first, we want to only *optionally* specify this for each test. We do this by defining an optional additional config for each test variant:

```ts
interface ComponentTestConfig {
  path: string[]; // Represents the Storybook hierarchy path
  variants: string[] | VariantConfig[];
}

interface VariantConfig {
  name: string;
  assertDynamic?: boolean;
}
```

Now we change the test components to:

```ts
const components: ComponentTestConfig[] = [
  {
    path: ["background"],
    variants: [{
      name: "static",
      assertDynamic: false,
    }, {
      name: "dynamic",
      assertDynamic: true,
    }],
  },
  ...
]
```

We refactor screenshot-taking to be able to reuse it:

```ts
  const takeScreenshot = (page: Page) => {
    const frame = page.frame({ name: "storybook-preview-iframe" });
          if (!frame) {
            throw new Error("Could not find Storybook iframe");
          }
    return frame
    .locator("#storybook-root > :first-child")
    .screenshot();
  }
```

Then we update our tests as such:

```ts
  for (const config of components) {
    const storybookUrl = config.path.join("-");
    const storybookPath = config.path.join("/");
    describe(storybookPath, () => {
      for (const variant of config.variants) {
        const variantConfig = typeof variant === "string" ? {
          name: variant,
        } : variant;
        const testName = variantConfig.name;
        test(`${testName} should render the same`, async () => {
          const variantPrefix = `--${variantConfig.name}`;

          await page.goto(
            `http://localhost:6006/?path=/story/${storybookUrl}${variantPrefix}`,
          );

          const screenshot = await takeScreenshot(page);

          const screenshotSize = sizeOf(screenshot);
          const diffDirection =
            screenshotSize.width &&
            screenshotSize.height &&
            screenshotSize.width > screenshotSize.height
              ? "vertical"
              : "horizontal";

          if (!variantConfig.assertDynamic) {
            // don't compare dynamic screenshots against baseline
            // @ts-ignore
            expect(screenshot).toMatchImageSnapshot({
              diffDirection,
              storeReceivedOnFailure: true,
              customSnapshotsDir: "screenshots/baseline",
              customSnapshotIdentifier: `${storybookPath}/${testName}`,
              customDiffDir: "screenshots/testing/diff",
              customReceivedDir: "screenshots/testing/actual",
              customReceivedPostfix: "",
            });
          }

          if (variantConfig.assertDynamic !== undefined) {
            await new Promise(r => setTimeout(r, 1000));
            const newScreenshot = await takeScreenshot(page);

            if (variantConfig.assertDynamic) {
              expect(newScreenshot).not.toEqual(screenshot);
            } else {
              // do the same assertion from before so that we can see what changed the
              // second time around if a static screenshot turns out to be dynamic
              //
              // @ts-ignore
              expect(newScreenshot).toMatchImageSnapshot({
                diffDirection,
                storeReceivedOnFailure: true,
                customSnapshotsDir: "screenshots/baseline",
                customSnapshotIdentifier: `${storybookPath}/${testName}`,
                customDiffDir: "screenshots/testing/diff",
                customReceivedDir: "screenshots/testing/actual",
                customReceivedPostfix: "",
              });
            }
          }
        });
      }
    });
  }
```

We get an error:

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests > background > dynamic should render the same
TypeError: this.customTesters is not iterable
 ❯ Proxy.<anonymous> ../node_modules/playwright/lib/matchers/expect.js:166:37
 ❯ src/routes/storybook.test.ts:159:41
    157| 
    158|             if (variantConfig.assertDynamic) {
    159|               expect(newScreenshot).not.toEqual(screenshot);
       |                                         ^
    160|             } else {
    161|               // do the same assertion from before so that we can see …
```

It appears we can't use our regular testers here. Even doing `expect(Buffer.compare(screenshot, newScreenshot)).not.toEqual(0);` doesn't help. We confirm that we literally can't use any of the usual matchers:

```ts
expect("one").not.toEqual("two");
```

fails as well.

We finally see that [others](https://github.com/microsoft/playwright/issues/20432) have run into this issue too. Our usage of `expect.extend({ toMatchImageSnapshot });` turns out to be a red herring, as the problem is that Playwright replaces the default `expect` with its own. We'll have to check from the list of assertions [here](https://playwright.dev/docs/test-assertions). This is the solution:


```ts
expect(Buffer.compare(screenshot, newScreenshot) !== 0).toBeTruthy();
```

Of course, we should also test that our tests are actually discriminatory between passing and failing states. If we disable the animation, does the test for dynamism now fail? It turns out that it does, so we are good here.

#### Refactoring duplicate config

Note that if we are to now add a new setting such as `allowSizeMismatch`, we now need to add it to two separate configs because it got duplicated now. To avoid this, refactor out the common parts:

```ts
import { toMatchImageSnapshot, type MatchImageSnapshotOptions } from "jest-image-snapshot";

const baseMatchOptions: MatchImageSnapshotOptions = {
    allowSizeMismatch: true,
    storeReceivedOnFailure: true,
    customSnapshotsDir: "screenshots/baseline",
    customDiffDir: "screenshots/testing/diff",
    customReceivedDir: "screenshots/testing/actual",
    customReceivedPostfix: "",
  };

  for (const config of components) {
    ...
            const matchOptions = {
              ...baseMatchOptions,
              diffDirection,
              customSnapshotIdentifier: `${storybookPath}/${testName}`,
            };

            if (!variantConfig.assertDynamic) {
              // don't compare dynamic screenshots against baseline
              // @ts-ignore
              expect(screenshot).toMatchImageSnapshot(matchOptions);
            }

            if (variantConfig.assertDynamic !== undefined) {
              ...
                expect(newScreenshot).toMatchImageSnapshot(matchOptions);
              ...
            }
          },
          ...
```

### Testing entire body

Sometimes, if we change the shadow of an element, the shadow is not part of the element's bounding box and therefore won't be captured in the screenshot. For these cases, we may want to zoom out to take a screenshot of the entire body instead of just the element itself. We can add an option:

```ts
interface ComponentTestConfig {
  ...
  screenshotEntireBody?: boolean;
}

const components: ComponentTestConfig[] = [
  ...
  {
    path: ["dashboard", "api-keys-display"],
    variants: ["loading", "unknown", "known"],
    screenshotEntireBody: true,
  },
  {
    path: ["dashboard", "metadata"],
    variants: ["metadata"],
    screenshotEntireBody: true,
  },
  ...
];

...

  const takeScreenshot = (page: Page, screenshotEntireBody?: boolean) => {
    const frame = page.frame({ name: "storybook-preview-iframe" });
    if (!frame) {
      throw new Error("Could not find Storybook iframe");
    }
    const locator = screenshotEntireBody ? "body" : "#storybook-root > :first-child";
    return frame.locator(locator).screenshot();
  };

...

          const screenshot = await takeScreenshot(page, config.screenshotEntireBody);

...
```

You may find some of these full-body tests to be flaky. In that case, specify a certain number of retries as explained [here](/general-notes/setup/tauri/vitest.md).

## Parallelization

You can follow the instructions [here](https://vitest.dev/guide/test-context.html) to make tests concurrent. Note that this only affects tests within a single suite; separate suites of tests will still execute sequentially.

For example, to refactor `src-svelte/src/routes/storybook.test.ts` to run tests in parallel:

```ts
import {
  type Browser,
  chromium,
  type Page,
  type BrowserContext,
} from "@playwright/test";
import { afterAll, beforeAll, afterEach, beforeEach, describe, test, type TestContext } from "vitest";
...

interface StorybookTestContext {
  page: Page;
}

describe.concurrent("Storybook visual tests", () => {
  let storybookProcess: ChildProcess | undefined;
  let browser: Browser;
  let browserContext: BrowserContext;

  beforeAll(async () => {
    browser = await chromium.launch({ headless: true });
    browserContext = await browser.newContext();
    storybookProcess = await ensureStorybookRunning();
  });

  afterAll(async () => {
    await browserContext.close();
    await browser.close();
    await killStorybook(storybookProcess);
  });

  beforeEach<StorybookTestContext>(
    async (context: TestContext & StorybookTestContext) => {
      context.page = await browserContext.newPage();
      context.expect.extend({ toMatchImageSnapshot });
    },
  );

  afterEach<StorybookTestContext>(
    async (context: TestContext & StorybookTestContext) => {
      await context.page.close();
    },
  );

  ...

  for (const config of components) {
    const storybookUrl = config.path.join("-");
    const storybookPath = config.path.join("/");
    for (const variant of config.variants) {
      const variantConfig =
        typeof variant === "string"
          ? {
              name: variant,
            }
          : variant;
      const testName = variantConfig.name;
      test(
        `${testName} should render the same`,
        async ({ expect, page }: TestContext & StorybookTestContext) => {
          const variantPrefix = `--${variantConfig.name}`;

          await page.goto(
            `http://localhost:6006/?path=/story/${storybookUrl}${variantPrefix}`,
          );

          ...
        },
        {
          retry: 4,
          timeout: 10_000,
        },
      );
    }
  }
});

```

Note that the nested `describe` has been stripped to allow full concurrency across all tests in this file. Because we have removed that context from which test is being run, we should add the information back into each test:

```ts
      const testName = `${storybookPath}/${variantConfig.name}.png`;
```

However, this means that we should also change `matchOptions` to use `variantConfig.name` instead of `testName`, or else none of the existing screenshot files will match because new ones will be created instead at paths like `src-svelte/screenshots/baseline/screens/dashboard/api-keys-display/screens/dashboard/api-keys-display/...`:

```ts
          const matchOptions = {
            ...baseMatchOptions,
            diffDirection,
            customSnapshotIdentifier: `${storybookPath}/${variantConfig.name}`,
          };
```

## Errors

### Test timeout

If you get an error such as

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests > navigation/sidebar > settings-selected should render the same
Error: Test timed out in 5000ms.
If this is a long-running test, pass a timeout value as the last argument or configure it globally with "testTimeout".
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[4/4]⎯
```

then you can increase the timeout like so:

```ts
...

        test(`${testName} should render the same`, async () => {
          ...
        }, 40_000);

...
```

If you're specifying other options such as `retry`, then add `timeout` like so:

```ts
...

        test(`${testName} should render the same`, async () => {
          ...
        }, {
          retry: 4,
          timeout: 40_000,
        });

...
```

### Element visibility timeout

If instead you get a test like this:

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests > navigation/sidebar > settings-selected should render the same
TimeoutError: locator.screenshot: Timeout 29859.952000000005ms exceeded.
=========================== logs ===========================
taking element screenshot
  waiting for element to be visible and stable
    element is not visible - waiting...
============================================================
```

and you are in the specific context of trying to take a Storybook screenshot, this may be because the `#storybook-root` element itself has zero height despite having child elements that are visible -- for example, if its only child is a header element. In this case, you can try to take a screenshot of the child element instead.

```ts
          const screenshot = await frame
            .locator("#storybook-root > :first-child")
            .screenshot();
```

This has the added benefit of making the screenshot more compact, since it will only be the size of the child element and not the entire Storybook root element.

Note that we do this again in [settings.md](/zamm-notes/settings.md), making it

```ts
  const takeScreenshot = async (page: Page, screenshotEntireBody?: boolean) => {
    const frame = page.frame({ name: "storybook-preview-iframe" });
    if (!frame) {
      throw new Error("Could not find Storybook iframe");
    }
    let locator = screenshotEntireBody
      ? "body"
      : "#storybook-root > :first-child";
    const elementClass = await frame.locator(locator).getAttribute("class");
    if (elementClass === "storybook-wrapper") {
      locator = "#storybook-root > :first-child > :first-child";
    }
    return await frame.locator(locator).screenshot();
  };
```

### Faster timeouts

To timeout faster so that your tests can run faster, see [this answer](https://stackoverflow.com/a/68107808).

### New browser download needed

If you get

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests
Error: browserType.launch: Executable doesn't exist at /root/.cache/ms-playwright/chromium-1080/chrome-linux/chrome
╔═════════════════════════════════════════════════════════════════════════╗
║ Looks like Playwright Test or Playwright was just installed or updated. ║
║ Please run the following command to download new browsers:              ║
║                                                                         ║
║     yarn playwright install                                             ║
║                                                                         ║
║ <3 Playwright Team                                                      ║
╚═════════════════════════════════════════════════════════════════════════╝
```

then do as it says and run

```bash
$ yarn playwright install     
yarn run v1.22.19
$ /root/zamm/node_modules/.bin/playwright install
Removing unused browser at /root/.cache/ms-playwright/chromium-1076
Removing unused browser at /root/.cache/ms-playwright/firefox-1422
Removing unused browser at /root/.cache/ms-playwright/webkit-1883
Downloading Chromium 117.0.5938.62 (playwright build v1080) from https://playwright.azureedge.net/builds/chromium/1080/chromium-linux.zip
153.1 Mb [====================] 100% 0.0s
Chromium 117.0.5938.62 (playwright build v1080) downloaded to /root/.cache/ms-playwright/chromium-1080
Downloading Firefox 117.0 (playwright build v1424) from https://playwright.azureedge.net/builds/firefox/1424/firefox-ubuntu-22.04.zip
78.8 Mb [====================] 100% 0.0s
Firefox 117.0 (playwright build v1424) downloaded to /root/.cache/ms-playwright/firefox-1424
Downloading Webkit 17.0 (playwright build v1908) from https://playwright.azureedge.net/builds/webkit/1908/webkit-ubuntu-22.04.zip
82.5 Mb [====================] 100% 0.0s
Webkit 17.0 (playwright build v1908) downloaded to /root/.cache/ms-playwright/webkit-1908
Done in 10.96s.
```

**Note that when a new browser is used,** some effects such as SVG filters may be rendered ever so slightly differently. This may cause your screenshot tests to fail. Visually inspect the changes, and if they are minor as expected, then update the baseline screenshots. Alternatively, as mentioned [here](https://news.ycombinator.com/item?id=32908506), update screenshots for a build that is known to be good. You may want to combine this with visual inspection if the differences are large enough, in case the old build relied on some quirks of the old browser version to render correctly.

### Vitest error unhandled

If you get an unhandled error:

```
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Errors ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

Vitest caught 1 unhandled error during the test run.
This might cause false positive tests. Resolve unhandled errors to make sure your tests are not affected.

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Unhandled Rejection ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯
Error: expect(received).toHaveAttribute(expected)

Expected string: "true"
Received string: "false"
Call log:
  - locator._expect with timeout 5000ms
  - waiting for getByRole('switch')
  -   locator resolved to <button tabindex="0" type="button" role="switch" id="swi…>…</button>
  -   unexpected value "false"
```

that's because of [this problem](https://github.com/vitest-dev/vitest/discussions/3229#discussioncomment-5685717). Your assert is likely making a promise. Await on it, for example:

```ts
await expect(onOffSwitch).toHaveAttribute("aria-checked", "false");
```

Then you get a proper error:

```
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/lib/Switch.playwright.test.ts > Switch drag test > plays clicking sound when drag released
Error: Test timed out in 5000ms.
If this is a long-running test, pass a timeout value as the last argument or configure it globally with "testTimeout".
```

## Moving screenshot locations

Say you want to move dashboard screenshots to be under a more general screens directory. Then:

Rename entire folders like `src-svelte/screenshots/baseline/dashboard/metadata/` to `src-svelte/screenshots/baseline/screens/dashboard/metadata/`.

Edit `src-svelte/src/routes/ApiKeysDisplay.stories.ts` from

```ts
export default {
  ...
  title: "Dashboard/API Keys Display",
  ...
};
```

to

```ts
export default {
  ...
  title: "Screens/Dashboard/API Keys Display",
  ...
};
```

Do the same thing for the stories at `src-svelte/src/routes/Metadata.stories.ts`.

Then, edit the tests at `src-svelte/src/routes/storybook.test.ts` from

```ts
const components: ComponentTestConfig[] = [
  ...
    {
    path: ["dashboard", "api-keys-display"],
    variants: ["loading", "unknown", "known"],
    screenshotEntireBody: true,
  },
  {
    path: ["dashboard", "metadata"],
    variants: ["metadata"],
    screenshotEntireBody: true,
  },
  ...
```

to

```ts
const components: ComponentTestConfig[] = [
  ...
    {
    path: ["screens", "dashboard", "api-keys-display"],
    variants: ["loading", "unknown", "known"],
    screenshotEntireBody: true,
  },
  {
    path: ["screens", "dashboard", "metadata"],
    variants: ["metadata"],
    screenshotEntireBody: true,
  },
  ...
```

Rearrange the entries, for example in alphabetical order, so that the screen stories still appear next to each other.

## Approximate screenshots

To allow for approximate screenshots to pass, you can use

```ts
                expect(newScreenshot).toMatchImageSnapshot({
                  ...
                  allowSizeMismatch: true,
                  ...
                });
```

## Debugging screenshots

### Opening the Playwright browser

To see how a Playwright browser (e.g. webkit) is rendering the test, you can open the page in the Playwright browser with

```bash
$ yarn playwright open -b webkit localhost:6006
```

Note that sometimes, it turns out headed/headful mode renders things differently from headless mode.

### Avoiding closing the browser at test time

We can avoid closing the test browser if the relevant test failed:

```ts
  afterEach<StorybookTestContext>(
    async (context: TestContext & StorybookTestContext) => {
      if (context.task.result?.state === "pass") {
        await context.page.close();
      }
    },
  );
```

## Storybook tweaks

If we don't clamp down on non-determinism, we may need to run our tests as many as 5 times before they pass. This wastes a lot of time before we're relatively certain that something is truly broken and not just a fluke of the latest rendering mishap.

After all is said and done in this section, we'll have gone from 5 total tries to just 1 single one.

### Dimensional differences

Sometimes, there are slight pixel-wide size differences between the different screenshots taken. We allow for multiple "variant" screenshots which, if any of them are matched, count as a pass for our tests:

```ts
...
import * as fs from "fs/promises";
...

async function findVariantFiles(directoryPath: string, filePrefix: string): Promise<string[]> {
  const files = await fs.readdir(directoryPath);
  return files.filter(file => {
    return file.startsWith(filePrefix) && file.match(/-variant-\d+\.png$/) !== null;
}).map(file => `${directoryPath}/${file}`);
}

async function checkVariants(variantFiles: string[], screenshot: Buffer) {
  for (const file of variantFiles) {
    const fileBuffer = await fs.readFile(file);
    if (Buffer.compare(fileBuffer, screenshot) === 0) {
      return true;
    }
  }
  return false;
}

...

describe.concurrent("Storybook visual tests", () => {
  ...
  for (const config of components) {
    ...
          // don't compare dynamic screenshots against baseline
          if (!variantConfig.assertDynamic) {
            // look for all files in filesystem with suffix -variant-x.png
            const variantFiles = await findVariantFiles(`screenshots/baseline/${storybookPath}`, variantConfig.name);
            const variantsMatch = await checkVariants(variantFiles, screenshot);
            if (!variantsMatch) {
               // @ts-ignore
              expect(screenshot).toMatchImageSnapshot(matchOptions);
            }
          }
    ...
  }
});
```

So for example, if we normally compare against `some/example/foo.png`, we can also set a `some/example/foo-variant-1.png` to be a valid match.

Note that we'll want to make sure that our tests can still fail so that they don't give us false confidence. For example, if we instead do

```ts
const variantsMatch = checkVariants(variantFiles, screenshot);
```

then the tests will always trivially pass because `checkVariants` returns a Promise, which will always be truthy.

### Clearing out the screenshot comparisons directory

In the above example, we avoid calling `toMatchImageSnapshot` if there are any variant matches because a mismatch will cause the whole test to fail. However, not calling that function at all means that nothing will remove screenshots that have previously differed but now match up fine. As such, we remove it manually ourselves:

```ts
describe.concurrent("Storybook visual tests", () => {
  ...

  beforeAll(async () => {
    ...

    try {
      await fs.rmdir("screenshots/testing", { recursive: true });
    } catch (e) {
      // ignore, it's okay if the folder already doesn't exist
    }
  });

  ...
});
```

Since we get the warning

```
(node:63345) [DEP0147] DeprecationWarning: In future versions of Node.js, fs.rmdir(path, { recursive: true }) will be removed. Use fs.rm(path, { recursive: true }) instead
(Use `node --trace-deprecation ...` to show where the warning was created)
```

we follow the suggestion and instead do

```ts
await fs.rm("screenshots/testing", { recursive: true, force: true });
```

Note that we can't just remove the `recursive: true` part because the empty folders inside will still exist afterwards.

### Waiting for fonts to load

Non-deterministic tests can be caused by the screenshot being taken before the font has finished loading. We see from [this discussion](https://github.com/microsoft/playwright/issues/18640) that we can do

```ts
await page.evaluate(() => document.fonts.ready);
```

but that this doesn't make a difference in Chromium. We'll follow the advice in that issue and use Webkit instead. We will need to check every single one of our screenshots afterwards, as they will all differ noticeably from Chromium:

```ts
import {
  ...,
  webkit,
  ...
} from "@playwright/test";

...

describe.concurrent("Storybook visual tests", () => {
  ...

  beforeAll(async () => {
    browser = await webkit.launch(...);
    ...
  });

  ...
});
```

We eventually change this to

```ts
          const frame = page.frame({ name: "storybook-preview-iframe" });
          if (!frame) {
            throw new Error("Could not find Storybook iframe");
          }
          await page.locator("button[title='Hide addons [A]']").click();

          // wait for fonts to load
          await frame.evaluate(() => document.fonts.ready);

          const screenshot = await takeScreenshot(
            frame,
            ...
          );

          ...

          if (variantConfig.assertDynamic !== undefined) {
            ...
            const newScreenshot = await takeScreenshot(
              frame,
              ...
            );
```

while refactoring `takeScreenshot` to take in a frame

```ts
  const takeScreenshot = async (frame: Frame, screenshotEntireBody?: boolean) => {
    ...
  };
```

This is in case the issue is that we were waiting for fonts on the main page to load rather than fonts inside the frame. Unfortunately, it appears that this actually hurts the chances of a frontend run completing successfully due to the storybook div not being found at this point in time, so we revert this change.

### Running consistently in headed mode

We see that we have one test that consistently fails now. The snackbar messages somehow renders with much more vertical whitespace than expected. We open up the Webkit browser to debug as noted above, just to find that it appears to render just fine in the browser, but not so much in the test. Upon further testing, we find that headless Webkit appears to somehow render things differently than headful Webkit in this particular case. We solve this by always running these tests in headed mode:

```ts
describe.concurrent("Storybook visual tests", () => {
  ...

  beforeAll(async () => {
    browser = await webkit.launch({ headless: false });
    ...
  }, 30_000); // webkit takes a little while to start up on headed mode

  ...
});
```

If we're running this in CI, then we should also make use of `xvfb` by editing our CI file (say, `.github/workflows/tests.yaml`) to use the Docker image specified [here](https://playwright.dev/docs/docker) (which, as noted [here](https://playwright.dev/docs/ci#running-headed), already contains `xvfb`):

```yaml
  svelte:
    ...
    runs-on: ubuntu-latest
    container:
      image: "mcr.microsoft.com/playwright:v1.40.0-jammy"
    env:
      PLAYWRIGHT_TIMEOUT: 60000
    ...
    steps:
      - name: Run Svelte Tests
        run: |
          xvfb-run yarn workspace gui test
```

We edit `src-svelte/src/routes/storybook.test.ts` again to dynamically change the overall timeout based on the Playwright timeout:

```ts
        {
          retry: 0,
          timeout: DEFAULT_TIMEOUT * 2.2,
        },
```

Our tests fail on CI due to screenshot mismatches. Upon further investigation, we realize that the likely cause behind the failulres on Firefox are due to using a different version of Playwright for the Docker image than is specified in `package.json`. We re-align the two versions in `.github/workflows/tests.yaml`:

```yaml
  svelte:
    ...
    container:
      image: "mcr.microsoft.com/playwright:v1.38.0-jammy"
```

This still fails. Even when we put in the line

```ts
  beforeAll(async () => {
    browser = await webkit.launch({ headless: false });
    console.log(`Running tests with Webkit version ${browser.version()}`);
    ...
  });
```

we see that Webkit version 17.0 is being used both locally and remotely.

Unable to reconcile these differences, we add support for alternative local gold files in `src-svelte/src/routes/storybook.test.ts`:

```ts
...

const SCREENSHOTS_BASE_DIR =
  process.env.SCREENSHOTS_BASE_DIR === undefined
    ? "screenshots"
    : process.env.SCREENSHOTS_BASE_DIR;

...

describe.concurrent("Storybook visual tests", () => {
  ...

  beforeAll(async () => {
    ...

    try {
      await fs.rm(`${SCREENSHOTS_BASE_DIR}/testing`, { recursive: true, force: true });
    } ...
  }, ...);

  const baseMatchOptions: MatchImageSnapshotOptions = {
    ...,
    customSnapshotsDir: `${SCREENSHOTS_BASE_DIR}/baseline`,
    customDiffDir: `${SCREENSHOTS_BASE_DIR}/testing/diff`,
    customReceivedDir: `${SCREENSHOTS_BASE_DIR}/testing/actual`,
    ...
  };

  ...

          if (!variantConfig.assertDynamic) {
            // look for all files in filesystem with suffix -variant-x.png
            const variantFiles = await findVariantFiles(
              `${SCREENSHOTS_BASE_DIR}/baseline/${storybookPath}`,
              variantConfig.name,
            );
            ...
          }
  ...
```

We edit `src-svelte/Makefile` as well to make it convenient to support this new method:

```Makefile
local-screenshots:
	mkdir -p screenshots/local
	cp -r screenshots/baseline screenshots/local/baseline

update-local-screenshots:
	cp -r screenshots/local/testing/actual/* screenshots/local/baseline/
```

and edit `src-svelte/screenshots/.gitignore` to ignore that new folder:

```gitignore
local/
```

At this point, we don't need to run it in headed mode anymore, and therefore don't need the extra timeout for the `beforeAll` hook:

```ts
  beforeAll(async () => {
    browser = await webkit.launch({ headless: true });
    ...
  });
```

### Using Firefox instead

To try to fix the differences between local and remote test runs, we change `webkit` to `firefox`, and also:

- set `headless` back to true
- remove the 30s timeout for the `beforeAll` hook

because both of these were workarounds specifically for Webkit, and are no longer needed if we are not using Webkit.

Unfortunately, with Firefox, we sometimes get this error:

```
Error: locator.click: Target crashed
=========================== logs ===========================
waiting for locator('button[title=\'Hide addons [A]\']')
  locator resolved to <button type="button" class="css-17dxjer" title="Hide ad…>…</button>
attempting click action
  waiting for element to be visible, enabled and stable
  element is visible, enabled and stable
  scrolling into view if needed
============================================================
```

As such, we set the retry back to 1:

```ts
        {
          retry: 1,
          timeout: DEFAULT_TIMEOUT * 2.2,
        },
```

On CI this still fails with

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests
Error: browserType.launch: Browser.enable): Browser closed.
==================== Browser output: ====================
<launching> /ms-playwright/firefox-1424/firefox/firefox -no-remote -headless -profile /tmp/playwright_firefoxdev_profile-vCYJc9 -juggler-pipe -silent
<launched> pid=1035
[pid=1035][err] Running Nightly as root in a regular user's session is not supported.  ($HOME is /github/home which is owned by uid 1001.)
[pid=1035] <process did exit: exitCode=1, signal=null>
[pid=1035] starting temporary directories cleanup
=========================== logs ===========================
<launching> /ms-playwright/firefox-1424/firefox/firefox -no-remote -headless -profile /tmp/playwright_firefoxdev_profile-vCYJc9 -juggler-pipe -silent
<launched> pid=1035
[pid=1035][err] Running Nightly as root in a regular user's session is not supported.  ($HOME is /github/home which is owned by uid 1001.)
[pid=1035] <process did exit: exitCode=1, signal=null>
[pid=1035] starting temporary directories cleanup
============================================================
```

We see that the [solution](https://github.com/microsoft/playwright/issues/6500) is to set `HOME` to `/root`, so we do that in `.github/workflows/tests.yaml`:

```yaml
  svelte:
    ...
    container:
      image: "mcr.microsoft.com/playwright:v1.38.0-jammy"
    env:
      HOME: /root
      ...
```

Now the tests run, and still produce a failure remotely. Because there's no advantage to using Firefox, and because there's the disadvantage of needing to run tests twice due to the flaky target error, we switch back to Webkit.

### Using SSIM

We can do as the Storybook repo maintainers recommend and use SSIM with a failure threshold of 1%:

```ts
  const baseMatchOptions: MatchImageSnapshotOptions = {
    ...,
    comparisonMethod: 'ssim',
    failureThreshold: 0.01,
    failureThresholdType: 'percent',
    ...
  };
```

Unfortunately, this strategy results in missing changes to an entire letter, so we undo our changes here at first. However, we later find that with the chat conversation screenshots, there are slight font kerning differences on different runs. We add it back in with a threshold of 0.5% to fight test flakiness, because at this threshold failures to resize chat bubbles are still successfully detected. This does mean that individual character swaps are no longer detected, but larger scale differences should still show up.

Because the image diffs are a lot less useful now, we use the solutions [here](https://stackoverflow.com/q/5132749) to highlight pixel-level differences. For example:

```bash
$ compare actual.png expected.png diff-output.png
```

To automate this, we create `src-svelte/diff-screenshots.py`:

```py
#!/usr/bin/env python3
import os
import subprocess
import sys

# Directory paths
relative_base_dir = os.environ.get("SCREENSHOTS_BASE_DIR", "screenshots/")
base_dir = os.path.abspath(relative_base_dir)
baseline_dir = f"{base_dir}/baseline/"
test_dir = f"{base_dir}/testing/actual/"
diff_dir = f"{base_dir}/testing/diff/"

# Check if test_dir exists
if not os.path.isdir(test_dir):
    print("No faulty screenshots at", test_dir)
    sys.exit()

# Traverse directory
for dirpath, dirnames, filenames in os.walk(test_dir):
    # Check each file
    for filename in filenames:
        # Create corresponding baseline and diff file paths
        base_file = os.path.join(
            baseline_dir, os.path.relpath(dirpath, test_dir), filename
        )
        diff_file = os.path.join(
            diff_dir,
            os.path.relpath(dirpath, test_dir),
            filename.rsplit(".", 1)[0] + "-diff.png",
        )

        # Make sure the output directory exists
        os.makedirs(os.path.dirname(diff_file), exist_ok=True)

        # Create and execute compare command
        compare_cmd = "compare {} {} {}".format(
            base_file, os.path.join(dirpath, filename), diff_file
        )
        subprocess.call(compare_cmd, shell=True)

```

This may be forgotten if we don't use it somewhere, so we add it to `src-svelte/Makefile`:

```Makefile
test:
	yarn test
	python3 diff-screenshots.py
```

This means that we have to install ImageMagick on CI too, so we edit the Playwright step in `.github/workflows/tests.yaml` to include:

```yaml
jobs:
  ...
  svelte:
    ...
    steps:
      ...
      - name: Install remaining dependencies
        run: |
          yarn playwright install
          sudo apt-get install -y imagemagick
      ...
```

Unfortunately, this somehow fails on CI with the error

```
/__w/_temp/8d4c3f31-3a9c-4de5-9fbf-6d2fbb954882.sh: 2: sudo: not found
```

We see that this appears to be because we're using the `mcr.microsoft.com/playwright:v1.38.0-jammy` Docker image instead of running directly on GitHub's base CI image. We try removing that image, and find ourselves with

```
 FAIL  src/routes/storybook.test.ts > Storybook visual tests
Error: browserType.launch: 
╔══════════════════════════════════════════════════════╗
║ Host system is missing dependencies to run browsers. ║
║ Missing libraries:                                   ║
║     libwoff2dec.so.1.0.2                             ║
║     libvpx.so.7                                      ║
║     libevent-2.1.so.7                                ║
║     libopus.so.0                                     ║
║     libharfbuzz-icu.so.0                             ║
║     libgstallocators-1.0.so.0                        ║
║     libgstapp-1.0.so.0                               ║
║     libgstpbutils-1.0.so.0                           ║
║     libgstaudio-1.0.so.0                             ║
║     libgsttag-1.0.so.0                               ║
║     libgstvideo-1.0.so.0                             ║
║     libgstgl-1.0.so.0                                ║
║     libgstcodecparsers-1.0.so.0                      ║
║     libgstfft-1.0.so.0                               ║
║     libhyphen.so.0                                   ║
║     libmanette-0.2.so.0                              ║
║     libflite.so.1                                    ║
║     libflite_usenglish.so.1                          ║
║     libflite_cmu_grapheme_lang.so.1                  ║
║     libflite_cmu_grapheme_lex.so.1                   ║
║     libflite_cmu_indic_lang.so.1                     ║
║     libflite_cmu_indic_lex.so.1                      ║
║     libflite_cmulex.so.1                             ║
║     libflite_cmu_time_awb.so.1                       ║
║     libflite_cmu_us_awb.so.1                         ║
║     libflite_cmu_us_kal16.so.1                       ║
║     libflite_cmu_us_kal.so.1                         ║
║     libflite_cmu_us_rms.so.1                         ║
║     libflite_cmu_us_slt.so.1                         ║
║     libGLESv2.so.2                                   ║
║     libx264.so                                       ║
╚══════════════════════════════════════════════════════╝
```

It turns out that we can specify this command instead in `.github\workflows\tests.yaml`:

```bash
$ yarn playwright install --with-deps
```

and the tests will run. However, this changes all the screenshots. As such, we update the existing tests on main first so that the image diffs remain coherent. We also add a step to `.github\workflows\tests.yaml` to do a better screenshot diff:

```yaml
jobs:
  ...
  svelte:
    ...
    steps:
      ...
      - name: Run Svelte Tests
        run: xvfb-run ...
      - name: Do better diff
        if: always()
        run: python3 diff-screenshots.py
        working-directory: src-svelte/
      ...
```

### Buffer error

If you get

```
TypeError: invalid invocation. input should be a Uint8Array
 ❯ Proxy.imageSize ../node_modules/image-size/dist/index.js:102:15
 ❯ test.retry src/routes/storybook.test.ts:261:34
    259| 
    260|           console.log(screenshot);
    261|           const screenshotSize = sizeOf(screenshot);
       |                                  ^
```

despite `screenshot` being logged as the expected form of

```
<Buffer 89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52 00 00 03 f6 00 00 00 16 08 02 00 00 00 e5 d6 90 1c 00 00 00 06 62 4b 47 44 00 ff 00 ff 00 ff a0 bd a7 ... 4473 more bytes>
```

then one possible solution appears to be cleaning and reinstalling node modules with

```bash
$ rm -rf node_modules
$ yarn
```

However, we find that the test only fails with a new error message because we failed to restart the Storybook server. Once we do so, it fails again with the same old message. We try again with the specific dependencies we have from the lock file, as mentioned [here](https://stackoverflow.com/a/58223363):

```bash
$ yarn install --frozen-lockfile
```

and find that it actually works now after a Storybook server restart. Unfortunately, it stops working again after another `playwright install`
 browser update. Upon further debugging,

```ts
console.log(screenshot instanceof Buffer);
```

shows as true, but

```ts
console.log(screenshot instanceof Uint8Array);
```

shows as false. This is weird because `Buffer` extends `Uint8Array` in NodeJS. We manually recreate the Buffer:

```ts
          const uint8ArrayWorkaround = new Uint8Array(
            screenshot.buffer,
            screenshot.byteOffset,
            screenshot.byteLength,
          );
          const screenshotSize = sizeOf(uint8ArrayWorkaround);
```

Oddly enough, now the test passes.

### Failed to initialize launcher service

If you get

```
Error: Error: Failed to initilialise launcher service unknown: Error: Couldn't initialize "wdio-image-comparison-service".
Error: Cannot find module '../build/Release/canvas.node'
Require stack:
- /root/zamm/node_modules/canvas/lib/bindings.js
- /root/zamm/node_modules/canvas/lib/canvas.js
- /root/zamm/node_modules/canvas/index.js
- /root/zamm/node_modules/webdriver-image-comparison/build/methods/images.js
- /root/zamm/node_modules/webdriver-image-comparison/build/commands/saveScreen.js
- /root/zamm/node_modules/wdio-image-comparison-service/build/service.js
- /root/zamm/node_modules/wdio-image-comparison-service/build/index.js
    at Function.Module._resolveFilename (node:internal/modules/cjs/loader:1048:15)
    at Function.Module._resolveFilename.sharedData.moduleResolveFilenameHook.installedValue [as _resolveFilename] (/root/zamm/node_modules/@cspotcode/source-map-support/source-map-support.js:811:30)
    ...
```

this appears to be due to [a problem](https://stackoverflow.com/a/63175855) with the canvas installation script. Nuking the `node_modules` folder and reinstalling it with `yarn` appears to fix the problem.

## Debugging

You can use JavaScript to access the Storybook iframe during development. For example:

```js
window.frames["storybook-preview-iframe"].contentDocument.body.scrollHeight
window.frames["storybook-preview-iframe"].contentDocument.getElementById("#storybook-root")
```
