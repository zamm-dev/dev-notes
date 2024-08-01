# App Layout

## UI size on the Mac

We see from [this question](https://stackoverflow.com/questions/9437584/what-does-webkit-min-device-pixel-ratio-2-stand-for) and [this tutorial](https://css-tricks.com/snippets/css/retina-display-media-query/) that it's possible to target Mac Retina displays in CSS. We do so by editing `src-svelte/src/routes/styles.css` to include:

```css
@media only screen and (-webkit-min-device-pixel-ratio: 2) {
  :root {
    font-size: 14px;
  }
}
```

Now we need to set the window size to be smaller as well. We see from [this answer](https://github.com/tauri-apps/tauri/discussions/3225) that we can set the size of the window once we have a reference to the window. We then see from [the docs](https://docs.rs/tauri/latest/tauri/window/struct.Window.html) how to get that reference to the main window. We try editing `src-tauri/src/main.rs`:

```rs
...
use anyhow::anyhow;
...
use tauri::{LogicalSize, ..., Size};
...

fn main() {
    ...
            tauri::Builder::default()
                .setup(|app| {
                    ...

                    #[cfg(target_os = "macos")]
                    {
                        app.get_window("main")
                            .ok_or(anyhow!("No main window"))?
                            .set_size(Size::Logical(LogicalSize {
                                width: 600.0,
                                height: 450.0,
                            }))?;
                    }

                    Ok(())
                }
                ...;
    ...
}
```

We find that although the pre-commit hooks pass on Mac OS, the new imports get optimized away on other platforms. As such, we change the above to:

```rs
                    #[cfg(target_os = "macos")]
                    {
                        app.get_window("main")
                            .ok_or(anyhow::anyhow!("No main window"))?
                            .set_size(tauri::Size::Logical(tauri::LogicalSize {
                                width: 600.0,
                                height: 450.0,
                            }))?;
                    }
```

Next, we find that we have to adjust the background text size as well. We refactor out root font size and related calculations into `src-svelte/src/lib/preferences.ts`:

```ts
...

const STANDARD_ROOT_EM = 18;

function getRootFontSize() {
  const rem = parseFloat(getComputedStyle(document.documentElement).fontSize);

  if (isNaN(rem)) {
    console.warn("Could not get root font size, assuming default of 18px");
    return 18;
  }
  return rem;
}

export const ROOT_EM = getRootFontSize();

export function getAdjustedFontSize(fontSize: number) {
  return Math.round(fontSize * (ROOT_EM / STANDARD_ROOT_EM));
}

```

The `isNaN` check prevents Vitest failures, when there is no styling information available.

We use this in `src-svelte/src/routes/BackgroundUI.svelte`:

```ts
  ...
  import { ..., getAdjustedFontSize } from "$lib/preferences";
  ...

  const CHAR_EM = getAdjustedFontSize(26);
  ...
```

and replace the uses of in `src-svelte/src/lib/Switch.svelte` and `src-svelte/src/lib/Slider.svelte`:

```ts
  ...
  import { ..., ROOT_EM } from "./preferences";
  ...

  const overshoot = 0.4 * ROOT_EM; // how much overshoot to allow per-side
```

We notice that the info box animations now start out oversized. We find that this is because of the hardcoded size in `src-svelte/src/lib/InfoBox.svelte`, which we now edit as well:

```ts
  ...
  import {
    ...,
    getAdjustedFontSize,
  } from "./preferences";
  ...

  function revealOutline(
    ...
  ): TransitionConfig {
    ...
    const heightPerTitleLinePx = getAdjustedFontSize(26);
    ...
  }

  ...
```

Next, we notice that the settings page credits now look off, only being single-column instead of double column for the window size. We edit `src-svelte/src/routes/credits/Grid.svelte` accordingly, shrinking everything by 25% and rounding to the nearest half-integer:

```css
  ...

  /* this takes sidebar width into account */
  @media (min-width: 46rem),
    only screen and (-webkit-min-device-pixel-ratio: 2) and (min-width: 34.5rem) {
    .credits-grid {
      ...
    }
  }

  /* this takes sidebar width into account */
  @media (min-width: 55rem),
    only screen and (-webkit-min-device-pixel-ratio: 2) and (min-width: 41rem) {
    .credits-grid {
      ...
    }
  }

  /* this takes sidebar width into account */
  @media (min-width: 64rem),
    only screen and (-webkit-min-device-pixel-ratio: 2) and (min-width: 48rem) {
    .credits-grid {
      ...
    }
  }

  ...
```

We do this for every other file that has media queries, except for `src-svelte/src/routes/api-calls/[slug]/ApiCallDisplay.svelte` and `src-svelte/src/lib/controls/ButtonGroup.svelte`, where we also edit the media queries to use `min-width` instead of `max-width`, or else the lowered screen size setting won't be triggered on Retina displays:

```css
  .conversation-links {
    ...
    flex-direction: column;
    ...
  }

  .conversation.previous-links,
  .conversation.next-links {
    flex: none;
    ...
  }

  @media (min-width: 46rem),
    only screen and (-webkit-min-device-pixel-ratio: 2) and (min-width: 34.5rem) {
    .conversation-links {
      flex-direction: row;
    }

    .conversation.previous-links,
    .conversation.next-links {
      flex: 1;
    }
  }
```

We also avoid editing the media queries in `src-svelte/src/routes/chat/MessageUI.svelte`, because those ones:

```css
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
```

depend on some other settings as well, and editing these alone will result in the background not always covering the text. Manual testing reveals that the existing media queries still work fine on Retina displays.

## Rounding out the left corners of the main content

It turns out that we need to do a major refactor of our app layout to achieve this effect.

In `src-svelte/src/routes/styles.css`, rename `--color-background` to `--color-foreground` and make a new `--color-background` that's equivalent to the sidebar's background color. Now the sidebar can be colorless. However, now the body should be made explicitly white to keep all the Storybook screenshots consistent:

```css
body {
  background-color: white;
}
```

Instead, we set the background in the `.app` div. Create `src-svelte/src/routes/AppLayout.svelte` so that we can also visualize it in a Storybook story:

```svelte
<script>
  import Sidebar from "./Sidebar.svelte";
  import Background from "./Background.svelte";
  import "./styles.css";
</script>

<div class="app">
  <Sidebar />

  <div class="main-container">
    <div class="background-layout">
      <Background />
    </div>
    <main>
      <slot />
    </main>
  </div>
</div>

<style>
  .app {
    box-sizing: border-box;
    height: 100vh;
    width: 100vw;
    position: absolute;
    top: 0;
    left: 0;
    background-color: var(--color-background);
    --main-corners: var(--corner-roundness) 0 0 var(--corner-roundness);
  }

  .main-container {
    height: 100vh;
    box-sizing: border-box;
    margin-left: var(--sidebar-width);
    overflow: scroll;
    border-radius: var(--main-corners);
    background-color: var(--color-foreground);
    box-shadow: calc(-1 * var(--shadow-offset)) 0 var(--shadow-blur) 0 #ccc;
  }

  .background-layout {
    z-index: 0;
    border-radius: var(--main-corners);
    position: absolute;
    top: 0;
    bottom: 0;
    left: var(--sidebar-width);
    right: 0;
  }

  main {
    position: relative;
    z-index: 1;
    padding: 1em;
  }
</style>

```

The main things to keep in mind here are:

- We want the main content to have rounded left corners
- We don't want the main content casting any shadows on the selected icon of the sidebar, or for the selected icon to cast any shadows on the main content either. We will need to understand the [stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Understanding_z-index/Stacking_context) to achieve this.
- We do want the main content to cast a shadow on the rest of the sidebar
- We don't want the animated background of the main content to bleed over into the sidebar or general page background (potentially visible through the bottom left corner)
- We want the main content to lay on top of the animated background
- We want the sidebar, the round corners of the main content, and the background to stay fixed while the contents scroll
- We want the sidebar and rounded corners to span the entire height of the window, without producing any scrollbars of their own

The corresponding story at `src-svelte/src/routes/AppLayout.stories.ts`:

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

and every time we create a new story, we should add it to the list at `src-svelte/src/routes/storybook.test.ts`. Now that we have decided to use the `Layout/` folder, we should also move the existing sidebar stories to be under that instead. In doing so, we notice that the `dashboard-selected` variant of the sidebar story is somehow no longer listed in the test, but clearly it was at some point because the screenshot is there. We do a git blame on main, which is at commit `6e94a38`, and see that the last time this changed was `d25aac3` when dashboard stories were moved under general screen stories. However, this change was just shifting the lines around and not meaningful on its own. We go to its parent, `05b4241`, and git blame again. To our surprise, this time it is `4e494a0`, where the screenshot was first defined. So when did the dashboard screenshot get added?

We go to `src-svelte/screenshots/baseline/navigation/sidebar/dashboard-selected.png` and see that it was created with 79b6c53. It appears the file got copied without us remembering to update the Storybook test. This is why automation is important.

Now the original `src-svelte/src/routes/+layout.svelte` can be simplified to:

```svelte
<script>
  import AppLayout from "./AppLayout.svelte";
  import "./styles.css";
</script>

<AppLayout>
  <slot />
</AppLayout>

```

Meanwhile in `src-svelte/src/routes/BackgroundUI.svelte`, to clean up scrollbars in the Storybook story:

```css
  .background {
    ...
    overflow: hidden;
    ...
  }

  .bg {
    ...
    position: absolute;
    ...
  }
```

In `src-svelte/src/routes/Sidebar.svelte`, to ensure rendering succeeds in Storybook despite the use of `$app/stores`:

```svelte
<script lang="ts">
  import { page } from "$app/stores";
  ...

  let currentRoute: string;
  $: currentRoute = $page.url?.pathname || "/";
</script>
```

In `src-svelte/src/routes/SidebarUI.svelte`:

```css
  header {
    float: left;
    box-sizing: border-box;
  }
```

We remove z-index, width, background-color, position, top, and left from the header because they're no longer needed.

We remove `header::before` entirely because now the shadow is cast properly by the main element and captured in the screenshot for the overall layout instead of the screenshot for the sidebar itself.

## Centering content horizontally

Edit `src-svelte/src/routes/AppLayout.svelte` to constrain the maximum width of the app layout and center it horizontally:

```css
  main {
    ...
    max-width: 70rem;
    margin: 0 auto;
  }
```

## Updating base animation speed with standard duration

We had previously introduced `--base-animation-speed` to the layout in [`settings.md`](/zamm-notes/settings.md). Now we introduce the standard duration to the layout as well, because it is in practice used more often.

We first create `src-svelte/src/routes/AnimationControl.svelte` to consolidate the animation control logic out of `AppLayout.svelte` so that it can be reused across "prod" and testing:

```svelte
<script lang="ts">
  import {
    animationSpeed,
    animationsOn,
    standardDuration,
  } from "$lib/preferences";

  $: standardDurationMs = $standardDuration.toFixed(2) + "ms";
</script>

<div
  class="container"
  class:animations-disabled={!$animationsOn}
  style="--base-animation-speed: {$animationSpeed}; --standard-duration: {standardDurationMs};"
>
  <slot />
</div>

<style>
  .container.animations-disabled :global(*) {
    animation-play-state: paused !important;
    transition: none !important;
  }
</style>

```

We create `src-svelte/src/routes/AnimationControl.test.ts` as well, containing much test logic from the original `AppLayout.svelte`:

```ts
import { expect, test } from "vitest";
import "@testing-library/jest-dom";
import { render } from "@testing-library/svelte";
import AnimationControl from "./AnimationControl.svelte";
import {
  animationsOn,
  animationSpeed,
} from "$lib/preferences";

describe("AnimationControl", () => {
  beforeEach(() => {
    animationsOn.set(true);
    animationSpeed.set(1);
  });

  test("will enable animations by default", async () => {
    render(AnimationControl, {});

    const animationControl = document.querySelector(".container") as Element;
    expect(animationControl.classList).not.toContainEqual("animations-disabled");
    expect(animationControl.getAttribute("style")).toEqual("--base-animation-speed: 1; --standard-duration: 100.00ms;");
  });

  test("will disable animations if preference overridden", async () => {
    animationsOn.set(false);
    render(AnimationControl, {});

    const animationControl = document.querySelector(".container") as Element;
    expect(animationControl.classList).toContainEqual("animations-disabled");
    expect(animationControl.getAttribute("style")).toEqual("--base-animation-speed: 1; --standard-duration: 0.00ms;");
  });

  test("will slow down animations if preference overridden", async () => {
    animationSpeed.set(0.9);
    render(AnimationControl, {});

    const animationControl = document.querySelector(".container") as Element;
    expect(animationControl.classList).not.toContainEqual("animations-disabled");
    expect(animationControl.getAttribute("style")).toEqual("--base-animation-speed: 0.9; --standard-duration: 111.11ms;");
  });
});

```

Now we can refactor `src-svelte/src/routes/AppLayout.svelte` to use this logic instead, and remove the parts that we have now consolidated into `AnimationControl.svelte`:

```svelte
<script lang="ts">
  ...
  import AnimationControl from "./AnimationControl.svelte";
  ...
</script>

<div id="app">
  <AnimationControl>
    <Sidebar />
    ...
  </AnimationControl>
</div>

<style>
  ...
</style>
```

We edit `src-svelte/src/routes/AppLayout.test.ts` as well to make the preference-setting tests more consistent, and to remove the test logic that we've since moved into `AnimationControl`:

```ts
describe("AppLayout", () => {
  ...

  test("will set animation if animation preference overridden", async () => {
    expect(get(animationsOn)).toBe(true);
    expect(tauriInvokeMock).not.toHaveBeenCalled();

    playback.addSamples(
      "../src-tauri/api/sample-calls/get_preferences-animations-override.yaml",
    );

    render(AppLayout, { currentRoute: "/" });
    await tickFor(3);
    expect(get(animationsOn)).toBe(false);
    expect(tauriInvokeMock).toBeCalledTimes(1);
  });

  test("will set animation speed if speed preference overridden", async () => {
    expect(get(animationSpeed)).toBe(1);
    expect(tauriInvokeMock).not.toHaveBeenCalled();

    playback.addSamples(
      "../src-tauri/api/sample-calls/get_preferences-animation-speed-override.yaml",
    );

    render(AppLayout, { currentRoute: "/" });
    await tickFor(3);
    expect(get(animationSpeed)).toBe(0.9);
    expect(tauriInvokeMock).toBeCalledTimes(1);
  });
});

```

We port these changes to `src-svelte/src/lib/__mocks__/MockAppLayout.svelte`, which had previously done its own imitation of `--base-animation-speed`:

```svelte
<script lang="ts">
  import AnimationControl from "../../routes/AnimationControl.svelte";
  import Snackbar from "$lib/snackbar/Snackbar.svelte";
</script>

<div class="storybook-wrapper">
  <AnimationControl>
    <slot />
    <Snackbar />
  </AnimationControl>
</div>

<style>
  .storybook-wrapper {
    max-width: 50rem;
    position: relative;
  }
</style>

```

We edit `src-svelte/src/lib/__mocks__/MockPageTransitions.svelte` to use `MockAppLayout`:

```svelte
<script lang="ts">
  import MockAppLayout from "./MockAppLayout.svelte";
  ...
</script>

<MockAppLayout>
  <PageTransition ...>
    ...
  </PageTransition>
</MockAppLayout>

```

We manually test the `Dashboard/Full Page` story because animations are not covered by our tests. Next, we do the same for `src-svelte/src/lib/__mocks__/MockTransitions.svelte`, where we get rid of the reset button because Storybook's reset button works just fine for our purposes:

```svelte
<script lang="ts">
  import { onMount } from "svelte";
  import MockAppLayout from "./MockAppLayout.svelte";

  let visible = false;

  onMount(() => {
    setTimeout(() => {
      visible = true;
    }, 50);
  });
</script>

<MockAppLayout>
  {#if visible}
    <slot />
  {/if}
</MockAppLayout>

```

We manually test the InfoBox stories as well to confirm that they're still working as expected.

For one of the last test-specific mock layouts, we edit `src-svelte/src/routes/PageTransitionView.svelte` to use the new mock setup instead of its own custom ones:

```svelte
<script lang="ts">
  import MockAppLayout from "$lib/__mocks__/MockAppLayout.svelte";
  ...
</script>

<MockAppLayout>
  ...
  <PageTransition ...>
    ...
  </PageTransition>
</MockAppLayout>

<style>
  ...
</style>
```

Next, we start replacing all custom calculations with the new standard duration variables and making sure to manually test their transitions, starting with `src-svelte/src/lib/controls/Button.svelte`:

```css
  .outer,
  .inner {
    ...
    transition-property: filter, transform;
    transition: calc(0.5 * var(--standard-duration)) ease-out;
  }
```

and `src-svelte/src/lib/controls/TextInput.svelte`:

```css
  input[type="text"] + .focus-border {
    ...
    transition: width calc(0.5 * var(--standard-duration)) ease-out;
  }
```

and `src-svelte/src/lib/snackbar/Snackbar.svelte`:

```svelte
<script lang="ts">
  import { standardDuration } from "$lib/preferences";
  ...

  $: setBaseAnimationDurationMs($standardDuration);
</script>

<div class="snackbars">
  {#each ...}
    <div
      in:fly|global={{ y: "1rem", duration: $standardDuration }}
      out:fade|global={{ duration: $standardDuration }}
      ...
    >
      ...
    </div>
  {/each}
</div>

```

and `src-svelte/src/lib/Slider.svelte`:

```ts
  const transitionAnimation =
    `transition: transform var(--standard-duration) ease-out;`;
```

and `src-svelte/src/lib/Switch.svelte`:

```ts
  const transitionAnimation = `
    transition: transform var(--standard-duration);
    transition-timing-function: cubic-bezier(0, 0, 0, 1.3);
  `;
```

and `src-svelte/src/routes/components/api-keys/Service.svelte`:

```css
  .api-key {
    ...
    transition: var(--standard-duration) ease-in;
  }
```

and `src-svelte/src/routes/PageTransition.svelte`:

```ts
  ...
  import { standardDuration } from "$lib/preferences";
  ...

  // twice the speed of sidebar UI slider
  $: totalDurationMs = 2 * $standardDuration;
  ...
```

and `src-svelte/src/routes/SidebarUI.svelte`:

```css
  header {
    --animation-duration: calc(2 * var(--standard-duration));
    ...
  }
```

We add a default to `src-svelte/src/routes/styles.css` in case anything goes wrong and the CSS variable ends up undefined:

```css
  :root {
    ...
    --standard-duration: 100ms;
    ...
  }
```

Finally, we come to `src-svelte/src/routes/BackgroundUI.svelte`:

```css
  .background {
    ...
    --base-duration: calc(150 * var(--standard-duration));
    ...
  }
```

With this new setup, the background no longer appears. Because this works already in "prod," we create a new view just for the background element so that test code does not impact regular code. We create `src-svelte/src/routes/BackgroundUIView.svelte`:

```svelte
<script lang="ts">
  import MockAppLayout from "$lib/__mocks__/MockAppLayout.svelte";
</script>

<MockAppLayout>
  <div class="background-container">
    <slot />
  </div>
</MockAppLayout>

<style>
  .background-container {
    position: fixed;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
  }
</style>

```

We edit `src-svelte/src/routes/BackgroundUI.stories.ts` accordingly to use this new view instead:

```ts
...
import BackgroundUIView from "./BackgroundUIView.svelte";

export default {
  ...
  decorators: [
    ...,
    (story: StoryFn) => {
      return {
        Component: BackgroundUIView,
        slot: story,
      };
    },
  ],
};

...
```

Finally, we have to edit `src-svelte/src/routes/storybook.test.ts` because now that we're using the new `MockLayout`, the actual components being tested are nested more deeply than before:

```ts
  const takeScreenshot = async (...) => {
    ...
    if (elementClass?.includes("storybook-wrapper")) {
      locator = ".storybook-wrapper > :first-child > :first-child";
    }
    ...
  };
```

We find that two screenshots have now changed: `src-svelte/screenshots/baseline/layout/sidebar/dashboard-selected.png` and `src-svelte/screenshots/baseline/layout/sidebar/settings-selected.png` now no longer have rounded corners at the bottom left corner, which is what we want after all but never realized was the case, so we update those screenshots as well.

While committing, we find that eslint complains

```
/root/zamm/src-svelte/src/lib/Slider.svelte
  14:1  error  This line has a length of 89. Maximum allowed is 88  max-len
```

The line in question is

```ts
  const transitionAnimation = `transition: transform var(--standard-duration) ease-out;`;
```

If we simply put a newline after the `=`, `prettier` will reformat it back to its current state, so we have to split it more vigorously:

```ts
  const transitionAnimation =
    `transition: ` + `transform var(--standard-duration) ease-out;`;
```

We do something similar with `src-svelte/src/routes/AnimationControl.svelte`, where there is the line

```svelte
<div
  ...
  style="--base-animation-speed: {$animationSpeed}; --standard-duration: {standardDurationMs};"
>
```

We turn this into

```svelte
<script lang="ts">
  ...
  $: style =
    `--base-animation-speed: ${$animationSpeed}; ` +
    `--standard-duration: ${standardDurationMs};`;
</script>

<div ... {style}>
  ...
</div>

```

### Debugging reveal animation regression

At some point, we realize that the reveal animation for the Active/Inactive indicator starts too late. The indicator appears long after everything else has appeared, such that the fade-in to green doen't even appear anymore because the fade has completed by the time the element appears. After digging through our Git history to find the last time this worked, we see that things are still fine on `88fa2e9` but are failing by `4c1ccaf`. We do a git bisect:

```bash
$ git bisect start 4c1ccaf 88fa2e9
Bisecting: 2 revisions left to test after this (roughly 1 step)
[7cc5050a60758f12f64305684ce0fddad75a29e5] Allow sample calls to represent API call failures as well
$ git bisect good                 
Bisecting: 0 revisions left to test after this (roughly 1 step)
[bd22202637df28192a199a451320c2afc53fc177] Merge pull request #41 from amosjyng/refactor/sample-call-failures
$ git bisect good
4c1ccaf4dc5a3eacb3f43e8f9db04fbfa5e6ee7f is the first bad commit
commit 4c1ccaf4dc5a3eacb3f43e8f9db04fbfa5e6ee7f
Author: Amos Jun-yeung Ng <me@amos.ng>
Date:   Sat Jan 6 11:15:36 2024 +1100

    Refactor animation settings to use standardDuration
    ...
```

Back then we didn't edit `src-svelte/src/lib/InfoBox.svelte` because it uses its own

```ts
  $: timingScaleFactor = shouldAnimate ? 1 / $animationSpeed : 0;
```

instead of the usual standardDuration. It appears that file is not the culprit, as the transition timing returned is the same for both elements of the API keys display.

Instead, reverting the files one by one, we see that reverting `src-svelte/src/routes/components/api-keys/Service.svelte` fixes the bug. The change in question is going from

```css
    transition-property: background-color, box-shadow;
    transition-duration: calc(0.1s / var(--base-animation-speed));
    transition-timing-function: ease-in;
```

to

```css
    transition-property: background-color, box-shadow;
    transition: var(--standard-duration) ease-in;
```

without realizing that the `transition` applies to all properties, not just the properties defiend in `transition-property`. We fix this while keeping the spirit of the refactor by changing the lines to

```css
    transition-property: background-color, box-shadow;
    transition-duration: var(--standard-duration);
    transition-timing-function: ease-in;
```

This would be a good example of an animation regression that you would ideally be able to write a test for with an animation testing framework. Even without it, this is where the infrastructure we've built, including our Git history, comes through for us.

## Positioning the snackbar

We notice that sometimes the snackbar is covered by other things. We edit `src-svelte/src/lib/snackbar/Snackbar.svelte`:

```css
  .snackbars {
    z-index: 100;
    ...
  }
```

Because `src-svelte/src/lib/__mocks__/MockAppLayout.svelte` looks like this:

```svelte
...

<div class="storybook-wrapper">
  <AnimationControl>
    <slot />
    <Snackbar />
  </AnimationControl>
</div>

...
```

we make sure to mark `src-svelte/src/routes/AnimationControl.svelte` as a positioned element:

```css
  .container {
    ...
    position: relative;
  }
```

We check the same for prod, and we see that `src-svelte/src/routes/AppLayout.svelte` looks like this:

```svelte
<div id="app">
  <AnimationControl>
    <Sidebar />

    <div class="main-container">
      <div class="background-layout">
        <Background />
      </div>
      <Snackbar />

      <main>
        ...
      </main>
    </div>
  </AnimationControl>
</div>

```

We edit the CSS for the parent element accordingly:

```css
  .main-container {
    ...
    position: relative;
    ...
  }
```

However, we find out from our end-to-end tests that the sidebar now appears under the main content. We realize we should undo this last change, because the `div` created by `AnimationControl` will now be the one that the snackbar's `z-index` applies to.

## Scrollbars

On Windows, we notice that the scrollbar appears in the y direction even when it is not needed. Furthermore, the page transition animation makes a scrollbar temporarily flicker into existence in the x direction as well. We edit the CSS for each of the elements in `src-svelte\src\routes\AppLayout.svelte` until we find the one that makes a difference:

```css
  .main-container {
    ...
    overflow-y: auto;
    overflow-x: hidden;
  }
```

We confirm that curiously enough, the Webkit install on Linux Mint does not have this problem.

## Off-white background

We find that the pure white background provides no contrast between the background and the title bar on the Mac and on Windows. We edit `src-svelte/src/routes/styles.css` to provide a new [offwhite](https://create.vista.com/colors/color-names/off-white/) color:

```css
  ...

  :root {
    ...
    --color-offwhite: #FAF9F6;
    ...
  }

  ...

  body {
    background-color: var(--color-offwhite);
  }

  ...
```

We notice that nothing changes just yet. That's because the app layout itself has a background color. We therefore edit `src-svelte/src/routes/AppLayout.svelte` as well:

```css
  .main-container {
    ...
    background-color: var(--color-offwhite);
    ...
  }
```

Now we edit the sidebar as well to match:

```css
  ...

  .indicator {
    ...
    background-color: var(--color-offwhite);
    ...
  }

  ...

  .indicator::before {
    ...
    box-shadow: ... var(--color-offwhite);
  }

  .indicator::after {
    ...
    box-shadow: ... var(--color-offwhite);
  }
```

We find that almost all of our screenshot tests now fail due to the body background color change. To make the diffs manageable, we revert the body background color in `src-svelte/src/routes/styles.css`:

```css
  body {
    background-color: white;
  }
```

Unfortunately, it seems Webdriver does not do a diff unless the difference in color is drastic enough, as our tests on CI pass despite the change. We update the gold screenshots anyway.

## Matrix-style background animation

For fun, we try editing the examples [here](https://medium.com/@javascriptacademy.dev/matrix-raining-code-effect-using-javascript-3da1e4cdf3da), [here](https://github.com/sumitKcs/matrix-effect), [here](https://github.com/jcubic/cmatrix), and [here](https://gist.github.com/kunaltyagi/eb8db625141b6b9d295a), and adapt it to our Svelte code by editing `src-svelte\src\routes\BackgroundUI.svelte` as such:

```svelte
<script lang="ts">
  import { onMount } from "svelte";

  const CHAR_EM = 20;
  const CHAR_GAP = 5;
  const BLOCK_SIZE = CHAR_EM + CHAR_GAP;
  const RESPAWN_THRESHOLD = 0.975;
  const ANIMATE_INTERVAL_MS = 75;
  const CHAR_INTERVAL_MS = 200;
  const ANIMATES_PER_CHAR = Math.round(CHAR_INTERVAL_MS / ANIMATE_INTERVAL_MS);
  const DDJ = [
    "道可道非常道",
    "名可名非常名",
    "無名天地之始",
    "有名萬物之母",
    "故常無欲以觀其妙",
    "常有欲以觀其徼",
    "此兩者同出而異名",
    "同謂之玄",
    "玄之又玄",
    "眾妙之門",
  ];
  export let animated = false;
  let animationState: string;
  let background: HTMLDivElement | null = null;
  let canvas: HTMLCanvasElement | null = null;
  let ctx: CanvasRenderingContext2D | null = null;
  let animateInterval: NodeJS.Timeout | undefined = undefined;
  let dropsPosition: number[] = [];
  let dropsAnimateCounter: number[] = [];
  let numColumns = 0;

  function stopAnimating() {
    clearInterval(animateInterval);
    animateInterval = undefined;
  }

  function startAnimating() {
    if (animateInterval) {
      console.warn("Animation already running");
      return;
    }

    animateInterval = setInterval(draw, ANIMATE_INTERVAL_MS);
  }

  function resizeCanvas() {
    if (!canvas || !background) {
      return;
    }

    stopAnimating();

    canvas.width = background.clientWidth;
    canvas.height = background.clientHeight;
    numColumns = Math.ceil((canvas.width - CHAR_GAP) / BLOCK_SIZE);

    ctx = canvas.getContext("2d");
    if (!ctx) {
      console.warn("Canvas context not available");
      return;
    }
    ctx.fillStyle = "#FAF9F6";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    dropsPosition = Array(numColumns).fill(1);
    dropsAnimateCounter = Array(numColumns).fill(0);

    startAnimating();
  }

  function draw() {
    if (!ctx || !canvas || numColumns === 0) {
      console.warn("Canvas not ready for drawing");
      return;
    }

    ctx.fillStyle = "#FAF9F633";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.fillStyle = "#D5CDB4C0";
    ctx.font = CHAR_EM + "px sans-serif";
    
    for (var column = 0; column < dropsPosition.length; column++) {
      const textLine = DDJ[column % DDJ.length];
      const textCharacter = textLine[dropsPosition[column] % textLine.length];
      ctx.fillText(textCharacter, CHAR_GAP + column * BLOCK_SIZE, dropsPosition[column] * BLOCK_SIZE);

      if (dropsPosition[column] * BLOCK_SIZE > canvas.height && Math.random() > RESPAWN_THRESHOLD)
        dropsPosition[column] = 0;

      if (dropsAnimateCounter[column]++ % ANIMATES_PER_CHAR === 0) {
        dropsPosition[column]++;
      }
    }
  }

  onMount(() => {
    resizeCanvas();
    window.addEventListener("resize", resizeCanvas);

    return () => {
      stopAnimating();
      window.removeEventListener("resize", resizeCanvas);
    };
  });

  $: animationState = animated ? "running" : "paused";
</script>

<div class="background" bind:this={background}>
  <canvas bind:this={canvas}></canvas>
</div>

<style>
  .background {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: -100;
    overflow: hidden;
  }
</style>

```

using lines from the beginning of the [Dao De Jing](https://www.gutenberg.org/files/49965/49965-h/49965-h.htm).

The animation is a bit blocky at low speeds, so we try to see if animating it more smoothly would help. However, a little while later:

```ts
  ...
  const DROP_SPEED = 2;
  const TRAIL_LENGTH = 5;
  ...
  let dropsCharIndex: number[] = [];
  ...

  function resizeCanvas() {
    ...
    dropsCharIndex = Array(numColumns).fill(0);
    ...
  }

  function draw() {
    ...
    ctx.fillStyle = "#FAF9F6";
    ...

    for (var column = 0; column < dropsPosition.length; column++) {
      const textLine = DDJ[...];
      for (var i = dropsCharIndex[column]; dropsCharIndex[column] - i < TRAIL_LENGTH; i--) {
        const textCharacter = textLine[i % textLine.length];
        ctx.fillText(
          textCharacter,
          CHAR_GAP + column * BLOCK_SIZE,
          dropsPosition[column] + (i * BLOCK_SIZE),
        );
      }

      if (dropsPosition[column] > canvas.height && Math.random() > RESPAWN_THRESHOLD)
        dropsPosition[column] = 0;

      dropsPosition[column] += DROP_SPEED;

      if (dropsAnimateCounter[column]++ % ANIMATES_PER_CHAR === 0) {
        dropsCharIndex[column]++;
      }
    }
  }
```

we realize that the sudden addition of the next character is going to look chunky no matter what anyways. We revert the changes and keep the original version.

Note that the way this works is that the entire canvas gets painted over as a transparent layer every frame; this is what causes previously painted text to appear to fade out. The new text is then painted on the screen. We choose the transparency so that the text gradually fades in over multiple frames, and gradually fades back out again.

Note that the beginning features a row of characters flowing down across the screen. We edit the respawn logic according to how it's done [here](https://codepen.io/riazxrazor/pen/Gjomdp), so that we don't have to keep regenerating a random number every time it goes past the canvas, and so that the text starts out random:

```ts
  ...
  let numRows = 0;
  ...

  function resizeCanvas() {
    ...
    numRows = Math.ceil(canvas.height / BLOCK_SIZE);

    ...

    dropsPosition = Array(numColumns)
      .fill(0)
      .map(() => Math.ceil(Math.random() * -numRows));
    ...
  }

  function draw() {
    ...

    for (var column = 0; column < dropsPosition.length; column++) {
      ...

      if (dropsPosition[column] > numRows) {
        dropsPosition[column] = Math.ceil(Math.random() * -numRows);
      }

      ...
    }
  }
```

In order to test this out better in the browser, we add the background to full page layouts in `src-svelte/src/lib/__mocks__/MockFullPageLayout.svelte`:

```svelte
<script lang="ts">
  ...
  import BackgroundUi from "../../routes/BackgroundUI.svelte";
</script>

<div id="mock-full-page-layout" ...>
  <div class="bg">
    <BackgroundUi />
  </div>
  ...
</div>

<style>
  .bg {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: -1;
  }

  ...
</style>
```

To make the effects more visible even when the info boxes take up most of the screen, we edit `src-svelte/src/lib/InfoBox.svelte` to feature more transparency:

```css
  .border-box {
    ...
    opacity: 0.85;
  }
```

We try to see if we can edit this to have a blurred background, which would allow us to provide even more transparency while keeping the foreground layer readable:

```svelte
<section ...>
  ...
  <div class="border-container">
    <div class="border-box" in:revealOutline|global={timing.borderBox}>
      <div class="blur"></div>
      <div class="cut-corners"></div>
    </div>

    ...
  </div>
</section>

<style>
  ...

  .border-box {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    --cut: 5rem;
  }

  .blur {
    --cut-depth: calc(15rem * cos(45deg));
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    clip-path: polygon(
      0% var(--cut-depth),
      var(--cut-depth) 0%,
      100% 0%,
      100% calc(100% - var(--cut-depth)),
      calc(100% - var(--cut-depth)) 100%,
      0% 100%
    );
    backdrop-filter: blur(4px);
  }

  .cut-corners {
    ...
    opacity: 0.7;
  }

  ...
</style>
```

where the former styles for `.border-box` are changed to `.cut-corners`.

While this CSS doesn't *quite* do what we want, once we set `cut-corners` to `display: none`, it does enough to demonstrate to us that while Firefox successfully applies the background blur to only the area defined by the clip-path, WebKit applies the blur to the entire rectangular area. We see from [this](https://stackblitz.com/edit/js-dljkdj?file=index.js) and [this](https://codepen.io/thebabydino/pen/zbjBqL) example (which apparently comes from [this post](https://css-tricks.com/blurred-borders-in-css/)) that the idea of applying the blur effect to a clipped area should be technically feasible. We [modify](https://codepen.io/amosjyng-the-reactor/pen/GRLePGq) the last example a little:

```html
<div class="background"></div>

<div class="blur"></div>

<div class="text">Blurred border box with the backdrop-filter + a single pseudo approach. Doesn't work in Firefox (no backdrop-filter support) or in Edge (no clip-path support). Needs the Experimental Web Platform features flag enabled in chrome://flags in order to work in Chrome.</div>
```

and

```scss
$url: url(https://images.unsplash.com/photo-1544070078-a212eda27b49?ixlib=rb-1.2.1&q=85&fm=jpg&crop=entropy&cs=srgb&ixid=eyJhcHBfaWQiOjE0NTg5fQ);
$b: 1.5em; // border-width

div {
	position: absolute;
	width: 600px;
	height: 400px;
	box-sizing: border-box;
	margin: 50px;
	--cut: 50px;
}

div.background {
	margin: 0;
	width: 700px;
	height: 500px;
	background: $url center/cover;
}

div.text {
	color: #000;
	padding: calc(var(--cut));
	font: 600 1.5em/ 1.375 segoe script, comic sans ms, cursive;
	text-shadow: 1px 1px #fff;
}

div.blur {	
	&:before {
		position: absolute;
		/* zero all offsets */
		top: 0; right: 0; bottom: 0; left: 0;
		background: rgba(#FFF, .7);
		/* doesn't work in Firefox */
		backdrop-filter: blur(9px);
		/* doesn't work in Edge */
		--cut-depth: calc(2 * var(--cut) * cos(45deg));
		--poly: polygon(
      0% var(--cut-depth),
      var(--cut-depth) 0%,
      100% 0%,
      100% calc(100% - var(--cut-depth)),
      calc(100% - var(--cut-depth)) 100%,
      0% 100%
    );;
		-webkit-clip-path: var(--poly);
						clip-path: var(--poly);
		content: ''
	}
}
```

to see that it is in fact possible to separate the layers out to apply a clipped blur in both Firefox and Chrome. Now we can try to figure out what is causing the clipped blur to not work in ZAMM.

We edit it a little further, to add in our round filter:

```html
<svg ...>
  <defs>
    <filter id="round">
      ...
    </filter>
  </defs>
  ...
</svg>
```

and apply it and the drop shadow to the background layer

```scss
div.blur {
  filter: url(#round) drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.4));

  ...
}
```

We find that [this](https://codepen.io/amosjyng-the-reactor/pen/OJGGVwZ) does not work in Firefox or Chrome. Applying the filter to the `::before` pseudo element doesn't work either. This may be because of the [backdrop root](https://stackoverflow.com/questions/63907743/parent-element-backdrop-filter-does-not-apply-for-its-child/63954144#63954144).

We add another div for the border, and add it on top of the background blur:

```html
<div class="border"></div>
```

We then edit the styling, moving the filters out of the blur layer and into the border box layer:

```scss
div {
	...
	--cut: 50px;
	--cut-depth: calc(...);
		--poly: polygon(
      ...
    );;
}

...

div.blur {
	&:before {
		--offset: 0px;
		...
		top: var(--offset); right: var(--offset); bottom: var(--offset); left: var(--offset);
    ...
  }
}

div.border {
	filter: url(#round) drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.4));
	opacity: 0.5;
	
	&:before {
		position: absolute;
		/* zero all offsets */
		top: 0; right: 0; bottom: 0; left: 0;
		background: #FFF;
		-webkit-clip-path: var(--poly);
						clip-path: var(--poly);
		content: '';
	}
}
```

The opacity needs to be applied to the border element itself and not its pseudo element, or else the round filter won't work. The blur filter originally has a non-zero offset due to initial worries over the sharp edges of the blur causing the rounded corners to be distorted, but it appears the spillover is visually negligible.

Now that we have a proof-of-concept working in both Firefox and Chrome, we try to see if it's possible to fit this into ZAMM, which is rendered using an older version of WebKit. We try editing `src-svelte\src\lib\InfoBox.svelte` again:

```svelte
<section
  ...
>
  ...
  <div class="border-container">
    <div class="border-box" ...>
      <div class="blur"></div>
      <div class="background"></div>
    </div>
    ...
  </div>
</section>

<style>
  ...

  .border-box, .border-box div, .border-box div::before {
    width: 100%;
    height: 100%;
    position: absolute;
  }

  .border-box {
    z-index: 1;
    --cut-depth: calc(2 * var(--cut) * cos(45deg));
		--poly: polygon(
      0% var(--cut-depth),
      var(--cut-depth) 0%,
      100% 0%,
      100% calc(100% - var(--cut-depth)),
      calc(100% - var(--cut-depth)) 100%,
      0% 100%
    );
  }

  .border-box .blur::before {
		backdrop-filter: blur(9px);		
		-webkit-clip-path: var(--poly);
						clip-path: var(--poly);
		content: "";
  }

  .border-box .background {
    filter: url(#round) drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.4));
	  opacity: 0.5;
  }

  .border-box .background::before {
    background: #FFF;
		-webkit-clip-path: var(--poly);
						clip-path: var(--poly);
		content: '';
  }

  ...
</style>
```

This works fine in Firefox, but on Chrome we see that for some reason, the blur is not clipped by the `clip-path`.

While debugging this, we notice that the "Reusable > InfoBox > Mount Transition" story is no longer correctly animating the reveal of the text content. Rather than debug what change caused this, we edit `src-svelte/src/lib/InfoBoxView.svelte` to ensure that `atomic-reveal` is applied to the entire paragraph:

```svelte
<div ...>
  <InfoBox ...>
    <p class="atomic-reveal">
      ...
    </p>
  </InfoBox>
</div>
```

We realize that this applies to `src-svelte\src\routes\PageTransitionView.svelte` too, so we edit that as well, also moving the "Toggle route" button outside of the MockAppLayout because new changes to CSS means that otherwise, the button will be below the info box layer and unclickable:

```svelte
<button class="route-toggle" ...>Toggle route</button>

<MockAppLayout>
  ...
      <InfoBox title="Simulation">
        <p class="atomic-reveal">
          ...
        </p>
      </InfoBox>
```

While debugging this, we find that removing the definition for `--corner-roundness` in `src-svelte\src\routes\styles.css` fixes the issue in the Tauri app, but not in the Storybook story in Chrome. Further debugging reveals that disabling the rule

```css
  #main-container {
    border-radius: 1px;
  }
```

in `src-svelte\src\routes\AppLayout.svelte` produces the same effect. It's unclear why border-radius would have such effects far downstream of the parent element.

As for Chrome, even copying the rendered HTML into the codepen:

```html

<div class="background"></div>

<section class="container s-BIEZu5ODCI1i"><svg width="0" height="0" xmlns="http://www.w3.org/2000/svg" version="1.1" style="visibility: hidden; position: absolute;"><defs><filter id="round"><feGaussianBlur in="SourceGraphic" stdDeviation="3" result="blur"></feGaussianBlur><feColorMatrix in="blur" mode="matrix" values="1 0 0 0 0  0 1 0 0 0  0 0 1 0 0  0 0 0 19 -9" result="goo"></feColorMatrix><feComposite in="SourceGraphic" in2="goo" operator="atop"></feComposite></filter></defs></svg><!--<RoundDef>--> <div class="border-container s-BIEZu5ODCI1i"><div class="border-box s-BIEZu5ODCI1i"><div class="blur s-BIEZu5ODCI1i"></div></div></div></section>
```

and editing the SCSS to be regular CSS:

```css
div.background, section {
	--w: 800px;
	--h: 800px;
	position: absolute;
	width: var(--w);
	height: var(--h);
	box-sizing: border-box;
	margin: 50px;
	--cut: 100px;
	--cut-depth: calc(2 * var(--cut) * cos(45deg));
		--poly: polygon(
      0% var(--cut-depth),
      var(--cut-depth) 0%,
      100% 0%,
      100% calc(100% - var(--cut-depth)),
      calc(100% - var(--cut-depth)) 100%,
      0% 100%
    );;
}

div.blur {
  width: 100%;
    height: 100%;
    position: absolute;
}

div.blur::before {
		--offset: 0px;
		position: absolute;
		top: var(--offset); right: var(--offset); bottom: var(--offset); left: var(--offset);
		backdrop-filter: blur(9px);
		-webkit-clip-path: var(--poly);
						clip-path: var(--poly);
		content: ''
}

div.background {
	margin: 0;
	width: calc(var(--w) + 100px);
	height: calc(var(--h) + 100px);
	background: url(https://images.unsplash.com/photo-1544070078-a212eda27b49?ixlib=rb-1.2.1&q=85&fm=jpg&crop=entropy&cs=srgb&ixid=eyJhcHBfaWQiOjE0NTg5fQ) center/cover;
}

```

so that `src-svelte\src\lib\InfoBox.svelte` can contain exactly the same CSS:

```css
<script lang="ts">
  import getComponentId from "./label-id";
  import RoundDef from "./RoundDef.svelte";
  import { cubicInOut, cubicOut, linear } from "svelte/easing";
  import { animationSpeed, animationsOn } from "./preferences";
  import { fade, type TransitionConfig } from "svelte/transition";
  import { firstAppLoad, firstPageLoad } from "./firstPageLoad";
  import { SubAnimation, PropertyAnimation } from "$lib/animation-timing";

  export let title = "";
  export let childNumber = 0;
  export let preDelay = $firstAppLoad ? 0 : 100;
  export let maxWidth: string | undefined = undefined;
  export let fullHeight = false;
</script>

<div class="background"></div>

<section class="container">
  <RoundDef />

  <div class="border-container">
    <div class="border-box">
      <div class="blur"></div>
    </div>
  </div>
</section>

<style>
  div.background, section {
	--w: 800px;
	--h: 800px;
	position: absolute;
	width: var(--w);
	height: var(--h);
	box-sizing: border-box;
	margin: 50px;
	--cut: 100px;
	--cut-depth: calc(2 * var(--cut) * cos(45deg));
		--poly: polygon(
      0% var(--cut-depth),
      var(--cut-depth) 0%,
      100% 0%,
      100% calc(100% - var(--cut-depth)),
      calc(100% - var(--cut-depth)) 100%,
      0% 100%
    );;
}

div.blur {
  width: 100%;
    height: 100%;
    position: absolute;
}

div.blur::before {
		--offset: 0px;
		position: absolute;
		top: var(--offset); right: var(--offset); bottom: var(--offset); left: var(--offset);
		backdrop-filter: blur(9px);
		-webkit-clip-path: var(--poly);
						clip-path: var(--poly);
		content: ''
}

div.background {
	margin: 0;
	width: calc(var(--w) + 100px);
	height: calc(var(--h) + 100px);
	background: url(https://images.unsplash.com/photo-1544070078-a212eda27b49?ixlib=rb-1.2.1&q=85&fm=jpg&crop=entropy&cs=srgb&ixid=eyJhcHBfaWQiOjE0NTg5fQ) center/cover;
}
</style>

```

doesn't make a difference. We edit the Storybook iframe to have a `<head>` that only contains the exact styles above, and edit the Storybook `<body>` to only contain the exact HTML above, but the problem persists. It goes away when we edit the entire document's head and body that way. It appears that *something* about being put into the Storybook iframe breaks things in Chrome, even if it renders fine outside of it.

Further experimentation with the Tauri app reveals that this problem is only triggered when `border-radius` is combined with `overflow-y` or `overflow-x`. This particular combination does not appear to trigger any problems on the latest Chrome.

We finally realize while trying to mitigate this problem that the rounded corners on the main element have been painted over by the background canvas. To keep the fix simple, we edit `src-svelte\src\routes\BackgroundUI.svelte`:

```css
  .background {
    ...
    border-radius: var(--main-corners);
  }
```

Finally, we edit `src-svelte\src\routes\AppLayout.svelte` so that `border-radius` and `overflow` attributes are no longer on the same node, by moving the overflow attributes from `#main-container` (where they're actually accidentally duplicated) to `main`:

```css
  main {
    ...
    overflow-y: auto;
    overflow-x: hidden;
    ...
  }
```

In the course of manually testing all the different pages, we now notice that there's an errant scroll bar for some reason on the LLM API calls page. Upon further debugging, it appears to be yet another  instance of `clientHeight` rounding causing our calculations to be off. (The first instance was in the ["Initial implementation > Frontend"](/zamm-notes/api-calls-display.md) section.) We edit `src-svelte/src/lib/Scrollable.svelte`:

```ts
  export function resizeScrollable() {
    ...
    requestAnimationFrame(() => {
      if (!container || !scrollable) {
        ...
      }

      const newHeight = Math.floor(container.getBoundingClientRect().height);
      scrollableHeight = `${newHeight}px`;
      ...
    });
  }
```

### Supporting the background animation setting

We want to support having background animations turned off, but we also want to keep testing the background with Playwright screenshots. As such, we'll have to use a random seed for determinism, but it turns out that there's [no built-in support](https://stackoverflow.com/questions/521295/seeding-the-random-number-generator-in-javascript) for this. As such, we pick a [popular JS library](https://www.npmjs.com/package/pure-rand), and use the console to generate a random seed for its RNG. We use all the [available bits](https://stackoverflow.com/a/53842060) for the seed:

```js
Math.random() * Number.MAX_SAFE_INTEGER 
```

Then we edit `src-svelte/src/routes/BackgroundUI.svelte`:

```ts
  ...
  import prand from "pure-rand";

  const rng = prand.xoroshiro128plus(8650539321744612);
  ...
  const STATIC_INITIAL_DRAWS = 100;

  ...

  function nextColumnPosition() {
    return prand.unsafeUniformIntDistribution(-numRows, 0, rng);
  }

  function resizeCanvas() {
    ...

    dropsPosition = Array(numColumns).fill(0).map(nextColumnPosition);
    ...

    if (animated) {
      startAnimating();
    } else {
      for (let i = 0; i < STATIC_INITIAL_DRAWS; i++) {
        draw();
      }
    }
  }

  function draw() {
    ...
      if (dropsPosition[...] > ...) {
        dropsPosition[column] = nextColumnPosition();
      }
    ...
  }

  function updateAnimationState(nowAnimating: boolean) {
    if (nowAnimating) {
      startAnimating();
    } else {
      stopAnimating();
    }
  }

  ...

  $: updateAnimationState(animated);
```

And we edit `src-svelte/src/lib/__mocks__/MockFullPageLayout.svelte` to mock the new setting properly by importing `Background` instead of `BackgroundUi`:

```svelte
<script lang="ts">
  ...
  import Background from "../../routes/Background.svelte";
</script>

<div id="mock-full-page-layout" ...>
  <div class="bg">
    <Background />
  </div>
  ...
</div>
```

Now we can temporarily try this out in other stories.

When committing these changes, we get the error

```
[error] src-svelte/src/routes/BackgroundUI.svelte: Error: unknown node type: Script
[error]     at Object.print (/root/zamm/node_modules/prettier-plugin-svelte/plugin.js:1452:11)
[error]     at callPluginPrintFunction (/root/zamm/node_modules/prettier/index.js:8601:26)
[error]     at mainPrintInternal (/root/zamm/node_modules/prettier/index.js:8550:22)
```

We see that we've already encountered this problem before [here](/zamm-notes/chat.md), so we follow the solution there and do

```bash
$ yarn install --frozen-lockfile
```

again. It appears a more permanent solution would be to set constraints on the various versions in `package.json` themselves.

### Supporting animation speed setting

We edit `src-svelte/src/routes/BackgroundUI.svelte` again, simplifying the `ANIMATES_PER_CHAR` logic in the meantime:

```ts
  ...
  import { standardDuration } from "$lib/preferences";
  ...

  const ANIMATES_PER_CHAR = 2;
  ...
  $: animateIntervalMs = $standardDuration / 2;

  ...

  function startAnimating() {
    ...

    animateInterval = setInterval(draw, animateIntervalMs);
  }

  ...

  function updateAnimationSpeed(_newSpeed: number) {
    stopAnimating();
    startAnimating();
  }

  ...

  $: updateAnimationSpeed(animateIntervalMs);
```

### Adding support for disabling transparency

Because the blur effect doesn't work on certain versions of WebKit on Linux (which makes the foreground text that much harder to read), and because the text shadows render incorrectly on a transparent background on other versions of WebKit on Linux, we add a setting to disable transparency.

#### Frontend impermanent setting

We start on the frontend by adding an impermanent setting to `src-svelte/src/lib/preferences.ts`:

```ts
export const transparencyOn = writable(false);
```

We expose that in `src-svelte/src/routes/settings/Settings.svelte`:

```svelte
<script lang="ts">
  ...
  import {
    ...,
    transparencyOn,
    ...
  } from "$lib/preferences";

  ...
</script>

<InfoBox title="Settings">
  ...

  <div class="container">
    <SubInfoBox subheading="Other visual effects">
      <SettingsSwitch
        label="Transparency"
        bind:toggledOn={$transparencyOn}
      />
    </SubInfoBox>
  </div>

  ...
</div>
```

We then make use of this in `src-svelte/src/lib/InfoBox.svelte`:

```svelte
<script lang="ts">
  ...
  import { ..., transparencyOn } from "./preferences";
  ...
</script>

<section
  ...
>
  ...

  <div class="border-container">
    <div class="border-box" class:transparent={$transparencyOn} ...>
      ...
    </div>
    ...
  </div>
</section>

<style>
  ...

  .border-box .blur {
    display: none;
  }

  .border-box.transparent .blur {
    display: block;
  }

  ...

  .border-box .background {
    filter: url(#round) drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.4));
  }

  .border-box.transparent .background {
    filter: url(#round) drop-shadow(0px 1px 4px rgba(26, 58, 58, 0.8));
    opacity: 0.5;
  }

  ...
</style>
```

This toggle helps us realize that the transparency effect affects the info box shadow as well, so we up the shadow opacity to 0.8 when transparency is on to compensate.

We realize that it's possible to not generate null values for the Tauri API response by [marking](https://stackoverflow.com/a/53900684) the Serde fields as skipping serialization, so we do that refactor first by editing `src-tauri/src/commands/preferences/models.rs` as such to add the new tag in to every field:

```rs
#[derive(Debug, ...)]
pub struct Preferences {
    #[serde(skip_serializing_if = "Option::is_none")]
    animations_on: Option<bool>,
    #[serde(skip_serializing_if = "Option::is_none")]
    background_animation: Option<bool>,
    #[serde(skip_serializing_if = "Option::is_none")]
    animation_speed: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    sound_on: Option<bool>,
    #[serde(skip_serializing_if = "Option::is_none")]
    volume: Option<f64>,
}
```

`src-svelte/src/lib/bindings.ts` got updated automatically. As such, we then edit all the `get_preference` sample API calls, such as `src-tauri/api/sample-calls/get_preferences-animation-speed-override.yaml`, to match by not including undefined fields:

```yaml
request: ["get_preferences"]
response:
  message: >
    {
      "animation_speed": 0.9
    }

```

This means that `src-tauri/api/sample-calls/get_preferences-no-file.yaml` now looks like this:

```yaml
request: ["get_preferences"]
response:
  message: >
    {}
sideEffects:
  disk:
    startStateDirectory: empty
    endStateDirectory: empty

```

The AppLayout tests are now failing, so we edit `src-svelte/src/routes/AppLayout.svelte` to check instead that the values are [not nullish](https://stackoverflow.com/a/5515385) instead of merely exactly not null:

```ts
  onMount(async () => {
    ...
    if (prefs.sound_on != null) {
      ...
    }

    if (prefs.volume != null) {
      ...
    }

    if (prefs.animations_on != null) {
      ...
    }

    if (prefs.background_animation == null) {
      ...
    } else {
      ...
    }

    if (prefs.animation_speed != null) {
      ...
    }

    ...
  });
```

We go on to edit all of the `set_preference` API calls, such as `src-tauri/api/sample-calls/set_preferences-sound-on.yaml`, to not include `null`'s anymore:

```yaml
request:
  - set_preferences
  - >
    {
      "preferences": {
        "sound_on": true
      }
    }
response:
  message: "null"
sideEffects:
  disk:
    startStateDirectory: preferences/sound-off-extra-settings
    endStateDirectory: preferences/sound-on-extra-settings

```

The Settings tests fail now, so we edit `src-svelte/src/routes/settings/Settings.svelte` to remove all the instances of `...NullPreferences,` that add in the `null`'s, and we edit `src-svelte/src/lib/preferences.ts` to remove the now-unneeded `NullPreferences`.

Now we can cleanly add a new preference. We start by editing `src-tauri/src/commands/preferences/models.rs`:

```rs
#[derive(Debug, ...)]
pub struct Preferences {
    ...,
    #[serde(skip_serializing_if = "Option::is_none")]
    transparency_on: Option<bool>,
    ...
}
```

We create the sample preference file `src-tauri/api/sample-disk-writes/preferences/transparency-on/preferences.toml`:

```toml
transparency_on = true

```

and a similar one at `src-tauri/api/sample-disk-writes/preferences/transparency-off/preferences.toml` for the opposite condition.

We also create `src-tauri/api/sample-calls/get_preferences-transparency-on.yaml`:

```yaml
request: ["get_preferences"]
response:
  message: >
    {
      "transparency_on": true
    }
sideEffects:
  disk:
    startStateDirectory: preferences/transparency-on
    endStateDirectory: preferences/transparency-on

```

and `src-tauri/api/sample-calls/get_preferences-transparency-off.yaml`.

We can now test for this in `src-tauri/src/commands/preferences/read.rs`:

```rs
    #[tokio::test]
    async fn test_get_preferences_with_transparency_off() {
        check_get_preferences_sample(
            function_name!(),
            "./api/sample-calls/get_preferences-transparency-off.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_get_preferences_with_transparency_on() {
        check_get_preferences_sample(
            function_name!(),
            "./api/sample-calls/get_preferences-transparency-on.yaml",
        )
        .await;
    }
```

We should test the preference writing as well, so we create `src-tauri/api/sample-calls/set_preferences-transparency-on.yaml`:

```yaml
request:
  - set_preferences
  - >
    {
      "preferences": {
        "transparency_on": true
      }
    }
response:
  message: "null"
sideEffects:
  disk:
    endStateDirectory: preferences/transparency-on

```

and the corresponding `src-tauri/api/sample-calls/set_preferences-transparency-off.yaml`. We then check these new sample calls with new tests in `src-tauri/src/commands/preferences/write.rs`:

```rs
    #[tokio::test]
    async fn test_set_preferences_transparency_on() {
        check_set_preferences_sample(
            function_name!(),
            "./api/sample-calls/set_preferences-transparency-on.yaml",
        )
        .await;
    }

    #[tokio::test]
    async fn test_set_preferences_transparency_off() {
        check_set_preferences_sample(
            function_name!(),
            "./api/sample-calls/set_preferences-transparency-off.yaml",
        )
        .await;
    }
```

As usual, `src-svelte/src/lib/bindings.ts` gets updated automatically. We now check that these same API calls are consumed correctly on the frontend, first with preference reading at `src-svelte/src/routes/AppLayout.svelte`:

```ts
  ...
  import {
    ...,
    transparencyOn,
    ...
  } from "$lib/preferences";

  ...

  onMount(async () => {
    ...

    if (prefs.transparency_on != null) {
      transparencyOn.set(prefs.transparency_on);
    }

    ...
  });
```

We add a corresponding test to `src-svelte/src/routes/AppLayout.test.ts`:

```ts
...
import {
  ...,
  transparencyOn,
} from "$lib/preferences";
...

  test("will set transparency if transparency preference overridden", async () => {
    expect(get(transparencyOn)).toBe(false);
    expect(tauriInvokeMock).not.toHaveBeenCalled();

    playback.addSamples(
      "../src-tauri/api/sample-calls/get_preferences-transparency-on.yaml",
    );

    render(AppLayout, { currentRoute: "/" });
    await waitFor(() => {
      expect(get(transparencyOn)).toBe(true);
    });
    expect(tauriInvokeMock).toHaveReturnedTimes(1);
  });
```

Then, we implement preference writing at `src-svelte/src/routes/settings/Settings.svelte`:

```svelte
<script lang="ts">
  ...

  const onTransparencyToggle = (newValue: boolean) => {
    setPreferences({
      transparency_on: newValue,
    });
  };

  ...
</script>

<InfoBox title="Settings">
  ...

  <div class="container">
    <SubInfoBox subheading="Other visual effects">
      <SettingsSwitch label="Transparency" ... onToggle={onTransparencyToggle} />
    </SubInfoBox>
  </div>

  ...
</InfoBox>
```

We test this in `src-svelte/src/routes/settings/Settings.test.ts`, where we find that we have to reset the global stores after previous test cases run:

```ts
...
import { ..., transparencyOn } from "$lib/preferences";
...

describe("Settings", () => {
  ...

  beforeEach(() => {
    ...

    soundOn.set(true);
    volume.set(1);
    transparencyOn.set(false);
  });

  ...

  test("can toggle transparency on and off while saving setting", async () => {
    render(Settings, {});
    expect(get(transparencyOn)).toBe(false);
    expect(tauriInvokeMock).not.toHaveBeenCalled();

    const otherVisualsRegion = screen.getByRole("region", { name: "Other visual effects" });
    const transparencySwitch = getByLabelText(otherVisualsRegion, "Transparency");
    playback.addCalls(playSwitchSoundCall);
    playback.addSamples("../src-tauri/api/sample-calls/set_preferences-transparency-on.yaml");
    await act(() => userEvent.click(transparencySwitch));
    expect(get(transparencyOn)).toBe(true);
    expect(tauriInvokeMock).toHaveReturnedTimes(2);
    expect(playback.unmatchedCalls.length).toBe(0);

    playback.addCalls(playSwitchSoundCall);
    playback.addSamples("../src-tauri/api/sample-calls/set_preferences-transparency-off.yaml");
    await act(() => userEvent.click(transparencySwitch));
    expect(get(transparencyOn)).toBe(false);
    expect(tauriInvokeMock).toHaveReturnedTimes(4);
    expect(playback.unmatchedCalls.length).toBe(0);
  });
});

```

In the course of writing these tests, we ended up creating a typo when specifying the directories to read from. As such, we edit `src-tauri/src/test_helpers/api_testing.rs` to add clearer error messages to the unwrap:

```rs
fn copy_dir_all(...) -> io::Result<()> {
    fs::create_dir_all(&dst).unwrap_or_else(|_| panic!("Error creating directory at {:?}", dst.as_ref()));
    for entry in fs::read_dir(&src).unwrap_or_else(|_| panic!("Error reading from directory at {:?}", src.as_ref())) {
      ...
    }
    ...
}
```

#### Default transparency on Windows

We want to enable transparency by default, but only on Windows. We find out that we can indeed [selectively apply macros](https://users.rust-lang.org/t/how-do-i-apply-macros-only-on-release/48757).

At first we try `#[cfg_attr(target_os = "windows", serde(default = "Some(true)"))]`, but with that we get the error

```
error: failed to parse path: "Some(true)"
  --> src\commands\preferences\models.rs:23:55
   |
23 |     #[cfg_attr(target_os = "windows", serde(default = "Some(true)"))]
   |                                                       ^^^^^^^^^^^^
```

We see that the correct way to do it is actually [with a function call](https://stackoverflow.com/a/65973982). We finally get it compiling with

```rs
fn default_true() -> Option<bool> {
    Some(true)
}

#[derive(...)]
pub struct Preferences {
    ...
    #[serde(skip_serializing_if = "Option::is_none")]
    #[cfg_attr(target_os = "windows", serde(default = "default_true"))]
    transparency_on: Option<bool>,
    ...
}
```

but this also changes the output when writing the preferences file, which we don't want. As such, we edit only `src-tauri\src\commands\preferences\read.rs`. Note that we add a `#[allow(unused_mut)]` to avoid compilation warnings on Linux.

```rs
fn get_preferences_happy_path(
    ...
) -> ZammResult<Preferences> {
    ...
    #[allow(unused_mut)]
    let mut found_preferences = if preferences_path.exists() {
        ...
        preferences
    } else {
        ...
        Preferences::default()
    };
    #[cfg(target_os = "windows")]
    if (found_preferences.transparency_on.is_none()) {
        found_preferences.transparency_on = Some(true);
    }
    Ok(found_preferences)
}
```

To do this, we have to make the `transparency_on` public in `src-tauri\src\commands\preferences\models.rs`. If we do that, we might as well make it public for the rest of the fields:

```rs
#[derive(...)]
pub struct Preferences {
    #[serde(...)]
    pub animations_on: Option<bool>,
    #[serde(...)]
    pub background_animation: Option<bool>,
    #[serde(...)]
    pub animation_speed: Option<f64>,
    #[serde(...)]
    pub transparency_on: Option<bool>,
    #[serde(...)]
    pub sound_on: Option<bool>,
    #[serde(...)]
    pub volume: Option<f64>,
}
```

Some tests are failing now, as expected, so we edit `src-tauri\src\commands\preferences\read.rs` further:

```rs
#[cfg(test)]
mod tests {
    ...

    #[cfg(not(target_os = "windows"))]
    #[tokio::test]
    async fn test_get_preferences_without_file() {
        ...
    }

    #[cfg(target_os = "windows")]
    #[tokio::test]
    async fn test_get_preferences_without_file() {
        check_get_preferences_sample(
            function_name!(),
            "./api/sample-calls/get_preferences-no-file-windows.yaml",
        )
        .await;
    }

    #[cfg(not(target_os = "windows"))]
    #[tokio::test]
    async fn test_get_preferences_with_sound_override() {
        ...
    }

    #[cfg(not(target_os = "windows"))]
    #[tokio::test]
    async fn test_get_preferences_with_volume_override() {
        ...
    }

    ...

    #[cfg(not(target_os = "windows"))]
    #[tokio::test]
    async fn test_get_preferences_with_extra_settings() {
        ...
    }
}
```

We create a new file at `src-tauri/api/sample-calls/get_preferences-no-file-windows.yaml`, just for the API call return on Windows:

```rs
request: ["get_preferences"]
response:
  message: >
    {
      "transparency_on": true
    }
sideEffects:
  disk:
    startStateDirectory: empty
    endStateDirectory: empty

```

#### Removing transparency for default screenshot tests

We move our changes from `MockFullPageLayout` to `src-svelte\src\lib\__mocks__\MockPageTransitions.svelte` instead:

```svelte
<script lang="ts">
  ...
  import { ..., transparencyOn, backgroundAnimation } from "$lib/preferences";
  import Background from "../../routes/Background.svelte";

  ...
  transparencyOn.set(true);
  backgroundAnimation.set(true);
</script>

<div id="mock-transitions">
  <MockFullPageLayout>
    <div class="bg">
      <Background />
    </div>

  ...
</div>

<style>
  ...

  .bg {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: -1;
  }
</style>
```

This way, only the "full-page" stories will have transparency and background effects applied to them, whereas all other screenshots should remain the same.

#### Adding Chinese font for end-to-end tests

We find that the end-to-end tests are rendering squares in the background. We realize that we should add a Chinese font. We pick Zhi Mang Xing for its stylism:

```bash
$  yarn add @fontsource/zhi-mang-xing
```

We import it in `src-svelte/src/routes/styles.css`:

```css
@import "@fontsource/zhi-mang-xing";
```

We edit `src-svelte/src/routes/BackgroundUI.svelte` to convert all the text to Simplified Chinese, which the font is for, and to update the font family and font size:

```ts
  const CHAR_EM = 26;
  const CHAR_GAP = 2;
  ...
  const DDJ = [
    "道可道非常道",
    "名可名非常名",
    "无名天地之始",
    "有名万物之母",
    "故常无欲以观其妙",
    "常有欲以观其徼",
    "此两者同出而异名",
    "同谓之玄",
    "玄之又玄",
    "众妙之门",
  ];

  ...

  function draw() {
    ...
    ctx.font = CHAR_EM + "px 'Zhi Mang Xing', sans-serif";
    ...
  }

  ...
```

Unfortunately, sometimes the font does not load in time for the characters to be rendered for the Storybook screenshot.

It turns out that fonts generally [don't load](https://stackoverflow.com/a/52763507) until they're needed. There are ways to put text [inside a div](https://stackoverflow.com/a/29248183) to trigger the load, or to [base-64 encode](https://stackoverflow.com/a/23794798) the font, or to [preload it](https://fontsource.org/docs/getting-started/preload) with FontSource, but that last one appears as if it might be complicated with Vite's bundling. First, we'll try

```bash
$ yarn add fontfaceobserver
$ yarn add -D @types/fontfaceobserver
```

We then edit `src-svelte\src\routes\BackgroundUI.svelte`, noting that the animation speed update also triggers the animation even when it's turned off, so we add an extra guard there:

```ts
  ...
  import FontFaceObserver from "fontfaceobserver";
  
  ...
  const TEXT_FONT = CHAR_EM + "px 'Zhi Mang Xing', sans-serif";
  ...

  function ensureChineseFontLoaded(callback: () => void) {
    var zhiMangXing = new FontFaceObserver('Zhi Mang Xing');
    zhiMangXing.load("中文字").then(callback).catch(function (err) {
      console.warn("Could not load Chinese font for background animation:", err);
    });
  }

  function startAnimating() {
    if (!animated) { // this is possible from the animation speed change trigger
      console.warn("Animation not enabled");
      return;
    }

    ensureChineseFontLoaded(function () {
      if (animateInterval) {
        console.warn("Animation already running");
        return;
      }

      animateInterval = setInterval(draw, animateIntervalMs);
    });
  }

  ...

  function resizeCanvas() {
    ...

    if (animated) {
      ...
    } else {
      ensureChineseFontLoaded(function () {
        for (let i = 0; i < STATIC_INITIAL_DRAWS; i++) {
          draw();
        }
      });
    }
  }

  function draw() {
    ...

    ctx.font = TEXT_FONT;

    ...
  }
```

It is unclear if this is working completely, since the Playwright screenshots appear to render some characters not in that font. We try copying the font file to `src-svelte\static\public-fonts\zhi-mang-xing.ttf` and editing `src-svelte\src\routes\BackgroundUI.svelte` further:

```ts
  onMount(() => {
    const fontFile = new FontFace(
      "Zhi Mang Xing",
      "url(/public-fonts/zhi-mang-xing.ttf)",
    );
    document.fonts.add(fontFile);

    fontFile.load().then(
      () => {
        ...
      },
      (err) => {
        console.error(err);
      },
    );

    return () => {
      ...
    };
  });
```

Unfortunately, this now causes tests in `src-svelte\src\routes\AppLayout.test.ts` to fail:

```
 FAIL  src/routes/AppLayout.test.ts > AppLayout > will set transparency if transparency preference overridden
ReferenceError: FontFace is not defined
 ❯ src/routes/BackgroundUI.svelte:154:11
    152|
    153|   onMount(() => {
    154|     const fontFile = new FontFace(
       |           ^
    155|       "Zhi Mang Xing",
    156|       "url(/public-fonts/zhi-mang-xing.ttf)",
```

We mock `FontFace`, which leads us to

```
 FAIL  src/routes/AppLayout.test.ts > AppLayout > will set transparency if transparency preference overridden
TypeError: Cannot read properties of undefined (reading 'add')        
 ❯ src/routes/BackgroundUI.svelte:139:20
    137|       "url(/public-fonts/zhi-mang-xing.ttf)",
    138|     );
    139|     document.fonts.add(fontFile);
       |                    ^
    140|
    141|     fontFile.load().then(
```

As such, we use [this](https://stackoverflow.com/a/71545957) as a way to get the mock working:

```ts
vi.stubGlobal("FontFace", function () {
  return {
    load: () => Promise.resolve(),
  };
});
Object.defineProperty(document, "fonts", {
  value: {
    add: vi.fn(),
    load: vi.fn(),
  },
});
```

Now we get the error

```
Could not load Chinese font for background animation: TypeError: b.context.document.fonts.load is not a function
    at h (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\fontfaceobserver\fontfaceobserver.standalone.js:5:292)
    at C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\fontfaceobserver\fontfaceobserver.standalone.js:5:376
    ...
```

We remove the code we added previously to `src-svelte\src\routes\BackgroundUI.svelte` for this, as it appears the `fontFile.load()` fix works well. We also remove the dependencies `fontfaceobserver`, `@types/fontfaceobserver`, and `@fontsource/zhi-mang-xing` via `yarn remove`, and remove the import from `src-svelte\src\routes\styles.css`. Now vitest on Windows successfully runs that test file, but then exits ungracefully at the end with

```
error Command failed with exit code 3221225477.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

#### Addressing segfault

It appears that this is a [Windows-specific problem](https://stackoverflow.com/a/58959576) with NodeJS, but as it turns out, NodeJS on Linux exits too with a segfault with exit code 139. It appears that this has no effect on CI tests. Upgrading to Node v22 on Windows fixes this issue.

After a while, even CI tests start failing in this manner with the message

```
Segmentation fault (core dumped)
error Command failed with exit code 139.
```

Ironically, we are now unable to reproduce this on the Linux machine that is still running Node v20.5.1. We see from [this answer](https://stackoverflow.com/a/33849752) that we can go [here](https://nodejs.org/dist/v20.5.1/) to try to download the same version of NodeJS to install on Windows once again to try to reproduce the issue. We are able to reproduce the issue on Windows, but it does not appear easy to avoid the crash on this older version of NodeJS while still getting this test to pass. Even installing the latest version of the Node v20 series, `v20.13.1`, doesn't help with the crash on Windows. We try updating to use Node `22.2.0`. We now get a whole bunch of errors during the build, such as

```
60.16 ../../nan/nan.h: In function 'void Nan::SetAccessor(v8::Local<v8::ObjectTemplate>, v8::Local<v8::String>, Nan::GetterCallback, Nan::SetterCallback, v8::Local<v8::Value>, v8::AccessControl, v8::PropertyAttribute, Nan::imp::Sig)':
60.16 ../../nan/nan.h:2558:3: error: no matching function for call to 'v8::ObjectTemplate::SetAccessor(v8::Local<v8::String>&, void (*&)(v8::Local<v8::Name>, const v8::PropertyCallbackInfo<v8::Value>&), void (*&)(v8::Local<v8::Name>, v8::Local<v8::Value>, const v8::PropertyCallbackInfo<void>&), v8::Local<v8::Object>&, v8::AccessControl&, v8::PropertyAttribute&)'
60.16  2558 |   );
60.16       |   ^
60.16 In file included from /root/.cache/node-gyp/22.2.0/include/node/v8-function.h:15,
60.16                  from /root/.cache/node-gyp/22.2.0/include/node/v8.h:33,
60.16                  from /root/.cache/node-gyp/22.2.0/include/node/node.h:73,
60.16                  from ../../nan/nan.h:62,
60.16                  from ../src/backend/Backend.h:6,
60.16                  from ../src/backend/Backend.cc:1:
60.16 /root/.cache/node-gyp/22.2.0/include/node/v8-template.h:1049:8: note: candidate: 'void v8::ObjectTemplate::SetAccessor(v8::Local<v8::String>, v8::AccessorGetterCallback, v8::AccessorSetterCallback, v8::Local<v8::Value>, v8::PropertyAttribute, v8::SideEffectType, v8::SideEffectType)'
```

which according to [this answer](https://stackoverflow.com/q/64316140) might mean that package upgrades are required. We try `yarn upgrade` in the root directory, and then the build succeeds locally. On CI, however, we run into errors like this for the steps that don't use the Docker image:

```
make: Entering directory '/home/runner/work/zamm/zamm/node_modules/canvas/build'
  SOLINK_MODULE(target) Release/obj.target/canvas-postbuild.node
  COPY Release/canvas-postbuild.node
  CXX(target) Release/obj.target/canvas/src/backend/Backend.o
  CXX(target) Release/obj.target/canvas/src/backend/ImageBackend.o
  CXX(target) Release/obj.target/canvas/src/backend/PdfBackend.o
  CXX(target) Release/obj.target/canvas/src/backend/SvgBackend.o
  CXX(target) Release/obj.target/canvas/src/bmp/BMPParser.o
  CXX(target) Release/obj.target/canvas/src/Backends.o
  CXX(target) Release/obj.target/canvas/src/Canvas.o
  CXX(target) Release/obj.target/canvas/src/CanvasGradient.o
  CXX(target) Release/obj.target/canvas/src/CanvasPattern.o
In file included from ../src/CanvasPattern.cc:6:
../src/Image.h:18:10: fatal error: gif_lib.h: No such file or directory
   18 | #include <gif_lib.h>
      |          ^~~~~~~~~~~
compilation terminated.
make: *** [canvas.target.mk:161: Release/obj.target/canvas/src/CanvasPattern.o] Error 1
make: Leaving directory '/home/runner/work/zamm/zamm/node_modules/canvas/build'
gyp ERR! build error 
```

[It appears](https://github.com/dominictarr/feedopensource/issues/25#issue-25625002) that we might just have to install `libgif-dev`. We edit `.github/workflows/tests.yaml`:

```yaml
...

jobs:
  ...
  pre-commit:
    ...
    steps:
      ...
      - name: Install main Node dependencies
        run: |
          sudo apt install -y libgif-dev
          yarn install --frozen-lockfile
      ...
  svelte:
    ...
    steps:
      ...
      - name: Install Node dependencies
        run: |
          sudo apt install -y libgif-dev
          ...
  e2e:
    ...
    steps:
      ...
      - name: Install Node dependencies
        run: |
          sudo apt install -y libgif-dev
          ...
```

Unfortunately, the same exit code 139 is still occurring on CI, with failing screenshot and end-to-end tests to boot. Since we've already done the work to upgrade to the latest Node, we might as well keep that progress and just update the screenshots. However, the problem with Node crashing after all tests run still needs to be solved.

We see that there have [been](https://github.com/vitest-dev/vitest/issues/3143) past [issues](https://github.com/vitest-dev/vitest/issues/317) with Vitest segfaults, but these were all pre-1.0.0 release. Nonetheless, we try out the suggestion to use `--pool=forks` instead (the suggestion to use `--no-threads` is [outdated](https://github.com/vitest-dev/vitest/issues/4620#issuecomment-1832683849)). Unfortunately, this breaks the slider, switch, and API calls tests -- basically any test that makes use of Playwright, except surprisingly for the Storybook tests. It is unclear why this would affect anything anyways, given that we have already edited `src-svelte\vitest.config.ts` to enforce `singleThread: true`.

Running locally on Linux with

```bash
$ yarn vitest --pool=forks --exclude src/routes/storybook.test.ts run
```

to avoid expensive Storybook tests does not reproduce the issue. Running that locally on Windows does reproduce the timeout issue, but it seems only intermittently so.

Reinstalling Node 22 on Windows fails with

```
error C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\canvas: Command failed.
Exit code: 7
Command: node-pre-gyp install --fallback-to-build --update-binary
Arguments:
Directory: C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\canvas
Output:
node-pre-gyp info it worked if it ends with ok
node-pre-gyp info using node-pre-gyp@1.0.11
node-pre-gyp info using node@22.2.0 | win32 | x64
(node:18996) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
node-pre-gyp http GET https://github.com/Automattic/node-canvas/releases/download/v2.11.2/canvas-v2.11.2-node-v127-win32-unknown-x64.tar.gz
node-pre-gyp ERR! install response status 404 Not Found on https://github.com/Automattic/node-canvas/releases/download/v2.11.2/canvas-v2.11.2-node-v127-win32-unknown-x64.tar.gz
node-pre-gyp WARN Pre-built binaries not installable for canvas@2.11.2 and node@22.2.0 (node-v127 ABI, unknown) (falling back to source compile with node-gyp)
node-pre-gyp WARN Hit error response status 404 Not Found on https://github.com/Automattic/node-canvas/releases/download/v2.11.2/canvas-v2.11.2-node-v127-win32-unknown-x64.tar.gz
node-pre-gyp ERR! UNCAUGHT EXCEPTION
node-pre-gyp ERR! stack Error: spawn EINVAL
node-pre-gyp ERR! stack     at ChildProcess.spawn (node:internal/child_process:421:11)
node-pre-gyp ERR! stack     at Object.spawn (node:child_process:760:9)
node-pre-gyp ERR! stack     at module.exports.run_gyp (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\@mapbox\node-pre-gyp\lib\util\compile.js:80:18)
node-pre-gyp ERR! stack     at build (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\@mapbox\node-pre-gyp\lib\build.js:41:13)
node-pre-gyp ERR! stack     at self.commands.<computed> [as build] (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\@mapbox\node-pre-gyp\lib\node-pre-gyp.js:86:37)
node-pre-gyp ERR! stack     at run (C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\@mapbox\node-pre-gyp\lib\main.js:81:30)
node-pre-gyp ERR! stack     at process.processTicksAndRejections (node:internal/process/task_queues:77:11)
node-pre-gyp ERR! System Windows_NT 10.0.22631
node-pre-gyp ERR! command "C:\\Program Files\\nodejs\\node.exe" "C:\\Users\\Amos Ng\\Documents\\projects\\zamm-dev\\zamm\\node_modules\\@mapbox\\node-pre-gyp\\bin\\node-pre-gyp" "install" "--fallback-to-build" "--update-binary"
```

It appears that there are [no prebuilt binaries](https://stackoverflow.com/a/73097307) available for NodeJS v22 yet, as seen on the [canvas v2.11.2 release page](https://github.com/Automattic/node-canvas/releases/tag/v2.11.2). However, the pre-existing installed packages for Node 20 work fine.

We try to see if we are able to reproduce the segfault inside of Docker, in which case we can either try to fix it or if not, to use the Docker image on CI tests. We take this opportunity to edit `Dockerfile` to point to the new repo location:

```Dockerfile
...
LABEL org.opencontainers.image.source="https://github.com/zamm-dev/zamm"

...

# dev dependencies
RUN yarn playwright install --with-deps && \
  apt install -y imagemagick
```

We edit `Makefile` as well to reflect this new tag:

```Makefile
BUILD_IMAGE = ghcr.io/zamm-dev/zamm:v0.1.4-build
```

and for good measure update the repo location in `src-tauri/Cargo.toml`:

```toml
repository = "https://github.com/zamm-dev/zamm"
```

We edit `src-svelte/Makefile` to give ourselves an easy way to run quick tests on Vitest:

```Makefile
quick-test:
	yarn vitest --exclude src/routes/storybook.test.ts run
```

and finally edit `.github/workflows/tests.yaml` to try using the new image on CI for frontend tests as well. We remove the "Install remaining dependencies" step for the Svelte tests here because they should be covered by the Docker build step:

```yaml
...

jobs:
  build:
    ...
    container:
      image: "ghcr.io/zamm-dev/zamm:v0.1.4-build"
      ...
    ...
  svelte:
    ...
    container:
      image: "ghcr.io/zamm-dev/zamm:v0.1.4-build"
      options: --user root
    ...
    steps:
      ...
      - name: Run Svelte Tests
        run: yarn workspace gui test
```

It turns out that we still need the install Playwright step because the `/github/home/` path is used instead:

```
 FAIL  src/lib/Slider.playwright.test.ts > Slider drag test
Error: browserType.launch: Executable doesn't exist at /github/home/.cache/ms-playwright/chromium-1097/chrome-linux/chrome
╔═════════════════════════════════════════════════════════════════════════╗
║ Looks like Playwright Test or Playwright was just installed or updated. ║
║ Please run the following command to download new browsers:              ║
║                                                                         ║
║     yarn playwright install                                             ║
║                                                                         ║
║ <3 Playwright Team                                                      ║
╚═════════════════════════════════════════════════════════════════════════╝
```

Instead of needing to download it again, we just copy it over:

```yaml
      - name: Copy over playwright binaries
        run: |
          mkdir -p /github/home/.cache/
          cp -R /root/.cache/ms-playwright /github/home/.cache/ms-playwright
```

When we finally run the tests locally, we run into:

```
 FAIL  src/routes/api-calls/ApiCalls.playwright.test.ts > Api Calls endless scroll test > loads more messages when scrolled to end
Error: page.goto: net::ERR_CONNECTION_REFUSED at http://localhost:6006/?path=/story/screens-llm-call-list--full
Call log:
  - navigating to "http://localhost:6006/?path=/story/screens-llm-call-list--full", waiting until "load"

 ❯ getScrollElement src/routes/api-calls/ApiCalls.playwright.test.ts:38:16
     36|   const getScrollElement = async () => {
     37|     const url = `http://localhost:6006/?path=/story/screens-llm-call-list--full`;
     38|     await page.goto(url);
       |                ^
     39|     await page.locator("button[title='Hide addons [A]']").click();
     40|
 ❯ test.retry src/routes/api-calls/ApiCalls.playwright.test.ts:66:47
```

This appears to only be a fluke due to a slow startup with Storybook. The segfault turns out to sometimes happen and sometimes not inside the Docker image. A repeatable command that's usable even after `make clean` is

```bash
$ docker run --rm -v .:/zamm -w /zamm ghcr.io/zamm-dev/zamm:v0.1.4-build bash -c "yarn install --frozen-lockfile && cd src-svelte && make quick-test"
```

We find that using `--pool=forks` during testing does in fact fix the segfault problem in the Docker container locally, so we edit `src-svelte/Makefile`:

```Makefile
quick-test:
	yarn vitest --pool=forks --exclude src/routes/storybook.test.ts run

```

and `src-svelte/package.json`:

```json
{
  ...,
  "scripts": {
    ...,
    "test": "vitest --pool=forks run",
    ...
  },
  ...
}
```

The slider tests fail on CI due to timeouts, but at least the screenshot tests function fine as usual. We notice that the first test sometimes fail locally as well, but succeeds on the retry. As such, we edit `src-svelte/src/lib/test-helpers.ts` to see if warming up the Storybook server helps:

```ts
async function startStorybook(): Promise<ChildProcess> {
  return new Promise((resolve, reject) => {
    ...
    storybookProcess.stdout.on("data", (data) => {
      ...
      if (storybookStartupMessage.test(...)) {
        fetch("http://localhost:6006")
          .then(() => {
            resolve(storybookProcess);
          })
          .catch(reject);
      }
    });
    ...
  });
}
```

This fixes the timeout flukes locally, but not for CI. We try changing the tests to trigger Playwright timeouts instead of Vitest timeouts to see what is taking so long, and edit `src-svelte/vitest.config.ts` to extend default test timeouts:

```ts
export default defineConfig({
  ...
  test: {
    ...,
    testTimeout: 20_000,
    ...
  },
});
```

We also add `context.setDefaultTimeout(9_000);` to `src-svelte/src/lib/Slider.playwright.test.ts`, `src-svelte/src/lib/Switch.playwright.test.ts`, and `src-svelte/src/routes/api-calls/ApiCalls.playwright.test.ts`. This finally does get the tests to pass on CI. However, are other non-Playwright tests that do a `waitFor` that shouldn't take so long to fail, so we remove our changes to the global test config at `src-svelte/vitest.config.ts` and instead add proper constants to `src-svelte/src/lib/test-helpers.ts`:

```ts
export const PLAYWRIGHT_TIMEOUT = 9_000;
export const PLAYWRIGHT_TEST_TIMEOUT = 20_000;
```

We then do an import in each of the above `test.ts` files:

```ts
import { PLAYWRIGHT_TIMEOUT, PLAYWRIGHT_TEST_TIMEOUT } from "$lib/test-helpers.ts";
```

and set `context.setDefaultTimeout(PLAYWRIGHT_TIMEOUT);` in the test setup and `timeout: PLAYWRIGHT_TEST_TIMEOUT` after each test. We get the Svelte error

```
/root/zamm/src-svelte/src/routes/api-calls/ApiCalls.playwright.test.ts:14:8
Error: An import path can only end with a '.ts' extension when 'allowImportingTsExtensions' is enabled.
  PLAYWRIGHT_TEST_TIMEOUT,
} from "$lib/test-helpers.ts";
```

so we do

```ts
import { PLAYWRIGHT_TIMEOUT, PLAYWRIGHT_TEST_TIMEOUT } from "$lib/test-helpers";
```

instead. We decide to do this refactor in `src-svelte/src/routes/storybook.test.ts` as well, and so we end up changing `src-svelte/src/lib/test-helpers.ts` with the Storybook logic:

```ts
export const PLAYWRIGHT_TIMEOUT =
  process.env.PLAYWRIGHT_TIMEOUT === undefined
    ? PLAYWRIGHT_TIMEOUT
    : parseInt(process.env.PLAYWRIGHT_TIMEOUT);
export const PLAYWRIGHT_TEST_TIMEOUT = 2.2 * PLAYWRIGHT_TIMEOUT;
```

We als notice that in `.github\workflows\tests.yaml`, there are the lines

```yaml
    env:
      PLAYWRIGHT_TIMEOUT: 60000
```

We try setting `PLAYWRIGHT_TIMEOUT` to 9,000 instead, but that results in a lot of timeouts on GitHub CI. As such, we leave it at the original values for now.

##### API calls CI test failure

Now we run into test flakiness for `src-svelte/src/routes/api-calls/ApiCalls.playwright.test.ts`, but only on CI:

```
 ❯ src/routes/api-calls/ApiCalls.playwright.test.ts  (1 test | 1 failed) 140215ms
   ❯ src/routes/api-calls/ApiCalls.playwright.test.ts > Api Calls endless scroll test > loads more messages when scrolled to end
     → Timed out 5000ms waiting for expect(locator).toHaveText(expected)

Locator: locator('.scroll-contents').locator('a:nth-last-child(2) .message.instance .text-container')
Expected string: "Mocking number 10."
Received string: ""
Call log:
  - locator._expect with timeout 5000ms
  - waiting for locator('.scroll-contents').locator('a:nth-last-child(2) .message.instance .text-container')

     → locator.click: Timeout 60000ms exceeded.
Call log:
  - waiting for locator('button[title=\'Hide addons [A]\']')
  -   locator resolved to <button type="button" class="css-17dxjer" title="Hide ad…>…</button>
  - attempting click action
  -   waiting for element to be visible, enabled and stable
  -   element is visible, enabled and stable
  -   scrolling into view if needed
  -   done scrolling
  -   <div class="css-mzrf51">…</div> from <div class="css-sqdry3">…</div> subtree intercepts pointer events
  - retrying click action, attempt #1
  -   waiting for element to be visible, enabled and stable
  -   element is visible, enabled and stable
  -   scrolling into view if needed
  -   done scrolling
  -   <iframe allowfullscreen="" class="css-1e296jv" data-is-l…></iframe> from <div class="css-sqdry3">…</div> subtree intercepts pointer events
  - retrying click action, attempt #2
  -   waiting 20ms
  -   waiting for element to be visible, enabled and stable
  -   element is visible, enabled and stable
  -   scrolling into view if needed
  -   done scrolling
  -   <iframe allowfullscreen="" class="css-1e296jv" data-is-l…></iframe> from <div class="css-sqdry3">…</div> subtree intercepts pointer events
...
  - retrying click action, attempt #108
  -   waiting 500ms
  -   waiting for element to be visible, enabled and stable
  -   element is visible, enabled and stable
  -   scrolling into view if needed
  -   done scrolling
  -   <div class="css-1ki11f7" id="storybook-preview-wrappe…>…</div> from <div class="css-sqdry3">…</div> subtree intercepts pointer events
  - retrying click action, attempt #109
  -   waiting 500ms

 ❯ getScrollElement src/routes/api-calls/ApiCalls.playwright.test.ts:41:59
     39|     const url = `http://localhost:6006/?path=/story/screens-llm-call-l…
     40|     await page.goto(url);
     41|     await page.locator("button[title='Hide addons [A]']").click();
       |                                                           ^
     42| 
     43|     const maybeFrame = page.frame({ name: "storybook-preview-iframe" }…
 ❯ test.retry src/routes/api-calls/ApiCalls.playwright.test.ts:68:41
```

This is not reproducible in the local Docker image. It looks like the `#storybook-preview-wrapper` element that is the parent of the `iframe` is preventing it from being clicked. The rendered HTML for the page looks like this:

```
<div id="storybook-preview-wrapper" class="css-1ki11f7">
  <a tabindex="0" href="#reusable-external-link--external-link" class="css-1bi9mm9">
    Skip to sidebar
  </a>
  <iframe data-is-storybook="true" id="storybook-preview-iframe" title="storybook-preview-iframe" src="iframe.html?viewMode=story&amp;id=*" allow="clipboard-write;" allowfullscreen="" class="css-1e296jv" data-is-loaded="true">
  </iframe>
</div>
```

From [this issue](https://github.com/microsoft/playwright/issues/13576#issuecomment-1101468115), it appears that we can try working around this by editing `src-svelte\src\routes\api-calls\ApiCalls.playwright.test.ts`:

```ts
    await page.goto(url);
    await page.locator("button[title='Hide addons [A]']").dispatchEvent("click");
```

#### Scrollbar fix on main content

We discover that the scrollbar is now appearing in the middle of the screen, because we've changed the scroll to be on the `main` element instead, which is centered in the middle. To fix this, we edit `src-svelte\src\routes\AppLayout.svelte` again:

```svelte
<div id="app">
  <AnimationControl>
    ...

    <div id="main-container">
      ...

      <div class="content-container">
        <main>
          ...
        </main>
      </div>
    </div>
  </AnimationControl>
</div>

<style>
  ...

  .content-container {
    width: 100%;
    height: 100%;
    position: relative;
    overflow-y: auto;
    overflow-x: hidden;
  }

  main {
    position: relative;
    max-width: 70rem;
    margin: 0 auto;
  }
</style>
```

We wrap `main` in another div that handles all the overflow, but it turns out the scrollbars are being drawn over by the background. As such, we set `position: relative;`, which appears to trigger the `z-index` such that the scrollbar gets drawn over the animated background once more.

#### Storybook test

We edit `src-svelte\src\lib\InfoBox.stories.ts` to add a new story demonstrating the InfoBox with transparency on:

```ts
...
import MockPageTransitions from "./__mocks__/MockPageTransitions.svelte";

...

export const Transparent: StoryObj = Template.bind({}) as any;
Transparent.args = {
  title: "Simulation",
  maxWidth: "50rem",
};
Transparent.parameters = {
  preferences: {
    animationsOn: false,
    backgroundAnimation: false,
    transparencyOn: true,
  },
};
Transparent.decorators = [
  SvelteStoresDecorator,
  (story: StoryFn) => {
    return {
      Component: MockPageTransitions,
      slot: story,
    };
  },
];

```

We realize that `src-svelte\src\lib\__mocks__\MockPageTransitions.svelte` is overridding the `backgroundAnimation` setting on mount, so we remove that from that Svelte component.

Finally, we edit `src-svelte\src\routes\storybook.test.ts`:

```ts
import {
  ...,
  firefox,
  ...
} from "@playwright/test";

...

const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["reusable", "infobox"],
    variants: [..., "transparent"],
    screenshotEntireBody: true,
  },
  ...
];

...

describe.concurrent("Storybook visual tests", () => {
  ...

  beforeAll(async () => {
    browser = await firefox.launch({ headless: false });
    console.log(`Running tests with Firefox version ${browser.version()}`);

  ...
});
```

Unfortunately, it turns out that even though Firefox is clearly correctly rendering the blur in headed mode, the screenshot captured does not include the blur. However, it works with Chromium. As such, we'll have to add an extra option for the browser.

First, we edit `src-svelte/src/lib/InfoBoxView.svelte` to have a container that includes the InfoBox shadow:

```svelte
<div class="screenshot-container" ...>
  <InfoBox ...>
    ...
  </InfoBox>
</div>

<style>
  .screenshot-container {
    padding: 1rem;
    margin: -1rem;
  }
</style>

```

We then edit `src-svelte\src\routes\storybook.test.ts`:

```ts
import {
  ...
  chromium,
  webkit,
  ...
} from "@playwright/test";

...

interface VariantConfig {
  ...
  browser?: "webkit" | "chromium";
  selector?: string;
  ...
}

const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["reusable", "infobox"],
    variants: [
      ...,
      {
        name: "transparent",
        browser: "chromium",
        selector: ".screenshot-container",
      },
    ],
    screenshotEntireBody: true,
  },
  ...
];

...

describe.concurrent("Storybook visual tests", () => {
  let webkitBrowser: Browser;
  let webkitBrowserContext: BrowserContext;
  let chromiumBrowser: Browser;
  let chromiumBrowserContext: BrowserContext;

  beforeAll(async () => {
    webkitBrowser = await webkit.launch({ headless: true });
    webkitBrowserContext = await webkitBrowser.newContext();
    webkitBrowserContext.setDefaultTimeout(DEFAULT_TIMEOUT);

    chromiumBrowser = await chromium.launch({ headless: true });
    chromiumBrowserContext = await chromiumBrowser.newContext();
    chromiumBrowserContext.setDefaultTimeout(DEFAULT_TIMEOUT);

    console.log(`Running tests with Webkit v${webkitBrowser.version()} and ` + `Chromium v${chromiumBrowser.version()}`);

    ...
  });

  afterAll(async () => {
    await webkitBrowserContext.close();
    await webkitBrowser.close();
    await chromiumBrowserContext.close();
    await chromiumBrowser.close();
  });

  beforeEach(async (context) => {
    context.expect.extend(...);
  });

  const takeScreenshot = async (
    frame: Frame,
    page: Page,
    resizeWindow: boolean,
    selector?: string,
    screenshotEntireBody?: boolean,
  ) => {
    let locatorStr;
    if (selector) {
      locatorStr = selector;
    } else {
      locatorStr = screenshotEntireBody
        ? ...
        : ...;
      ...
    }

    ...
  }

  ...
      test(
        `${testName} should render the same`,
        async ({ expect }) => {
          const page = variantConfig.browser === "chromium" ? await chromiumBrowserContext.newPage() : await webkitBrowserContext.newPage();

          ...

          const screenshot = await takeScreenshot(
            ...,
            variantConfig.selector,
            ...
          );

          ...

          if (variantConfig.assertDynamic !== undefined) {
            ...
            const newScreenshot = await takeScreenshot(
              ...,
              variantConfig.selector,
              ...,
            );

            ...
          }

          await page.close();
        },
        {
          retry: 1,
          ...
        },
      );
  ...
};
```

Note that we no longer set the `page` in `beforeEach` because we don't know the browser beforehand, which means we no longer need `StorybookTestContext`. We also manually do the `afterEach` action with `page.close` at the end of the test.

Finally, we commit the new screenshot at `src-svelte\screenshots\baseline\reusable\infobox\transparent.png`.
