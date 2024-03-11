# Credits page

## Adding an icon

We edit `src-svelte/src/routes/SidebarUI.svelte` to add a new entry for a link to the credits page on the sidebar, colored in pink:

```svelte
<script lang="ts">
  ...
  import IconHeartFill from "~icons/ph/heart-fill";
  ...

  const routes: App.Route[] = [
    ...,
    {
      name: "Credits",
      path: "/credits",
      icon: IconHeartFill,
    }
  ];

  ...
</script>

...
  <nav>
    <div class="indicator" ...></div>
    {#each routes as route}
      <a
        ...
        class:icon={true}
        class={route.name.toLowerCase()}
        ...
      >
        ...
      </a>
    {/each}
  </nav>
...

<style>
  ...
  .icon[aria-current="page"].credits > :global(:only-child) {
    color: #FF1A40;
    filter: url(#inset-shadow);
  }
  ...
</style>
```

We demonstrate this new icon in `src-svelte/src/routes/SidebarUI.stories.ts`:

```ts
...

export const CreditsSelected: StoryObj = Template.bind({}) as any;
CreditsSelected.args = {
  currentRoute: "/credits",
  dummyLinks: true,
};

...
```

and add it to `src-svelte/src/routes/storybook.test.ts` for screenshot testing:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["layout", "sidebar"],
    variants: [..., "credits-selected"],
  },
  ...
];
```

We check that the new screenshots look right, and update the gold set with them.

## Setting up the page

We create a new blank page at `src-svelte/src/routes/credits/Credits.svelte`, based on the settings page:

```svelte
<script lang="ts">
  import InfoBox from "$lib/InfoBox.svelte";
  import SubInfoBox from "$lib/SubInfoBox.svelte";
</script>

<InfoBox title="Credits">
  <div class="container">
    <SubInfoBox subheading="Zen and the Automation of Metaprogramming for the Masses">
      Hello
    </SubInfoBox>
  </div>
</InfoBox>

<style>
  .container {
    margin-top: 1rem;
  }

  .container :global(.sub-info-box .content) {
    --side-padding: 0.8rem;
    display: grid;
    grid-template-columns: 1fr;
    gap: 0.1rem;
    margin: 0.5rem calc(-1 * var(--side-padding));
  }

  /* this takes sidebar width into account */
  @media (min-width: 52rem) {
    .container :global(.sub-info-box .content) {
      grid-template-columns: 1fr 1fr;
    }
  }
</style>

```

We add stories for this at `src-svelte/src/routes/credits/Credits.stories.ts`:

```ts
import CreditsComponent from "./Credits.svelte";
import MockPageTransitions from "$lib/__mocks__/MockPageTransitions.svelte";
import type { StoryObj, StoryFn } from "@storybook/svelte";

export default {
  component: CreditsComponent,
  title: "Screens/Credits",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: CreditsComponent,
  props: args,
});

export const Tablet: StoryObj = Template.bind({}) as any;
Tablet.parameters = {
  viewport: {
    defaultViewport: "tablet",
  },
};

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

We define the page itself at `src-svelte/src/routes/credits/+page.svelte`:

```svelte
<script lang="ts">
  import Credits from "./Credits.svelte";
</script>

<Credits />

```

Next, we define a Creditor component (not a Contributor because we want to thank the maintainers of our dependencies as well) at `src-svelte/src/routes/credits/Creditor.svelte`:

```svelte
<script lang="ts">
  export let name: string;
  export let url: string;
  export let urlDisplay = formatUrl(url);

  function formatUrl(url: string) {
    if (url.startsWith("https://")) {
      return url.slice(8);
    }
    if (url.startsWith("http://")) {
      return url.slice(7);
    }
    return url;
  }
</script>

<div class="creditor">
  <h4>{name}</h4>
  <a href={url} target="_blank" rel="noopener noreferrer">
    {urlDisplay}
  </a>
</div>

<style>
  .creditor {
    padding: 1rem;
  }

  h4 {
    font-weight: normal;
    margin: 0;
  }
</style>

```

We define a story for this at `src-svelte/src/routes/credits/Creditor.stories.ts`:

