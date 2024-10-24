# Setting up NodeJS with `asdf`

Assuming that you already have `asdf` set up, next install the NodeJS plugin:

```bash
$ asdf plugin add nodejs
updating plugin repository...remote: Enumerating objects: 50, done.
remote: Counting objects: 100% (50/50), done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 50 (delta 26), reused 43 (delta 22), pack-reused 0
Unpacking objects: 100% (50/50), 69.95 KiB | 1023.00 KiB/s, done.
From https://github.com/asdf-vm/asdf-plugins
   097d4aa..b03baaa  master     -> origin/master
HEAD is now at b03baaa feat: add asdf-oapi-codegen plugin (#864)
```

Then install the latest version of NodeJS:

```bash
$ asdf install nodejs latest
Trying to update node-build... ok
Downloading node-v20.5.0-linux-x64.tar.gz...
-> https://nodejs.org/dist/v20.5.0/node-v20.5.0-linux-x64.tar.gz
Installing node-v20.5.0-linux-x64...
Installed node-v20.5.0-linux-x64 to /home/amos/.asdf/installs/nodejs/20.5.0
```

Mark the version we just installed as the global version of NodeJS:

```bash
$ asdf global nodejs 20.5.0
```

To add support for yarn as well, install it via npm:

```bash
$ npm install --global yarn

added 1 package in 4s
npm notice 
npm notice New patch version of npm available! 9.8.0 -> 9.8.1
npm notice Changelog: https://github.com/npm/cli/releases/tag/v9.8.1
npm notice Run npm install -g npm@9.8.1 to update!
npm notice 
Reshimming asdf nodejs...
```

## Setting up NodeJS on Windows

