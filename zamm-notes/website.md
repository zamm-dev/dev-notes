# zamm.dev website configuration


## Release links component

We want to show the same set of release links on the front page as we do on the blog post. First we define the new data we want at `src/content/config.ts`:

```ts
...

export const ReleaseLinksSchema = z.object({
	page: z.string(),
	linuxAppImage: z.string(),
	linuxDeb: z.string(),
	macDmg: z.string(),
	windowsExe: z.string(),
	windowsMsi: z.string(),
});

export type ReleaseLinks = z.infer<typeof ReleaseLinksSchema>;

const BlogSchema = z.object({
	zammVersion: z.string(),
	...,
	releaseLinks: ReleaseLinksSchema,
	...
});

export type BlogPost = z.infer<typeof BlogSchema>;

const blog = defineCollection({
	type: 'content',
	// Type-check frontmatter using a schema
	schema: BlogSchema,
});

...

```

where `ReleaseLinksSchema` is the new set of metadata we wish to associate with each release, nested within `BlogPost`. We find out about the use of `z.infer` from [this](https://zod.dev/?id=basic-usage) zod documentation page. We name these types explicitly because we originally thought we'd need to make use of `ReleaseLinks` directly, and then later want to make use of `BlogPost` directly without having to refer to it indirectly via the blog collection `'data'` field.

Next, we define our new component at `src/components/ReleaseLinks.astro`:

```astro
---
import type { BlogPost } from '../content/config';
import versionSlug from '../lib/versionSlug';


type Props = {
  release: BlogPost;
  linkBlogPost?: boolean;
};

const {
  release,
  linkBlogPost,
} = Astro.props;

const {
  zammVersion: releaseVersion,
  title: blogPostTitle,
  releaseLinks: {
    page: releasePageLink,
    windowsExe: windowsExeLink,
    windowsMsi: windowsMsiLink,
    macDmg: macDmgLink,
    linuxAppImage: linuxAppImageLink,
    linuxDeb: linuxDebLink,
  },
} = release;

const blogPostLink = `/blog/${versionSlug(release)}`;
---

<div class="release-container">
  Download <a href={releasePageLink}>v{releaseVersion}</a> {linkBlogPost &&
    <span>
      &ndash; <a href={blogPostLink}>{blogPostTitle}</a>
    </span>
  } for:

  <ul>
    <li>
      Windows: <a href={windowsExeLink} download>exe</a> | <a href={windowsMsiLink} download>msi</a>
    </li>
    <li>
      Mac: <a href={macDmgLink} download>dmg</a>
    </li>
    <li>
      Linux: <a href={linuxAppImageLink} download>AppImage</a> | <a href={linuxDebLink} download>deb</a>
    </li>
  </ul>
</div>

<style>
  .release-container {
    background-color: #add8e667;
    padding: 2rem;
    margin-bottom: 3rem;
    border-radius: 1rem;
  }

  ul {
    margin: 0;
  }
</style>

```

Note that we're allowing for the option of linking to the blog post itself from the front page. This option is off for the blog post itself because of course it doesn't make sense to link to the same blog post you're already on.

Now we edit `src/lib/versionSlug.ts` to take in a `BlogPost` instead of a `CollectionEntry`:

```ts
import type { BlogPost } from "../content/config";

export default function (post: BlogPost) {
  return `v${post.zammVersion}`;
}

```

We need to do this because `BlogPost.astro` does not have access to the collection entry, only to the data.

Now that we do this, we need to edit all the places where we use this version slug function. We just have to use the `.data` field instead of the entire blog entry object. We start with `src/pages/rss.xml.js`:

```js
		items: posts.map((post) => ({
			...,
			link: `/blog/${versionSlug(post.data)}/`,
		})),
```

Then we edit `src/pages/blog/index.astro`:

```astro
						posts.map((post) => (
							...
								<a href={`/blog/${versionSlug(post.data)}/`}>
							...
						))
```

We edit `src/pages/blog/[...slug].astro`:

```astro
export async function getStaticPaths() {
	...
	return posts.map((post) => ({
		params: { slug: versionSlug(post.data) },
		...
	}));
}
```

Now we add the download links to the homepage at `src/pages/index.astro`:

```astro
---
...
import ReleaseLinks from '../components/ReleaseLinks.astro';
...
---

...
		<main>
			...
			<ReleaseLinks release={latestPost.data} linkBlogPost />
		</main>
...
```

and to each blog post at `src/layouts/BlogPost.astro`:

```astro
---
import type { BlogPost } from '../content/config';
...
import ReleaseLinks from '../components/ReleaseLinks.astro';

type Props = BlogPost;

const data = Astro.props;
const { zammVersion, title, description, pubDate, updatedDate, heroImage } = data;
...
---

...
				<div class="prose">
					<ReleaseLinks release={data} />
					<slot />
				</div>
...
```

and to our blog post itself at `src/content/blog/v0.1.0.mdx`:

```mdx
---
releaseLinks:
  page: https://github.com/zamm-dev/zamm/releases/tag/v0.1.0
  windowsExe: https://github.com/zamm-dev/zamm/releases/download/v0.1.0/zamm_0.1.0_x64-setup.exe
  windowsMsi: https://github.com/zamm-dev/zamm/releases/download/v0.1.0/zamm_0.1.0_x64_en-US.msi
  macDmg: https://github.com/zamm-dev/zamm/releases/download/v0.1.0/zamm_0.1.0_universal.dmg
  linuxAppImage: https://github.com/zamm-dev/zamm/releases/download/v0.1.0/zamm_0.1.0_amd64.AppImage
  linuxDeb: https://github.com/zamm-dev/zamm/releases/download/v0.1.0/zamm_0.1.0_amd64.deb
---

...
```

after first finding out that the area enclosed by `---` is called the "frontmatter," and then finding out that the frontmatter is [defined](https://github.com/withastro/astro/issues/5285) in [YAML format](https://assemble.io/docs/YAML-front-matter.html), allowing us to properly nest the `releaseLinks` object.

## Social links component

We edit `src\content\config.ts` to define an object that refers to the set of discussion links associated with the latest release:

```ts
export const DiscussionLinksSchema = z.object({
	hackerNews: z.string().optional(),
});

export type DiscussionLinks = z.infer<typeof DiscussionLinksSchema>;

const BlogSchema = z.object({
	...,
	discussions: DiscussionLinksSchema.optional(),
	...
});

```

We create a new component at `src\components\DiscussionLinks.astro`:

```astro
---
import type { DiscussionLinks } from '../content/config';

type Props = DiscussionLinks | undefined;

const {
  hackerNews: hackerNewsLink,
} = Astro.props ?? {};

---

{hackerNewsLink &&
  <div class="discussion-container">
    <p>Discuss on <a href={hackerNewsLink}>HN</a></p>
    <hr />
  </div>
}

<style>
  .discussion-container {
    color: rgb(var(--gray));
  }
  
  hr {
    margin: 2rem 0 4rem;
  }

  p {
    margin: 0;
  }

  a {
    color: rgb(var(--gray));
  }
</style>

```

We insert it into the blog post schema at `src\layouts\BlogPost.astro`. We put it at the top instead of the bottom because it feels cheap to say "DISCUSS THIS NOW ON SOCIAL MEDIA!" right after the love message of the inaugural blog post.

```astro
---
...
import DiscussionLinks from '../components/DiscussionLinks.astro';

...
---

...
				<div class="prose">
					<ReleaseLinks ... />
					<DiscussionLinks {...data.discussions} />
					<slot />
				</div>
...
```

Because `DiscussionLinks` now separate the downloads from the rest of the blog post, we reduce the margins in `src\components\ReleaseLinks.astro`:

```css
  .release-container {
    ...
    margin-bottom: 2rem;
    ...
  }
```

Finally, we make use of this in `src\content\blog\v0.1.0.mdx`

```astro
---
...
discussions:
  hackerNews: https://news.ycombinator.com/item?id=39370512
---
```

## Post history

We edit `src\layouts\BlogPost.astro` to style the post history the way we want, with a link to the GitHub history for the post:

```astro
---
...
const postHistoryUrl = `https://github.com/zamm-dev/website/commits/main/src/content/blog/v${zammVersion}.mdx`;
---

<html lang="en">
	<head>
		...
		<style>
      ...
      .dates {
				margin: 0 auto 0.5em;
				color: rgb(var(--gray));
				width: fit-content;
			}
			.dates td {
				text-align: right;
				padding: 0 0.25rem;
			}
			.dates td.date {
				text-align: left;
			}
			.dates a {
				color: rgb(var(--gray));
			}
		</style>
	</head>

  <body>
    ...
        <div class="title">
					<h1>...</h1>
					<table class="dates">
						<tr>
							<td>Published</td>
							<td class="date"><FormattedDate date={pubDate} /></td>
						</tr>
						{
							updatedDate && (
								<tr>
									<td><a href={postHistoryUrl}>Last update</a></td>
									<td class="date"><FormattedDate date={updatedDate} /></td>
								</tr>
							)
						}
					</table>
				</div>
        ...
  </body>
</html>
```

We update `src\content\blog\v0.1.0.mdx` accordingly:

```astro
---
...
updatedDate: 'February 20, 2024'
...
---

...
```

## Blog post list

### Showing items as a list of rows

The default layout looks confusing for our screenshots. We edit `src/pages/blog/index.astro`:

```astro
<!doctype html>
<html lang="en">
	<head>
		...
		<style>
			...
			ul li {
				width: 100%;
			}
			...
			ul li .img-container {
				width: 200px;
			}
			ul li img {
				border-radius: 12px;
				max-height: 100px;
				max-width: 200px;
				margin: 0 auto;
			}
			ul li a {
				display: flex;
				flex-direction: row;
				gap: 1rem;
				align-items: center;
			}
			...
			.title {
				...
				flex: 1;
			}
			...
		</style>

		...
	</head>
	<body>
		...
					{
						posts.map((post) => (
							<li>
								<a href={...}>
									<div class="img-container">
										<img src={...} alt={post.data.title} />
									</div>
									...
								</a>
							</li>
						))
					}
		...
	</body>
</html>
```