```ts
import CreditorComponent from "./Creditor.svelte";
import type { StoryObj, StoryFn } from "@storybook/svelte";

export default {
  component: CreditorComponent,
  title: "Screens/Credits/Creditor",
  argTypes: {},
};

const Template = ({ ...args }) => ({
  Component: CreditorComponent,
  props: args,
});

export const GithubContributor: StoryObj = Template.bind({}) as any;
GithubContributor.args = {
  name: "Amos Jun-yeung Ng",
  url: "https://github.com/amosjyng/",
};

```

and make use of it in `src-svelte/src/routes/credits/Credits.svelte`:

```svelte
<script lang="ts">
  ...
  import Creditor from "./Creditor.svelte";
</script>

...
    <SubInfoBox
      subheading="Zen ..."
    >
      <Creditor name="Amos Jun-yeung Ng" url="https://github.com/amosjyng/" />
    </SubInfoBox>
...

<style>
  ...

  .container :global(.sub-info-box .content) {
    ...
    margin: 0;
  }

  ...
</style>
```

Next, to extract the GitHub username, we first refactor `src-svelte/src/routes/credits/Creditor.svelte` to move `formatUrl` into the module scope so that it can be tested directly in a test, and then we modify it to extract GitHub usernames:

```svelte
<script lang="ts" context="module">
  export function formatUrl(url: string) {
    if (url.startsWith("https://github.com/")) {
      return url.slice(19, -1);
    }
    ...
  }
</script>
```

We test this new functionality in `src-svelte/src/routes/credits/Creditor.test.ts`:

```ts
import { expect, test } from "vitest";
import { formatUrl } from './Creditor.svelte';

describe("URL formatter", () => {
  test("formats HTTP(S) URL correctly", () => {
    expect(formatUrl("http://yahoo.com")).toEqual("yahoo.com");
    expect(formatUrl("https://google.com")).toEqual("google.com");
  });

  test("formats Github URLs correctly", () => {
    expect(formatUrl("https://github.com/amosjyng/")).toEqual("amosjyng");
  });
});

```

