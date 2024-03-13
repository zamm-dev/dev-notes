# Starting a new Astro project

Do

```bash
$ pnpm create astro@latest
...
 next   Liftoff confirmed. Explore your project!

         Enter your project directory using cd ./zamm-website
         Run pnpm dev to start the dev server. CTRL+C to stop.
         Add frameworks like react or tailwind using astro add.

         Stuck? Join us at https://astro.build/chat

╭─────╮  Houston:
│ ◠ ◡ ◠  Good luck out there, astronaut! 🚀
╰─────╯
```

We run the command as instructed:

```
$ pnpm dev

> zamm-website@0.0.1 dev C:\Users\Amos Ng\Documents\projects\zamm-dev\website
> astro dev

▶ Astro collects anonymous usage data.
  This information helps us improve Astro.
  Run "astro telemetry disable" to opt-out.
  https://astro.build/telemetry


 astro  v4.3.5 ready in 374 ms

┃ Local    http://localhost:4321/
┃ Network  use --host to expose

19:26:49 watching for file changes...
```

We find that we are unable to access the local webserver. We try with the `--host` option, and a permissions dialog pops up. We allow network access for the NodeJS executable, and find that we can now reach the webserver. We kill it, and start it again without the `--host` option, and find that we are once again unable to access the website. It appears that `--host` is necessary after all.

We add Svelte to this project:

```
$ astro "add" "svelte"

✔ Resolving packages...
20:12:01 
  Astro will run the following command:
  If you skip this step, you can always run it yourself later

 ╭─────────────────────────────────────────────────╮
 │ pnpm add @astrojs/svelte@^5.0.3 svelte@^4.2.10  │
 ╰─────────────────────────────────────────────────╯

√ Continue? ... yes
✔ Installing dependencies...
20:12:12
  Astro will generate a minimal ./svelte.config.js file.

√ Continue? ... yes
20:12:16
  Astro will make the following changes to your config file:

 ╭ astro.config.mjs ─────────────────────────────╮
 │ import { defineConfig } from 'astro/config';  │
 │ import mdx from '@astrojs/mdx';               │
 │ import sitemap from '@astrojs/sitemap';       │
 │                                               │
 │ import svelte from "@astrojs/svelte";         │
 │                                               │
 │ // https://astro.build/config                 │
 │ export default defineConfig({                 │
 │   site: 'https://example.com',                │
 │   integrations: [mdx(), sitemap(), svelte()]  │
 │ });                                           │
 ╰───────────────────────────────────────────────╯

√ Continue? ... yes
20:12:24
   success  Added the following integration to your project:
  - @astrojs/svelte
```

## Customizing the blog

We remove the existing `.md` files in `src/content/blog/`, and rename `src/content/blog/using-mdx.mdx` to `src/content/blog/v0.1.0.mdx` before modifying it as such:

```astro
---
zammVersion: '0.1.0'
title: 'Zen and the Automation of Metaprogramming for the Masses'
description: 'ZAMM now has a GUI.'
pubDate: 'February 14, 2024'
heroImage: '/blog-placeholder-5.jpg'
---

ZAMM now has a GUI.
```

We want to tag this blog post with a release version, so we edit `src\content\config.ts` to include `zammVersion` in the schema:

```ts
const blog = defineCollection({
	...
	schema: z.object({
		zammVersion: z.string(),
		...
	}),
});
```

We make use of this new metadata in `src\layouts\BlogPost.astro`:

```astro
...
type Props = CollectionEntry<'blog'>['data'];

const { zammVersion, ... } = Astro.props;
const fullTitle = `v${zammVersion} - ${title}`;
---

<html lang="en">
	<head>
		<BaseHead title={fullTitle} ... />
    <style>
      ...
      .title h1 {
				margin: 0 -5em 0.5em;
			}
			.title h1 .version {
				font-size: 0.8em;
				color: rgb(var(--gray));
			}
      ...
    </style>
  </head>

  <body>
    ...
    <main>
			<article>
        ...
        <div class="prose">
					<div class="title">
            ...
            <h1><span class="version">v{zammVersion}: </span>{title}</h1>
            ...
          </div>
          ...
        </div>
      </article>
    </main>
    ...
  </body>
</html>
```

where we widen the space for the title in order to produce less line wrapping, and display the version number next to the main title. We also do the same in `src\pages\blog\index.astro`:

