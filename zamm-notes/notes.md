# Copying over notes from the old repo

We find that there is [this method](https://unix.stackexchange.com/a/107647) for copying over files of a specific type while preserving the directory structure:

```bash
$ find SOURCE_DIRECTORY -name '*.csv' -exec cp --parents \{\} TARGET_DIRECTORY \;
```

Afterwards, we need to update the links ourselves.
