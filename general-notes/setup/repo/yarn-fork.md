# Forking a repo locally

If some repo does *almost* what you want them to do, but not quite, you can clone it with

```bash
$ mkdir forks
$ cd forks
$ git submodule add https://github.com/PuruVJ/neodrag.git
```

Then you can edit the `package.json` in your main project to point to this new location on disk. If the project does not have the package in their top-level directory, as this one does not, you will have to point it to the specific package you want instead:

```json
{
  ...
  "dependencies": {
    ...
    "@neodrag/svelte": "file:./forks/neodrag/packages/svelte"
  }
}
```

Run `yarn` again to update the install location for the dependency.

Follow the instructions [here](/general-notes/libraries/svelte/neodrag.md) to install the dependencies and compile the project.

If you get an error such as this:

```
11:08:15 PM [vite] Internal server error: Failed to resolve entry for package "@neodrag/svelte". The package may have incorrect main/module/exports specified in its package.json.
  Plugin: vite:import-analysis
  File: /root/zamm/src-svelte/src/lib/Switch.svelte
      at packageEntryFailure (file:///root/zamm/node_modules/vite/dist/node/chunks/dep-df561101.js:28691:11)
      ...
```

We head to the `src-svelte/forks/neodrag/packages/svelte` directory and see that the corresponding entries in the `package.json` are:

```json
{
  ...
  "main": "./dist/index.js",
	"module": "./dist/index.js",
	"exports": {
		".": {
			"types": "./dist/index.d.ts",
			"import": {
				"production": "./dist/min/index.js",
				"development": "./dist/index.js"
			},
			"default": "./dist/min/index.js"
		},
		"./package.json": "./package.json"
	},
  ...
}
```

We had followed the instructions to build it, so it does exist:

```bash
$ ls ./dist/index.js
./dist/index.js
```

We can instead go back to our own project, and reinstall our dependencies:

```bash
$ rm -rf node_modules
$ yarn
```

Now everything works. Note that if you're using workspaces, you may have to remove `node_modules` from both the workspace and the project root. In particular, Storybook keeps a cache in the local workspace `node_modules`, whereas non-executable project dependencies go in the root project `node_modules`.

Note that you'll want to make sure that you can make changes that are observably reflected in downstream usage, for example with a Storybook component, before moving on to make real changes. This might include removing exports to see if the build process breaks (if it doesn't, that means the change was not registered in the build process). If things keep not working, you may want to resort to more and more drastic measures, such as removing the build output entirely and seeing if the build process now errors out on the missing output. If it starts working, you may want to start testing smaller and smaller changes, such as changing the functionality of the component in obvious ways to see if those particular changes are being registered as well. For JavaScript, this might invovlve setting off an alert, which is very obvious when called.