Next, we want to display the GitHub logo if the credit goes towards a GitHub user or project. We download the [logo SVG](https://github.com/logos) from GitHub, run it through a [URL encoder](https://yoksel.github.io/url-encoder/), and then [set the width](https://css-tricks.com/scale-svg/) or height on it. We do all this in `src-svelte/src/routes/credits/GitHubIcon.svelte`:

```svelte
<script lang="ts">
  export let size = "18px";
</script>

<img
  src="data:image/svg+xml,%3Csvg width='98' height='96' xmlns='http://www.w3.org/2000/svg'%3E%3Cpath fill-rule='evenodd' clip-rule='evenodd' d='M48.854 0C21.839 0 0 22 0 49.217c0 21.756 13.993 40.172 33.405 46.69 2.427.49 3.316-1.059 3.316-2.362 0-1.141-.08-5.052-.08-9.127-13.59 2.934-16.42-5.867-16.42-5.867-2.184-5.704-5.42-7.17-5.42-7.17-4.448-3.015.324-3.015.324-3.015 4.934.326 7.523 5.052 7.523 5.052 4.367 7.496 11.404 5.378 14.235 4.074.404-3.178 1.699-5.378 3.074-6.6-10.839-1.141-22.243-5.378-22.243-24.283 0-5.378 1.94-9.778 5.014-13.2-.485-1.222-2.184-6.275.486-13.038 0 0 4.125-1.304 13.426 5.052a46.97 46.97 0 0 1 12.214-1.63c4.125 0 8.33.571 12.213 1.63 9.302-6.356 13.427-5.052 13.427-5.052 2.67 6.763.97 11.816.485 13.038 3.155 3.422 5.015 7.822 5.015 13.2 0 18.905-11.404 23.06-22.324 24.283 1.78 1.548 3.316 4.481 3.316 9.126 0 6.6-.08 11.897-.08 13.526 0 1.304.89 2.853 3.316 2.364 19.412-6.52 33.405-24.935 33.405-46.691C97.707 22 75.788 0 48.854 0z' fill='%2324292f'/%3E%3C/svg%3E"
  alt="GitHub"
  width={size}
/>

```

We make use of this in `src-svelte/src/routes/credits/Creditor.svelte`:

```svelte
<script lang="ts">
  import GitHubIcon from "./GitHubIcon.svelte";

  ...

  const isGitHubLink = url.startsWith("https://github.com");
</script>

<div class="creditor">
  ...
  <div class="external-link">
    {#if isGitHubLink}
      <GitHubIcon />
    {/if}
    <a href=...>
      ...
    </a>
  </div>
</div>

<style>
  ...

  .external-link {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
</style>
```

We add a screenshot test entry to `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "credits", "creditor"],
    variants: ["github-contributor"],
  }
];
```

eslint complains about the max line length for the GitHub icon image source in `src-svelte/src/routes/credits/GitHubIcon.svelte`. We don't want to break that line up, so we just disable the rule there:

```svelte
...

<!-- eslint-disable max-len -->
<img
  src="..."
  ...
/>
```

Next, we want finer-grained control over what counts as a grid layout, so we the grid CSS logic out of `Credits.svelte` into `src-svelte/src/routes/credits/Grid.svelte`:

```svelte
<div class="credits-grid">
  <slot />
</div>

<style>
  .credits-grid {
    --side-padding: 0.8rem;
    display: grid;
    grid-template-columns: 1fr;
    gap: 0.1rem;
    margin: -0.5rem 0 0;
  }

  /* this takes sidebar width into account */
  @media (min-width: 52rem) {
    .credits-grid {
      grid-template-columns: 1fr 1fr;
    }
  }
</style>

```

We want to support logos, so we edit `src-svelte/src/routes/credits/Creditor.svelte` and wrap out existing detail display into a `details` div:

```svelte
<script lang="ts">
  ...

  export let logo: string | undefined = undefined;
  ...

  ...
  const logoLink = logo ? `/logos/${logo}.png` : undefined;
</script>

<div class="creditor">
  {#if logo}
    <img src={logoLink} alt={name} />
  {/if}
  <div class="details">
    <h4>{name}</h4>
    ...
  </div>
</div>

<style>
  .creditor {
    ...
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  img {
    width: 2rem;
  }

  ...
</style>
```

We add the Tauri logo to `src-svelte/static/logos/tauri.png`, and reference all this in `src-svelte/src/routes/credits/Credits.svelte`:

```svelte
<script lang="ts">
  ...
  import Grid from "./Grid.svelte";
</script>

<InfoBox title="Credits">
  <div class="container">
    <SubInfoBox
      subheading="Zen ..."
    >
      <Grid>
        <Creditor name="Amos Jun-yeung Ng"... />
      </Grid>
    </SubInfoBox>
  </div>
  <div class="container">
    <SubInfoBox subheading="Frameworks">
      <Grid>
        <Creditor name="Tauri" logo="tauri" url="https://tauri.app/" />
      </Grid>
    </SubInfoBox>
  </div>
</InfoBox>

<style>
  ... grid styles removed ...
</style>

```

We realize that we actually want the Tauri link display to not include the trailing slash at the end, so we edit `src-svelte/src/routes/credits/Creditor.svelte` to always strip it out if it exists:

```ts
  export function formatUrl(url: string) {
    if (url.endsWith("/")) {
      url = url.slice(0, -1);
    }

    if (url.startsWith("https://github.com/")) {
      return url.slice(19);
    }
    ...
  }
```

We now check for this in `src-svelte/src/routes/credits/Creditor.test.ts` as well:

```ts
  test("strips ending slash from URL", () => {
    expect(formatUrl("https://tauri.app/")).toEqual("tauri.app");
  });
```

We add a new story to `src-svelte/src/routes/credits/Creditor.stories.ts`:

```ts
export const DependencyWithIcon: StoryObj = Template.bind({}) as any;
DependencyWithIcon.args = {
  name: "Tauri",
  logo: "tauri",
  url: "https://tauri.app/",
};
```

and register this in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "credits", "creditor"],
    variants: [..., "dependency-with-icon"],
  },
];
```

We notice in the Full Page reveal that different lines of the details are revealed at different times. This is because the Info Box reveal animation assumes that these separate elements should be revealed separately. Instead, we edit `src-svelte/src/routes/credits/Creditor.svelte` to add the `atomic-reveal` class to get the reveal effect to apply to the creditor div as a whole:

```svelte
<div class="creditor atomic-reveal">
  ...