Go [here](https://nodejs.org/en/download) and download the installer. During installation, check the box that also installs Chocolatey for an easy install.

If you get the error

```
error Error: ENOENT: no such file or directory, copyfile 'C:\Users\Amos Ng\AppData\Local\Yarn\Cache\v6\npm-@tsconfig-node10-1.0.9-df4907fc07a886922637b15e02d4cebc4c0021b2-integrity\node_modules\@tsconfig\node10\package.json' -> 'C:\Users\Amos Ng\Documents\projects\zamm-dev\zamm\node_modules\@tsconfig\node10\package.json'
```

while doing `yarn install`, try deleting `C:\Users\Amos Ng\AppData\Local\Yarn\Cache`. We see from [this answer](https://github.com/yarnpkg/yarn/issues/5275#issuecomment-377153206) that we can also just run `yarn cache clean`.

## Setting up NodeJS on Mac for ZAMM

### Version 20.5.1

When we try NodeJS version 20.5.1, we run into this error:

```
Traceback (most recent call last):
  File "/Users/amos/.asdf/installs/nodejs/20.5.1/lib/node_modules/npm/node_modules/node-gyp/gyp/gyp_main.py", line 42, in <module>
    import gyp  # noqa: E402
    ^^^^^^^^^^
  File "/Users/amos/.asdf/installs/nodejs/20.5.1/lib/node_modules/npm/node_modules/node-gyp/gyp/pylib/gyp/__init__.py", line 9, in <module>
    import gyp.input
  File "/Users/amos/.asdf/installs/nodejs/20.5.1/lib/node_modules/npm/node_modules/node-gyp/gyp/pylib/gyp/input.py", line 19, in <module>
    from distutils.version import StrictVersion
ModuleNotFoundError: No module named 'distutils'
gyp ERR! configure error 
gyp ERR! stack Error: `gyp` failed with exit code: 1
```

As [this answer](https://stackoverflow.com/a/77360702) suggests:

```bash
$ pip install setuptools
Collecting setuptools
  Downloading setuptools-71.1.0-py3-none-any.whl.metadata (6.6 kB)
Downloading setuptools-71.1.0-py3-none-any.whl (2.3 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 2.3/2.3 MB 411.4 kB/s eta 0:00:00
Installing collected packages: setuptools
Successfully installed setuptools-71.1.0

[notice] A new release of pip is available: 24.0 -> 24.2
[notice] To update, run: pip3 install --upgrade pip
Reshimming asdf python...
```

There are no more problems with this version of NodeJS.

### Version 22.5.1

When we try the latest NodeJS version of 22.5.1, we run into this error:

```
gyp info spawn /Library/Developer/CommandLineTools/usr/bin/python3
gyp info spawn args [
gyp info spawn args '/Users/amos/.asdf/installs/nodejs/22.5.1/lib/node_modules/npm/node_modules/node-gyp/gyp/gyp_main.py',
gyp info spawn args 'binding.gyp',
gyp info spawn args '-f',
gyp info spawn args 'make',
gyp info spawn args '-I',
gyp info spawn args '/Users/amos/Documents/zamm/node_modules/canvas/build/config.gypi',
gyp info spawn args '-I',
gyp info spawn args '/Users/amos/.asdf/installs/nodejs/22.5.1/lib/node_modules/npm/node_modules/node-gyp/addon.gypi',
gyp info spawn args '-I',
gyp info spawn args '/Users/amos/Library/Caches/node-gyp/22.5.1/include/node/common.gypi',
gyp info spawn args '-Dlibrary=shared_library',
gyp info spawn args '-Dvisibility=default',
gyp info spawn args '-Dnode_root_dir=/Users/amos/Library/Caches/node-gyp/22.5.1',
gyp info spawn args '-Dnode_gyp_dir=/Users/amos/.asdf/installs/nodejs/22.5.1/lib/node_modules/npm/node_modules/node-gyp',
gyp info spawn args '-Dnode_lib_file=/Users/amos/Library/Caches/node-gyp/22.5.1/<(target_arch)/node.lib',
gyp info spawn args '-Dmodule_root_dir=/Users/amos/Documents/zamm/node_modules/canvas',
gyp info spawn args '-Dnode_engine=v8',
gyp info spawn args '--depth=.',
gyp info spawn args '--no-parallel',
gyp info spawn args '--generator-output',
gyp info spawn args 'build',
gyp info spawn args '-Goutput_dir=.'
gyp info spawn args ]
/bin/sh: pkg-config: command not found
gyp: Call to 'pkg-config pixman-1 --libs' returned exit status 127 while in binding.gyp. while trying to load binding.gyp
gyp ERR! configure error 
gyp ERR! stack Error: `gyp` failed with exit code: 1
gyp ERR! stack at ChildProcess.<anonymous> (/Users/amos/.asdf/installs/nodejs/22.5.1/lib/node_modules/npm/node_modules/node-gyp/lib/configure.js:297:18)
gyp ERR! stack at ChildProcess.emit (node:events:520:28)
gyp ERR! stack at ChildProcess._handle.onexit (node:internal/child_process:294:12)
```

From [this answer](https://stackoverflow.com/a/29443025), we see that we could try installing these dependencies first:

```bash
$ brew install zeromq
$ brew install pkgconfig
```

When we try to `make` again, we see

```
Package pixman-1 was not found in the pkg-config search path.
Perhaps you should add the directory containing `pixman-1.pc'
to the PKG_CONFIG_PATH environment variable
No package 'pixman-1' found
gyp: Call to 'pkg-config pixman-1 --libs' returned exit status 1 while in binding.gyp. while trying to load binding.gyp
```

We see from [this answer](https://stackoverflow.com/a/64562622) that the required dependencies may be:

```bash
$ brew install pkg-config cairo pango libpng jpeg giflib librsvg
```

The next failure is

```
gyp info spawn args [ 'BUILDTYPE=Release', '-C', 'build' ]
  SOLINK_MODULE(target) Release/canvas-postbuild.node
  CXX(target) Release/obj.target/canvas/src/backend/Backend.o
In file included from ../src/backend/Backend.cc:1:
In file included from ../src/backend/Backend.h:6:
../../nan/nan.h:2548:8: error: no matching member function for call to 'SetAccessor'
  tpl->SetAccessor(
  ~~~~~^~~~~~~~~~~
/Users/amos/Library/Caches/node-gyp/22.5.1/include/node/v8-template.h:1055:8: note: candidate function not viable: no known conversion from 'v8::AccessControl' to 'PropertyAttribute' for 5th argument
  void SetAccessor(
       ^
/Users/amos/Library/Caches/node-gyp/22.5.1/include/node/v8-template.h:1049:8: note: candidate function not viable: no known conversion from 'imp::NativeGetter' (aka 'void (*)(v8::Local<v8::Name>, const v8::PropertyCallbackInfo<v8::Value> &)') to 'AccessorGetterCallback' (aka 'void (*)(Local<String>, const PropertyCallbackInfo<Value> &)') for 2nd argument
  void SetAccessor(
       ^
In file included from ../src/backend/Backend.cc:1:
In file included from ../src/backend/Backend.h:6:
../../nan/nan.h:2594:8: error: no matching member function for call to 'SetAccessor'
  tpl->SetAccessor(
  ~~~~~^~~~~~~~~~~
/Users/amos/Library/Caches/node-gyp/22.5.1/include/node/v8-template.h:1055:8: note: candidate function not viable: no known conversion from 'v8::AccessControl' to 'PropertyAttribute' for 5th argument
  void SetAccessor(
       ^
/Users/amos/Library/Caches/node-gyp/22.5.1/include/node/v8-template.h:1049:8: note: candidate function not viable: no known conversion from 'imp::NativeGetter' (aka 'void (*)(v8::Local<v8::Name>, const v8::PropertyCallbackInfo<v8::Value> &)') to 'AccessorGetterCallback' (aka 'void (*)(Local<String>, const PropertyCallbackInfo<Value> &)') for 2nd argument
  void SetAccessor(
       ^
In file included from ../src/backend/Backend.cc:1:
../src/backend/Backend.h:60:14: warning: private field 'backend' is not used [-Wunused-private-field]
    Backend* backend;
             ^
1 warning and 2 errors generated.
make[2]: *** [Release/obj.target/canvas/src/backend/Backend.o] Error 1
gyp ERR! build error 
gyp ERR! stack Error: `make` failed with exit code: 2
gyp ERR! stack at ChildProcess.<anonymous> (/Users/amos/.asdf/installs/nodejs/22.5.1/lib/node_modules/npm/node_modules/node-gyp/lib/build.js:209:23)
gyp ERR! System Darwin 23.5.0
gyp ERR! command "/Users/amos/.asdf/installs/nodejs/22.5.1/bin/node" "/Users/amos/.asdf/installs/nodejs/22.5.1/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js" "build" "--fallback-to-build" "--update-binary" "--module=/Users/amos/Documents/zamm/node_modules/canvas/build/Release/canvas.node" "--module_name=canvas" "--module_path=/Users/amos/Documents/zamm/node_modules/canvas/build/Release" "--napi_version=9" "--node_abi_napi=napi" "--napi_build_version=0" "--node_napi_label=node-v127"
gyp ERR! cwd /Users/amos/Documents/zamm/node_modules/canvas
```
