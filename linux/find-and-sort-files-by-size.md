# Find files by type and sort them by size

`find` locates files, but sorting by size needs `du` (or `xargs`) piped into `sort -h`. The `-exec ... +` form is safer than piping through `xargs` because it doesn't break on filenames with spaces.

## Details

Find all `.log` files in a directory and list them sorted from largest to smallest:

```bash
find /var/log -type f -name "*.log" -exec du -h {} + | sort -rh
```

Flags explained:
- `-type f` — only regular files, not directories
- `-name "*.log"` — match by extension
- `-exec du -h {} +` — run `du -h` once on the whole batch of matched files (`{}` is replaced by the file list, `+` batches them instead of calling `du` once per file, which is faster)
- `sort -rh` — `-h` sorts "human-readable" sizes correctly (so `1.2M` sorts after `800K`, which plain `sort -r` gets wrong), `-r` reverses to largest-first

Only the top 5 biggest files:

```bash
find /var/log -type f -name "*.log" -exec du -h {} + | sort -rh | head -5
```

## Why it matters

The `xargs` version of this command is more commonly taught:

```bash
find /var/log -name "*.log" | xargs du -h | sort -rh
```

But it silently breaks on filenames containing spaces — `xargs` splits on whitespace by default, so `my report.log` becomes two separate (nonexistent) arguments and `du` fails on both. Tested this directly:

```
$ find . -name "*.log" | xargs du -h
du: cannot access 'my': No such file or directory
du: cannot access 'report.log': No such file or directory
```

Two fixes: use `find -exec ... +` (no pipe needed at all), or if you specifically need `xargs`, pair it with null-delimited output: `find ... -print0 | xargs -0 du -h`.

## Source

Tested locally with `find`, `du`, `xargs`, `sort` man pages