</div>
```

Next, we add the Svelte logo, which is an SVG, to `src-svelte/static/logos/svelte.svg`. This means that we must edit `src-svelte/src/routes/credits/Creditor.svelte` to handle SVGs:

```ts
  ...
  const logoLink = logo ? `/logos/${logo}` : undefined;
```

We must then edit the Tauri entry in `src-svelte/src/routes/credits/Credits.svelte` accordingly to include the image extension, and also add the Svelte entry:

```svelte
      <Grid>
        <Creditor name="Tauri" logo="tauri.png" ... />
        <Creditor name="Svelte" logo="svelte.svg" url="https://svelte.dev/" />
      </Grid>
```

Next, we figure out how to edit `src-svelte/src/routes/credits/Grid.svelte` to display the credit divs in as many columns as will fit on screen, while also being centered:

```css
  .credits-grid {
    display: flex;
    flex-wrap: wrap;
    margin: -0.5rem 0 0;
    justify-content: space-around;
  }
```

This in turn means that we get rid of the grid for main project contributors, at least until more contributors arrive. We edit `src-svelte/src/routes/credits/Credits.svelte` to remove the grid and place Amos directly inside the SubInfoBox:

```svelte
    <SubInfoBox
      subheading="Zen ..."
    >
      <Creditor name="Amos Jun-yeung Ng" ... />
    </SubInfoBox>
```

We continue to add more dependencies. We find that [SVG Porn](https://svgporn.com/) is a good site to find many logos on.

In the course of doing this, we find that there are more edge cases for URLs to be formatted. We edit the `formatUrl` function in `src-svelte/src/routes/credits/Creditor.svelte`:

```ts
  const MAX_URL_LENGTH = 15;

  function formatUrlHelper(urlString: string) {
    const url = new URL(urlString);
    if (url.hostname.endsWith("github.com")) {
      if (url.pathname.endsWith("/")) {
        return url.pathname.slice(1, -1);
      }
      return url.pathname.slice(1);
    }

    const hostname = url.hostname.startsWith("www.") ? url.hostname.slice(4) : url.hostname;

    let pathname: string;
    if (url.pathname === "/") {
      pathname = "";
    } else if (url.pathname.endsWith("/")) {
      pathname = url.pathname.slice(0, -1);
    } else {
      pathname = url.pathname;
    };
    return hostname + pathname;
  }

  export function formatUrl(urlString: string) {
    const formattedUrl = formatUrlHelper(urlString);
    if (formattedUrl.length > MAX_URL_LENGTH) {
      return formattedUrl.slice(0, MAX_URL_LENGTH - 3) + "...";
    }
    return formattedUrl;
  }
```

We add a few more test cases in `src-svelte/src/routes/credits/Creditor.test.ts`:

```ts
describe("URL formatter", () => {
  ...

  test("formats Github project URLs correctly", () => {
    expect(formatUrl("https://github.com/ai/nanoid")).toEqual("ai/nanoid");
  });

  ...

  test("strips www from URL", () => {
    expect(formatUrl("https://www.neodrag.dev/")).toEqual("neodrag.dev");
  });

  test("preserves path", () => {
    expect(formatUrl("https://www.a.com/b/c")).toEqual("a.com/b/c");
  });

  test("truncates long URLs", () => {
    expect(formatUrl("https://www.jacklmoore.com/autosize/")).toEqual("jacklmoore.c...");
  });
});

```

We find that the grid gets too busy with too many elements, so we once again change `src-svelte/src/routes/credits/Grid.svelte` to a CSS grid layout:

```css
  .credits-grid {
    margin: -0.5rem auto 0;
    display: grid;
    grid-template-columns: 1fr;
    justify-items: center;
  }

  /* this takes sidebar width into account */
  @media (min-width: 52rem) {
    .credits-grid {
      grid-template-columns: 1fr 1fr;
    }
  }
```

We edit `src-svelte/src/routes/credits/Creditor.svelte` as well to lessen the padding on single-column layouts:

```css
  .creditor {
    padding: 0.5rem 0;
    ...
  }

  @media (min-width: 52rem) {
    .creditor {
      padding: 1rem;
    }
  }