```astro
...
<html lang="en">
	<head>
		...
		<style>
      ...
      ul li h4 .version {
				font-size: 0.8em;
				color: rgb(var(--gray));
			}
      ...
    </style>
  </head>
  <body>
    ...
        <ul>
					{
						posts.map((post) => (
							...
									<h4 class="title">
										<span class="version">v{post.data.zammVersion}: </span>
										{post.data.title}
									</h4>
							...
						))
					}
				</ul>
    ...
  </body>
</html>
```

We do

```bash
$ pnpm astro check
...
src/pages/about.astro:5:2 - error ts(2322): Type '{ children: any[]; title: string; description: string; pubDate: Date; heroImage: string; }' is not assignable to type 'IntrinsicAttributes & { title: 
string; description: string; zammVersion: string; pubDate: Date; updatedDate?: Date | undefined; heroImage?: string | undefined; }'.
  Property 'zammVersion' is missing in type '{ children: any[]; title: string; description: string; 
pubDate: Date; heroImage: string; }' but required in type '{ title: string; description: string; zammVersion: string; pubDate: Date; updatedDate?: Date | undefined; heroImage?: string | undefined; }'.
5 <Layout
   ~~~~~~

Result (16 files): 
- 1 error
- 0 warnings
- 0 hints

 ELIFECYCLE  Command failed with exit code 1.
```

The "about" page isn't a regular blog post, so we copy over the template from `index.astro` and modify it a bit:

```astro
---
import BaseHead from '../components/BaseHead.astro';
import Header from '../components/Header.astro';
import Footer from '../components/Footer.astro';
import { SITE_DESCRIPTION } from '../consts';
---

<!doctype html>
<html lang="en">
	<head>
		<BaseHead title="About ZAMM" description={SITE_DESCRIPTION} />
	</head>
	<body>
		<Header />
		<main>
			<h1>Hi</h1>
			<p>This is a page about ZAMM.</p>
		</main>
		<Footer />
	</body>
</html>

```

Now our check succeeds:

```bash
$ pnpm astro check
...
Result (18 files): 
- 0 errors
- 0 warnings
- 0 hints
```

and we commit.

We also want our blog post slug to be different. It doesn't appear as if there is a way to customize the slug generation logic for Astro natively, but [this discussion](https://github.com/withastro/astro/issues/3468) mentions [rehype-slugify](https://www.npmjs.com/package/@microflash/rehype-slugify). Unfortunately, based on the description:

> This plugin is useful when you have relatively long documents and you want to be able to link to particular sections.