To change this to use your own fork (if you didn't already create a fork), do this inside the fork repo:

```bash
$ git remote remove origin
$ git remote add origin https://github.com/amosjyng/neodrag.git
$ git push --set-upstream origin z-customizations
...
remote: Resolving deltas: 100% (8/8), completed with 4 local objects.
remote: 
remote: Create a pull request for 'z-customizations' on GitHub by visiting:
remote:      https://github.com/amosjyng/neodrag/pull/new/z-customizations
remote: 
To https://github.com/amosjyng/neodrag.git
 * [new branch]      z-customizations -> z-customizations
Branch 'z-customizations' set up to track remote branch 'z-customizations' from 'origin'.
```

Then, in your own project root, edit `.gitmodules` to point to your own fork.

When you're done with your changes, you can open a pull request to see if the original maintainers are interested in incorporating your changes upstream.

## Makefile

If you add this to your Makefile:

```Makefile
build: forks/neodrag/packages/svelte/dist $(shell find src -type f \( -name "*.svelte" -o -name "*.js" -o -name "*.ts" -o -name "*.html" \) -not -path "*/node_modules/*")
	yarn && yarn svelte-kit sync && yarn build

forks/neodrag/packages/core/dist: forks/neodrag/packages/core/src/*
	cd forks/neodrag/packages/core && pnpm install && pnpm compile

forks/neodrag/packages/svelte/dist: forks/neodrag/packages/core/dist forks/neodrag/packages/svelte/src/*
	cd forks/neodrag/packages/svelte && pnpm install && pnpm compile
```

then you should also add the corresponding clean command:

```Makefile
 clean:
       rm -rf build node_modules ../node_modules forks/neodrag/packages/svelte/dist/dist
```

Once we edit the CSS without it being reflected in the build, we realize that we didn't include `css` in the list of dependent files. We can add it to the list of dependencies:

```Makefile
build: forks/neodrag/packages/svelte/dist $(shell find src -type f \( ... -name "*.css" ... \) -not -path "*/node_modules/*")
	...
```

## yarn linking

To automatically update the installed version of the package whenever you edit it for development purposes, you can try using `yarn link` as described [here](https://stackoverflow.com/a/41879331). Enter the directory for the dependency:

```bash
$ cd forks/neodrag/packages/svelte
$ yarn link                
yarn link v1.22.19
success Registered "@neodrag/svelte".
info You can now run `yarn link "@neodrag/svelte"` in the projects where you want to use this package and it will be used instead.
Done in 0.05s.
```

Then go back to the original directory and

```bash
$ yarn link @neodrag/svelte
yarn link v1.22.19
success Using linked package for "@neodrag/svelte".
Done in 0.05s.
```

However, you may run into this problem:

```
Failed to load url /forks/neodrag/packages/svelte/dist/index.js (resolved id: /root/zamm/src-svelte/forks/neodrag/packages/svelte/dist/index.js) in /root/zamm/src-svelte/src/lib/Switch.svelte. Does the file exist?
Failed to load url /forks/neodrag/packages/svelte/dist/index.js (resolved id: /root/zamm/src-svelte/forks/neodrag/packages/svelte/dist/index.js) in /root/zamm/src-svelte/src/lib/Switch.svelte. Does the file exist?
The request url "/root/zamm/src-svelte/forks/neodrag/packages/svelte/dist/index.js" is outside of Vite serving allow list.

- /root/zamm/src-svelte/src/lib
- /root/zamm/src-svelte/src/routes
- /root/zamm/src-svelte/.svelte-kit
- /root/zamm/src-svelte/src
- /root/zamm/src-svelte/node_modules
- /root/zamm/node_modules
- /root/zamm/src-svelte/.storybook

Refer to docs https://vitejs.dev/config/server-options.html#server-fs-allow for configurations and more details.
```

This is misleading because the file actually exists:

```bash
$ ls /root/zamm/src-svelte/forks/neodrag/packages/svelte/dist/index.js                      
/root/zamm/src-svelte/forks/neodrag/packages/svelte/dist/index.js
```

You would have to change the allow list, as described in one of the solutions [here](https://stackoverflow.com/questions/74902697/error-the-request-url-is-outside-of-vite-serving-allow-list-after-git-init).

To undo, do

```bash
$ yarn unlink @neodrag/svelte
yarn unlink v1.22.19
success Removed linked package "@neodrag/svelte".
info You will need to run `yarn install --force` to re-install the package that was linked.
Done in 0.07s.
```

## Moving the fork location

After creating a `forks` folder in the project's root directory, we wish to move our neodrag fork there too. We see that [newer versions of Git](https://stackoverflow.com/a/6310246) have this feature already, so we check that we can just do that as well with our Git:

```bash
$ git --version 
git version 2.43.0
$ git mv src-svelte/forks/neodrag forks/neodrag
$ git status
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   .gitmodules
        renamed:    src-svelte/forks/neodrag -> forks/neodrag
```

It appears that this only changes the path and not the submodule itself in `.gitmodules`, so we edit it:

```
...
[submodule "forks/neodrag"]
	path = forks/neodrag
	url = https://github.com/amosjyng/neodrag.git
...
```

We edit our `Dockerfile`:

```Dockerfile
RUN git clone --depth 1 --branch zamm/v0.1.1 https://github.com/amosjyng/neodrag.git forks/neodrag && \
  cd forks/neodrag && \
  ...
```

where we have to upload a `zamm/v0.1.1` tag to the remote repo instead of just using the commit hash `e954f97`, because otherwise we get the error

```
Cloning into 'forks/neodrag'...
warning: Could not find remote branch e954f97 to clone.
fatal: Remote branch e954f97 not found in upstream origin
```

We edit our `Makefile`:

```Makefile
copy-docker-deps:
	...
	mv -n /tmp/dependencies/forks/neodrag/packages/svelte/dist ./forks/neodrag/packages/svelte/
	...
```

and our `src-svelte/Makefile`:

```Makefile
build: ../forks/neodrag/packages/svelte/dist ...
	yarn ...

```

and our `.github/workflows/tests.yaml`:

```yaml
jobs:
  build:
    ...
    steps:
      ...
      - name: Upload artifacts
        ...
        with:
          name: builds
          path: |
            ...
            forks/neodrag/packages/core/dist/
            forks/neodrag/packages/svelte/dist/
            ...
          ...
  pre-commit:
    ...
    steps:
      ...
      - name: Build Neodrag fork
        run: |
          ...
          cd forks/neodrag && pnpm install && ...
```

and our `.github/workflows/publish.yaml`:

```yaml
      - name: Build frontend
        run: |
          ...
          cd forks/neodrag
          ...
          cd ..
					...
```

and our `src-svelte/package.json`:

```json
{
	...,
	"dependencies": {
    ...,
    "@neodrag/svelte": "file:../forks/neodrag/packages/svelte",
    ...
  }
}
```

With a `yarn install`, the reference inside `yarn.lock` should update too.

Our CI tests pass at first, but this is due to cached builds. When we run them again later, we run into the problem

```
cd src-svelte && make
make[1]: *** No rule to make target `../forks/neodrag/packages/svelte/dist', needed by `build'.  Stop.
make: *** [svelte] Error 2
```

We reproduce this locally

```bash
$ make clean
$ make
```

We update `src-svelte/Makefile`:

```Makefile
../forks/neodrag/packages/core/dist: ../forks/neodrag/packages/core/src/*
	cd ../forks/...

../forks/neodrag/packages/svelte/dist: ../forks/neodrag/packages/core/dist ../forks/neodrag/packages/svelte/src/*
	cd ../forks/...
```

and check that the build passes locally before trying it on CI.

We then eventually find in [`tauri.md`](/zamm-notes/tauri.md) that the publish script is now broken too. We fix it as described in that file.