```

We realize that this shows a single column on our default viewport, so instead we lower the `min-width` to `40rem` in both files.

We add a dedication, and set it apart from the credits info box:

```svelte
...

<div class="dedication-container">
  <InfoBox title="Dedication" childNumber={1}>
    <p class="atomic-reveal">
      This project is made with ❤️, and dedicated to <strong
        >Eliza Lynn Kith</strong
      >.
    </p>
  </InfoBox>
</div>

<style>
  ...

  .dedication-container {
    margin: 2rem 0;
  }
</style>
```

## Auto-scroll

We edit `src-svelte/src/routes/credits/Credits.svelte` to auto-scroll, but [stop](https://stackoverflow.com/a/24100456) as soon as the user manually scrolls:

```svelte
  ...
  import { standardDuration } from "$lib/preferences";
  import { onMount } from "svelte";
  ...

  let userScrolled = false;

  function pageScroll() {
    if (userScrolled) {
      return;
    }
    window.scrollBy(0,1);
    setTimeout(pageScroll,10);
  }

  onMount(() => { 
    document.addEventListener('DOMMouseScroll', () => {
      userScrolled = true;
    },false); 
    setTimeout(() => {
      pageScroll();
    }, 4.4 * $standardDuration);
  });
```

4.4 times the standard duration is obtained from `src-svelte/src/lib/InfoBox.test.ts`, where we see that the info box animations usually take 440 ms to finish on fast animation speed:

```ts
    // regular info box animation values
    expect(timing.infoBox).toEqual(
      new PrimitiveTimingMs({
        delayMs: 180,
        durationMs: 260,
      }),
    );
```

We realize that the above solution can actually never stop scrolling, so instead we use a setInterval solution that can then be cleared:


```ts
  let scrollInterval: NodeJS.Timeout | undefined = undefined;

  function stopScrolling() {
    if (scrollInterval) {
      clearInterval(scrollInterval)
    };
  }

  onMount(() => {
    document.addEventListener(
      "DOMMouseScroll",
      stopScrolling,
      false,
    );
    setTimeout(() => {
      scrollInterval = setInterval(function () {
        window.scrollBy(0, 1);
      }, 10);
    }, 4.4 * $standardDuration);

    return stopScrolling;
  });
```

This all works in the browser, but not in the app. This is because the app has everything scrolling within `main-container`. We edit `src-svelte/src/routes/AppLayout.svelte` to make `main-container` an ID instead of a class, because it should be unique:

```svelte
<div id="app">
  ...

    <div id="main-container">
      ...
    </div>
  ...
</div>

<style>
  ...

  #main-container {
    ...
  }

  ...
</style>
```

We now find this element in `src-svelte/src/routes/credits/Credits.svelte`:

```ts
  ...
  let scrollElement: HTMLDivElement | undefined = undefined;

  ...

  onMount(() => {
    const mainContainer = document.getElementById("main-container");
    if (mainContainer) {
      scrollElement = mainContainer as HTMLDivElement;
      document.addEventListener("DOMMouseScroll", stopScrolling, false);
      setTimeout(() => {
        scrollInterval = setInterval(function () {
          if (scrollElement) {
            scrollElement.scrollBy(0, 2);
          }
        }, 10);
      }, 4.4 * $standardDuration);
    }

    return function() {
      stopScrolling();
      document.removeEventListener("DOMMouseScroll", stopScrolling, false);
    };
  });
```

We realize that this event is actually [deprecated](https://developer.mozilla.org/en-US/docs/Web/API/Element/DOMMouseScroll_event), and doesn't work on the Mac. We change it to just a generic MouseEvent instead, and also remember to actually stop listening when the component is unmounted:

```ts
  onMount(() => {
    ...
    if (mainContainer) {
      ...
      scrollElement.addEventListener("MouseEvent", stopScrolling, true);
      ...
    }

    return function () {
      ...
      if (scrollElement) {
        scrollElement.removeEventListener("MouseEvent", stopScrolling, true);
      }
    };
  });
```

This is still not working. We finally just go with encapsulating everything in a div and added the event listeners directly to that div in HTML:

```ts
<div on:mousedown={stopScrolling} on:wheel={stopScrolling} role="none">
  ...