it appears this is not quite what we are looking for. Instead, we follow the guide [here](https://equk.co.uk/2023/02/02/generating-slug-from-title-in-astro/) and create `src\lib\versionSlug.ts`:

```ts
import type { CollectionEntry } from 'astro:content';

type BlogEntry = CollectionEntry<'blog'>;

export default function (entry: BlogEntry) {
  return `v${entry.data.zammVersion}`;
}

```

We edit `src\pages\rss.xml.js` to use this new version slug logic instead:

```js
...
import versionSlug from '../lib/versionSlug';

export async function GET(context) {
	const posts = await getCollection('blog');
	return rss({
		...,
		items: posts.map((post) => ({
			...,
			link: `/blog/${versionSlug(post)}/`,
		})),
	});
}

```

We edit this in `src\pages\blog\[...slug].astro` so that each blog post is served up at the right URL:

```astro
---
...
import versionSlug from '../../lib/versionSlug';
...

export async function getStaticPaths() {
	...
	return posts.map((post) => ({
		params: { slug: versionSlug(post) },
		...
	}));
}
...
---

...
```

We also edit `src\pages\blog\index.astro` for the main blog page to link correctly to each blog post:

```astro
---
...
import versionSlug from '../../lib/versionSlug';
...
---

<!doctype html>
<html lang="en">
  ...
  <body>
		...
		<main>
			<section>
				<ul>
					{
						posts.map((post) => (
							<li>
								<a href={`/blog/${versionSlug(post)}/`}>
									...
								</a>
							</li>
						))
					}
				</ul>
			</section>
		</main>
		<Footer />
	</body>
</html>

```

Let's edit the main page as well to always link to the latest release. We edit `src\pages\index.astro` like so, based on the documentation [here](https://www.howtocode.io/posts/astro/content-collections):

```astro
---
...
import { getCollection } from "astro:content";
import versionSlug from '../lib/versionSlug';

const blogPosts = await getCollection("blog");
// find latest blog post
const latestPost = blogPosts.sort((a, b) => {
	const aVersion = a.data.zammVersion.split('.').map(Number);
	const bVersion = b.data.zammVersion.split('.').map(Number);
	return aVersion[0] - bVersion[0] || aVersion[1] - bVersion[1] || aVersion[2] - bVersion[2];
})[0];
const { zammVersion: latestVersion } = latestPost.data;
---

<!doctype html>
<html lang="en">
	<head>
		...
		<style>
			h1 {
				text-align: center;
			}
		</style>
	</head>
	<body>
		...
		<main>
			<h1>ZAMM</h1>
			<p>Download <a href={`/blog/${versionSlug(latestPost)}`}>version {latestVersion}</a>.</p>
		</main>
		...
	</body>
</html>

```

We find that the social links in the header and footer repeat some code, so we refactor that out into `src/components/SocialLinks.astro`:

```astro
<div class="social-links">
  <a href="https://github.com/zamm-dev" target="_blank">
    <span class="sr-only">Go to the zamm.dev GitHub profile</span>
    <svg viewBox="0 0 16 16" aria-hidden="true" width="32" height="32" astro-icon="social/github"
      ><path
        fill="currentColor"
        d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.012 8.012 0 0 0 16 8c0-4.42-3.58-8-8-8z"
      ></path></svg
    >
  </a>
</div>

<style>
  .social-links {
		display: flex;
		justify-content: center;
		gap: 1em;
	}
</style>

```

We use this in `src\components\Footer.astro`:

```astro
---
import SocialLinks from './SocialLinks.astro';

...
---

<footer>
	<div class="contents">
		<p class="copyright">
			&copy; {today.getFullYear()} ZAMM Consulting, LLC.
		</p>
		<div class="social-container"><SocialLinks /></div>
	</div>
</footer>

<style>
	footer {
		padding: 2em 1em 2em 1em;
    ...
  }

	.contents {
		width: fit-content;
		margin: 0 auto;
		display: flex;
		flex-direction: row;
		gap: 1em;
	}

	.social-container {
		margin-top: 1em;
	}
	.social-container :global(.social-links a) {
		...
	}
	.social-container :global(.social-links a:hover) {
		...
	}
</style>
```

We merge all footer content into one row and center it. Note that because we have refactored the social links into a separate component, we add `:global` to the styles for that to apply. We also update the copyright text to reflect our website.

We use this in `src\components\Header.astro` as well:

```astro
---
...
import SocialLinks from './SocialLinks.astro';
...
---

<header>
	<nav>
		...
		<div class="social-container">
			<SocialLinks />
		</div>
	</nav>
</header>

<style>
  ...
  nav a, .social-container :global(.social-links a) {
		...
	}
	...
	.social-container :global(.social-links) {
		margin: 0;
	}
	.social-container :global(.social-links a) {
		height: 32px;
	}
	@media (max-width: 720px) {
		.social-container {
			display: none;
		}
	}
</style>
```

We use the same `:global` trick here as we did for the footer.

We customize the site by editing `src\consts.ts`:

```ts
export const SITE_TITLE = 'ZAMM';
export const SITE_DESCRIPTION = 'Zen and the Automation of Metaprogramming for the Masses';

```

It does not appear that there is a way to include CSS inside of the `.mdx` blog posts, so instead we put the CSS for blog post signatures inside `src\layouts\BlogPost.astro`:

```astro
...

<html lang="en">
	<head>
		...
		<style>
      ...
      .prose :global(.signature) {
				margin-left: auto;
				text-align: right;
			}
			.prose :global(.signature p) {
				margin: 0;
			}
      ...
    </style>
  </head>
  ...
</html>
```

### OpenGraph image

We find that the `og:image` is showing the default image despite us specifying a hero image for the blog post. We discover that this is because the default template in `src\layouts\BlogPost.astro` is missing this in `BaseHead`:

```astro
<html lang="en">
	<head>
		<BaseHead ... image={heroImage} />
		...
```

We also remove `public\blog-placeholder-1.jpg` because it says "Build the web you want", which is not what we want to display by default for our blog for any pages that we don't set a custom image for. After deleting that file, we search for all instances of it, and only find one in `src\components\BaseHead.astro`:

```ts
const { title, description, image = '/blog-placeholder-1.jpg' } = Astro.props;
```

We place a custom default image in `public\default-blog-image.png` and edit the above to

```ts
const { title, description, image = '/default-blog-image.jpg' } = Astro.props;
```

## Sitemaps

We see from [this documentation](https://docs.astro.build/en/guides/integrations-guide/sitemap/) that we are supposed to add references to the sitemaps ourselves. We see that the built sitemap does exist at `/sitemap-index.xml`, which points to a `/sitemap-0.xml` that also exists.

As suggested on the documentation site, we edit `src\components\BaseHead.astro` to include

```html
<link rel="sitemap" href="/sitemap-index.xml" />
```

We add it to `public\robots.txt` as well, creating the file itself because it doesn't exist yet:

```
User-agent: *
Allow: /

Sitemap: https://zamm.dev/sitemap-index.xml
```

## Code blocks inside HTML

While writing up a blog post, we find that

```mdx
<ol>
  <li>Now the tests fail with
		\`\`\`
		TypeError: range.getBoundingClientRect is not a function
		\`\`\`
  </li>
</ol>
```

fails with the error

```
MDXError: Expected a closing tag for `<li>` (69:3-69:7) before the end of `paragraph`
    at TransformContext.transform (file:///C:/Users/Amos%20Ng/Documents/projects/zamm-dev/website/node_modules/.pnpm/@astrojs+mdx@2.1.1_astro@4.3.5/node_modules/@astrojs/mdx/dist/index.js:92:27)
    at async Object.transform (file:///C:/Users/Amos%20Ng/Documents/projects/zamm-dev/website/node_modules/.pnpm/vite@5.1.0/node_modules/vite/dist/node/chunks/dep-nGG-_oRu.js:50818:30)
		...
```

It appears that this may be a [known problem](https://github.com/orgs/mdx-js/discussions/2241). Upon further experimentation, we discover that starting on a newline gets the MDX parser working correctly:

```mdx
<ol>
  <li>
    Now the tests fail with
    \`\`\`
    TypeError: range.getBoundingClientRect is not a function
    \`\`\`
  </li>
</ol>
```

### Styling

We style these code blocks with this code in `src\styles\global.css`, so that it is easier to read large stack traces:

```css
pre {
	white-space: pre-wrap;
	text-align: left;
	font-size: 1rem;
	...
}
```

## Dev mode on Windows

We edit `package.json` to get the dev script working by default on Windows:

```json
{
  ...,
  "scripts": {
    "dev": "astro dev --host",
    ...
  },
	...
}
```

Since we're not allowing external connections to NodeJS, this should be relatively safe.

## Canonical URL

Our submission to HN gets changed automatically into `example.com`, and this turns out to be because we have

```html
<link rel="canonical" href="https://example.com/">
```

in our header. We find out that this is because `src/components/BaseHead.astro` has

```astro
const canonicalURL = new URL(Astro.url.pathname, Astro.site);
```

Looking at the documentation for those two variables, and going through our code base, we see that `astro.config.mjs` has

```mjs
export default defineConfig({
  site: 'https://example.com',
  ...
});
```

We change that to `https://zamm.dev`. We are still getting the same behavior. We restart the dev server, and finally verify that it is now fixed.

## Playing videos on mobile

We see that the videos aren't autoplaying on mobile. We look for why this might be, and find [this answer](https://stackoverflow.com/a/49125732). We fix our video tags accordingly, like so:

```html
<video playsinline autoplay loop muted width="100%">
  <source src="/posts/v0.1.0/title-grow.mp4" type="video/mp4" />
  This browser does not display the video tag.
</video>
```

## Constraining hero image size

We find that we want the hero image to be at most 500px tall, while preserving the aspect ratio. We discover [this answer](https://stackoverflow.com/a/3751836). We edit `src/layouts/BlogPost.astro` as recommended, in the process moving `margin: 0 auto;` from the `img` to its parent container:

```css
			.hero-image {
				width: fit-content;
				margin: 0 auto;
			}
			.hero-image img {
				...
				max-height: 500px;
				width: auto;
				...
			}
```