</div>
```

Finally, the dedication is somehow situated at the bottom. Bottom margins do not appear to affect layout here. Fortunately, we have a surrounding div to add padding to now. We give it the `container` class and rename the other `container` classes to `subinfo-container`:

```svelte
<div class="container" on:mousedown={stopScrolling} on:wheel={stopScrolling} role="none">
  <InfoBox title="Credits">
    <div class="subinfo-container">
      ...
    </div>
    <div class="subinfo-container">
      ...
    </div>
  </InfoBox>

  ...
</div>

<style>
  .container {
    padding-bottom: 1rem;
  }

  .subinfo-container {
    ...
  }

  .dedication-container {
    margin-top: 2rem;
  }
</style>
```

## Fixing the Tauri logo render

The `dependency-with-icon` Storybook screenshot test fails. We see that this may be because the images load [lazily](https://stackoverflow.com/a/77288613). We try the associated solution by editing `src-svelte/src/routes/storybook.test.ts` while adhering to TypeScript recommendations:

```ts
      test<StorybookTestContext>(
        `${testName} should render the same`,
        async ({ expect, page }) => {
          ...
          // wait for images to load
          const imagesLocator = page.locator("//img");
          const images = await imagesLocator.evaluateAll((images) => {
            return images.map((i) => {
              i.scrollIntoView();
              return i as HTMLImageElement;
            });
          });
          const imagePromises = images.map(
            (i) => i.complete || new Promise((f) => (i.onload = f)),
          );
          await Promise.all(imagePromises);
          ...
      });
```

This still doesn't work. Upon further debugging, no `img`'s are actually being matched. Upon even further debugging, we see that when we edited the way dependency icons are specified, we forgot to change it in the story as well. We edit `src-svelte/src/routes/credits/Creditor.stories.ts` to fix this:

```ts
DependencyWithIcon.args = {
  ...,
  logo: "tauri.png",
  ...
};
```

and remove the other change to the Storybook test file because it wasn't working anyways.

## Adding Typodermic font logo

For Typodermic fonts, we'll display a small Typodermic logo next to the link, like we do for GitHub links. We add the Typodermic logo at `src-svelte/static/logos/typodermic.png`, and then create `src-svelte/src/routes/credits/TypodermicIcon.svelte` based on `GitHubIcon.svelte`:

```svelte
<script lang="ts">
  export let size = "18px";
</script>

<img src="/logos/typodermic.png" alt="Typodermic" width={size} />

```

We add the corresponding logic to `src-svelte/src/routes/credits/Creditor.svelte`, right next to where the existing logic for `isGitHubLink` is:

```svelte
<script lang="ts" context="module">
  ...

  function formatUrlHelper(urlString: string) {
    ...
    if (url.hostname.endsWith("github.com")) {
      ...
    }

    if (url.hostname.endsWith("typodermicfonts.com")) {
      return url.pathname.slice(1, -1);
    }

    ...
  }

  ...
</script>

<script lang="ts">
  ...
  import GitHubIcon from "...";
  import TypodermicIcon from "./TypodermicIcon.svelte";
  ...
  const isGitHubLink = ...;
  const isTypodermicLink = url.startsWith("https://typodermicfonts.com");
  ...
</script>

<div class="creditor atomic-reveal">
  ...
      {#if isGitHubLink}
        ...
      {:else if isTypodermicLink}
        <TypodermicIcon />
      {/if}
  ...
</div>
```

We add new stories to `src-svelte/src/routes/credits/Creditor.stories.ts` to demonstrate, along with the regular variant that we realize we've never added:

```ts
...

export const Regular: StoryObj = Template.bind({}) as any;
Regular.args = {
  name: "Serde",
  url: "https://serde.rs/",
};

...

export const TypodermicFont: StoryObj = Template.bind({}) as any;
TypodermicFont.args = {
  name: "Nasalization",
  url: "https://typodermicfonts.com/nasalization/",
};

...
```

As usual, we register these in `src-svelte/src/routes/storybook.test.ts`:

```ts
const components: ComponentTestConfig[] = [
  ...,
  {
    path: ["screens", "credits", "creditor"],
    variants: ["regular", ..., "typodermic-font", ...],
  },
];
```
