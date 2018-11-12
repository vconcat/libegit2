# libgit2 bindings for Emacs

[![Travis build status](https://travis-ci.org/magit/libegit2.svg?branch=master "Build Status")](https://travis-ci.org/magit/libegit2)
[![Appveyor build status](https://ci.appveyor.com/api/projects/status/jq45m47uubafv9kq/branch/master?svg=true)](https://ci.appveyor.com/project/tarsius/libegit2/branch/master)

This is an *experimental* module for libgit2 bindings to Emacs, intended to boost the performance of
[magit](https://github.com/magit/magit).

Other work in this direction:
- [ksjogo/emacs-libgit2](https://github.com/ksjogo/emacs-libgit2) in C, has been dormant for more
  than a year.
- [ubolonton/magit-libgit2](https://github.com/ubolonton/magit-libgit2) in Rust.

This module is written in C, and aims to be a thin wrapper around libgit2. That means that all
functions in the [libgit2 reference](https://libgit2.github.com/libgit2/#HEAD) should translate
more-or-less directly to Emacs, in the following sense:

- Function names are the same, except with underscores replaced by hyphens. The prefix is changed
  from `git-` to `libgit-`.
- Predicate functions are given a `-p` suffix, and words like "is" are removed,
  e.g. `git_repository_is_bare` becomes `libgit-repository-bare-p`.
- Output parameters become return values.
- Error codes become error signals (type `giterr`).
- Return types map to their natural Emacs counterparts, or opaque user pointers when not applicable
  (e.g. for `git-???` structures). Exceptions: `git-oid` and `git-buf` types are converted to Emacs
  strings.
- Boolean parameters or pointers towards the end of argument lists whose natural default value is
  false or NULL will be made optional.

Quality-of-life convenience functionality is better implemented in Emacs Lisp than in C.

## Building

There is a loader file written in Emacs Lisp that will build the module for you, but the
`git submodule` steps need to be run manually.

```
git submodule init
git submodule update
mkdir build
cd build
cmake ..
make
```

If you're on OSX and using Macports, you may need to set `CMAKE_PREFIX_PATH` to avoid linking
against the wrong libiconv. For example,

```
cmake -DCMAKE_PREFIX_PATH=/opt/local ..
```

## Testing

Ensure that you have [Cask](https://github.com/cask/cask) installed.

```
cask install
cd build
make test
```

To see more output for debugging new tests you can specify more verbose output.

```
make test ARGS=-V
```

## Using

Ensure that `libgit.el` is somewhere in your load path. Then

```elisp
(require 'libgit)
```

If the dynamic module was not already built, you should be asked to do it manually.

If you use [Borg](https://github.com/emacscollective/borg), the following `.gitmodules` entry should
work.

```
[submodule "libegit2"]
    path = lib/libegit2
    url = git@github.com:TheBB/libegit2.git
    build-step = git submodule init
    build-step = git submodule update
    build-step = mkdir -p build
    build-step = cd build && cmake ..
    build-step = cd build && make
```

## Contributing

### Adding a function

1. Find the section that the function belongs to (i.e. `git_SECTION_xyz`).
2. Create, if necessary, `src/egit-SECTION.h` and `src/egit-SECTION.c`.
3. In `src/egit-SECTION.h`, declare the function with `EGIT_DEFUN`. See existing headers for
   examples.
4. In `src/egit-SECTION.c`, document the function with `EGIT_DOC`. See existing files for examples.
5. In `src/egit-SECTION.c`, implement the function. See existing files for examples.
   1. Always check argument types in the beginning. Use `EGIT_ASSERT` for this. These macros may return.
   2. Then, extract the data needed from `emacs_value`. This may involve allocating buffers for strings.
   3. Call the `libgit2` backend function.
   4. Free any memory you might need to free that was allocated in step 2.
   5. Check the error code if applicable with `EGIT_CHECK_ERROR`. This macro may return.
   6. Create return value and return.
6. In `src/egit.c`, create a `DEFUN` call in `egit_init`. You may need to include a new header.

### Adding a type

Sometimes a struct of type `git_???` may need to be returned to Emacs as an opaque user pointer.
To do this, we use a wrapper structure with a type information tag.

Usually, objects that belong to a repository need to keep the repository alive until after they are
freed. To do this, we use a hash table with reference counting semantics for repositories to ensure
that none of them are freed out of turn.

1. In `src/egit.h` add an entry to the `egit_type` enum for the new type.
2. In `src/egit.h` ass a new `EGIT_ASSERT` macro for the new type.
3. In `src/egit.c` add a new entry to the `egit_finalize` switch statement to free a
   structure. If the new structure needs to keep a repository alive (usually the "owner" in libgit2
   terms), also call `egit_decref_repository` on these (see existing code for examples).
4. In `src/egit.c` add a new entry to the `egit_wrap` switch statement to increase the reference
   counts of the repository if it must be kept alive.
5. In `src/egit.c` add a new entry to the `egit_typeof` switch statement.
6. In `src/egit.c` add a new `egit_TYPE_p` predicate function.
7. In `src/egit.c` create a `DEFUN` call in `egit_init` for the predicate function.
8. In `interface.h` add two new symbols, `TYPE-p` and `TYPE`.
9. In `interface.c` initialize those symbols in the `em_init` function.

## Function list

This is a complete list of functions in libgit2. It therefore serves more or less as an upper bound
on the amount of work needed.

Legend:
- :heavy_check_mark: Function is implemented
- :x: Function should probably not be implemented (reason given)
- :interrobang: Undecided

Some functions are defined in libgit2 headers in the `sys` subdirectory, and are not reachable from
a standard include (i.e. `#include "git2.h"`). For now, we will skip those on the assumption that
they are more specialized.

Estimates (updated periodically):
- Implemented: 165 (21.2%)
- Should not implement: 145 (18.6%)
- To do: 468 (60.1%)
- Total: 778

### extra

These are functions that do not have a `libgit2` equivalent.

Type checkers and predicates:

- :heavy_check_mark: `git-typeof`
- :heavy_check_mark: `git-blame-p`
- :heavy_check_mark: `git-commit-p`
- :heavy_check_mark: `git-diff-p`
- :heavy_check_mark: `git-diff-delta-p`
- :heavy_check_mark: `git-diff-binary-p`
- :heavy_check_mark: `git-diff-hunk-p`
- :heavy_check_mark: `git-diff-line-p`
- :heavy_check_mark: `git-index-p`
- :heavy_check_mark: `git-index-entry-p`
- :heavy_check_mark: `git-object-p`
- :heavy_check_mark: `git-reference-p`
- :heavy_check_mark: `git-repository-p`
- :heavy_check_mark: `git-signature-p`
- :heavy_check_mark: `git-reference-direct-p`
- :heavy_check_mark: `git-reference-symbolic-p`
- :heavy_check_mark: `git-transaction-p`
- :heavy_check_mark: `git-tree-p`

Getters for public structs:

- :heavy_check_mark: `git-diff-delta-file-id`
- :heavy_check_mark: `git-diff-delta-file-path`
- :heavy_check_mark: `git-diff-delta-nfiles`
- :heavy_check_mark: `git-diff-delta-similarity`
- :heavy_check_mark: `git-diff-delta-status`
- :heavy_check_mark: `git-diff-delta-file-exists-p`
- :heavy_check_mark: `git-diff-hunk-header`
- :heavy_check_mark: `git-diff-hunk-lines`
- :heavy_check_mark: `git-diff-hunk-start`
- :heavy_check_mark: `git-diff-line-origin`
- :heavy_check_mark: `git-diff-line-lineno`
- :heavy_check_mark: `git-diff-line-content`
- :heavy_check_mark: `git-index-entry-path`
- :heavy_check_mark: `git-signature-name`
- :heavy_check_mark: `git-signature-email`
- :heavy_check_mark: `git-signature-time`

Iterators converted to map functions:

- :heavy_check_mark: `git-index-conflict-foreach`

### annotated

- :x: `git-annotated-commit-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-annotated-commit-from-fetchhead`
- :interrobang: `git-annotated-commit-from-ref`
- :interrobang: `git-annotated-commit-from-revspec`
- :interrobang: `git-annotated-commit-id`
- :interrobang: `git-annotated-commit-lookup`

### attr

- :interrobang: `git-attr-add-macro`
- :interrobang: `git-attr-cache-flush`
- :interrobang: `git-attr-foreach`
- :interrobang: `git-attr-get`
- :interrobang: `git-attr-get-many`
- :interrobang: `git-attr-value`

### blame

- :interrobang: `git-blame-buffer`
- :heavy_check_mark: `git-blame-file`
- :x: `git-blame-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-blame-get-hunk-byindex`
- :heavy_check_mark: `git-blame-get-hunk-byline`
- :heavy_check_mark: `git-blame-get-hunk-count`
- :x: `git-blame-init-options` (options are represented by an `alist`)

### blob

- :interrobang: `git-blob-create-frombuffer`
- :interrobang: `git-blob-create-fromdisk`
- :interrobang: `git-blob-create-fromstream`
- :interrobang: `git-blob-create-fromstream-commit`
- :interrobang: `git-blob-create-fromworkdir`
- :interrobang: `git-blob-dup`
- :interrobang: `git-blob-filtered-content`
- :x: `git-blob-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-blob-id`
- :interrobang: `git-blob-is-binary`
- :interrobang: `git-blob-lookup`
- :interrobang: `git-blob-lookup-prefix`
- :interrobang: `git-blob-owner`
- :interrobang: `git-blob-rawcontent`
- :interrobang: `git-blob-rawsize`

### branch

- :heavy_check_mark: `git-branch-create`
- :heavy_check_mark: `git-branch-create-from-annotated`
- :heavy_check_mark: `git-branch-delete`
- :heavy_check_mark: `git-branch-is-checked-out`
- :heavy_check_mark: `git-branch-is-head`
- :interrobang: `git-branch-iterator-free`
- :interrobang: `git-branch-iterator-new`
- :heavy_check_mark: `git-branch-lookup`
- :interrobang: `git-branch-move`
- :heavy_check_mark: `git-branch-name`
- :interrobang: `git-branch-next`
- :heavy_check_mark: `git-branch-remote-name`
- :interrobang: `git-branch-set-upstream`
- :heavy_check_mark: `git-branch-upstream`
- :heavy_check_mark: `git-branch-upstream-name`
- :heavy_check_mark: `git-branch-upstream-remote`

### buf

Probably none of these functions are necessary, since we can expose buffers to Emacs as strings.

- :x: `git-buf-contains-nul`
- :x: `git-buf-free` (memory management shouldn't be exposed to Emacs)
- :x: `git-buf-grow`
- :x: `git-buf-is-binary`
- :x: `git-buf-set`

### checkout

- :interrobang: `git-checkout-head`
- :interrobang: `git-checkout-index`
- :interrobang: `git-checkout-init-options`
- :interrobang: `git-checkout-tree`

### cherrypick

- :interrobang: `git-cherrypick`
- :interrobang: `git-cherrypick-commit`
- :interrobang: `git-cherrypick-init-options`

### clone

- :heavy_check_mark: `git-clone`
- :interrobang: `git-clone-init-options`

### commit

- :interrobang: `git-commit-amend`
- :heavy_check_mark: `git-commit-author`
- :heavy_check_mark: `git-commit-body`
- :heavy_check_mark: `git-commit-committer`
- :interrobang: `git-commit-create`
- :interrobang: `git-commit-create-buffer`
- :x: `git-commit-create-from-callback` (in `sys`)
- :x: `git-commit-create-from-ids` (in `sys`)
- :interrobang: `git-commit-create-v`
- :interrobang: `git-commit-create-with-signature`
- :interrobang: `git-commit-dup`
- :interrobang: `git-commit-extract-signature`
- :x: `git-commit-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-commit-header-field`
- :heavy_check_mark: `git-commit-id`
- :heavy_check_mark: `git-commit-lookup`
- :heavy_check_mark: `git-commit-lookup-prefix`
- :heavy_check_mark: `git-commit-message`
- :interrobang: `git-commit-message-encoding`
- :interrobang: `git-commit-message-raw`
- :heavy_check_mark: `git-commit-nth-gen-ancestor`
- :heavy_check_mark: `git-commit-owner`
- :heavy_check_mark: `git-commit-parent`
- :heavy_check_mark: `git-commit-parent-id`
- :heavy_check_mark: `git-commit-parentcount`
- :interrobang: `git-commit-raw-header`
- :heavy_check_mark: `git-commit-summary`
- :heavy_check_mark: `git-commit-time`
- :x: `git-commit-time-offset` (included in `git-commit-time`)
- :heavy_check_mark: `git-commit-tree`
- :heavy_check_mark: `git-commit-tree-id`

### config

- :x: `git-config-add-backend` (in `sys`)
- :interrobang: `git-config-add-file-ondisk`
- :interrobang: `git-config-backend-foreach-match`
- :interrobang: `git-config-delete-entry`
- :interrobang: `git-config-delete-multivar`
- :x: `git-config-entry-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-config-find-global`
- :heavy_check_mark: `git-config-find-programdata`
- :heavy_check_mark: `git-config-find-system`
- :heavy_check_mark: `git-config-find-xdg`
- :interrobang: `git-config-foreach`
- :interrobang: `git-config-foreach-match`
- :x: `git-config-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-config-get-bool`
- :interrobang: `git-config-get-entry`
- :x: `git-config-get-int32` (don't need different integer types)
- :heavy_check_mark: `git-config-get-int64` (as `-int`)
- :interrobang: `git-config-get-mapped`
- :interrobang: `git-config-get-multivar-foreach`
- :heavy_check_mark: `git-config-get-path`
- :heavy_check_mark: `git-config-get-string`
- :x: `git-config-get-string-buf` (probably fine with just `-get-string`)
- :x: `git-config-init-backend` (in `sys`)
- :x: `git-config-iterator-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-config-iterator-glob-new`
- :interrobang: `git-config-iterator-new`
- :heavy_check_mark: `git-config-lock`
- :interrobang: `git-config-lookup-map-value`
- :interrobang: `git-config-multivar-iterator-new`
- :interrobang: `git-config-new`
- :interrobang: `git-config-next`
- :interrobang: `git-config-open-default`
- :interrobang: `git-config-open-global`
- :interrobang: `git-config-open-level`
- :interrobang: `git-config-open-ondisk`
- :interrobang: `git-config-parse-bool`
- :interrobang: `git-config-parse-int32`
- :interrobang: `git-config-parse-int64`
- :interrobang: `git-config-parse-path`
- :heavy_check_mark: `git-config-set-bool`
- :x: `git-config-set-int32` (don't need different integer types)
- :heavy_check_mark: `git-config-set-int64` (as `-int`)
- :interrobang: `git-config-set-multivar`
- :heavy_check_mark: `git-config-set-string`
- :interrobang: `git-config-snapshot`

### cred

- :interrobang: `git-cred-default-new`
- :x: `git-cred-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-cred-has-username`
- :interrobang: `git-cred-ssh-custom-new`
- :interrobang: `git-cred-ssh-interactive-new`
- :interrobang: `git-cred-ssh-key-from-agent`
- :interrobang: `git-cred-ssh-key-memory-new`
- :interrobang: `git-cred-ssh-key-new`
- :interrobang: `git-cred-username-new`
- :interrobang: `git-cred-userpass`
- :interrobang: `git-cred-userpass-plaintext-new`

### describe

- :heavy_check_mark: `git-describe-commit`
- :x: `git-describe-format` (we return strings immediately)
- :x: `git-describe-result-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-describe-workdir`

### diff

- :interrobang: `git-diff-blob-to-buffer`
- :interrobang: `git-diff-blobs`
- :interrobang: `git-diff-buffers`
- :interrobang: `git-diff-commit-as-email`
- :interrobang: `git-diff-find-init-options`
- :interrobang: `git-diff-find-similar`
- :heavy_check_mark: `git-diff-foreach`
- :interrobang: `git-diff-format-email`
- :interrobang: `git-diff-format-email-init-options`
- :x: `git-diff-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-diff-from-buffer`
- :heavy_check_mark: `git-diff-get-delta`
- :x: `git-diff-get-perfdata` (in `sys`)
- :interrobang: `git-diff-get-stats`
- :heavy_check_mark: `git-diff-index-to-index`
- :heavy_check_mark: `git-diff-index-to-workdir`
- :interrobang: `git-diff-init-options`
- :interrobang: `git-diff-is-sorted-icase`
- :interrobang: `git-diff-merge`
- :heavy_check_mark: `git-diff-num-deltas`
- :heavy_check_mark: `git-diff-num-deltas-of-type` (use `git-diff-num-deltas`)
- :interrobang: `git-diff-patchid`
- :interrobang: `git-diff-patchid-init-options`
- :heavy_check_mark: `git-diff-print`
- :x: `git-diff-print-callback--to-buf` (in `sys`)
- :interrobang: `git-diff-print-callback--to-file-handle`
- :interrobang: `git-diff-stats-deletions`
- :interrobang: `git-diff-stats-files-changed`
- :x: `git-diff-stats-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-diff-stats-insertions`
- :interrobang: `git-diff-stats-to-buf`
- :interrobang: `git-diff-status-char`
- :interrobang: `git-diff-to-buf`
- :heavy_check_mark: `git-diff-tree-to-index`
- :heavy_check_mark: `git-diff-tree-to-tree`
- :heavy_check_mark: `git-diff-tree-to-workdir`
- :heavy_check_mark: `git-diff-tree-to-workdir-with-index`

### fetch

- :interrobang: `git-fetch-init-options`

### filter

- :x: `git-filter-init` (in `sys`)
- :interrobang: `git-filter-list-apply-to-blob`
- :interrobang: `git-filter-list-apply-to-data`
- :interrobang: `git-filter-list-apply-to-file`
- :interrobang: `git-filter-list-contains`
- :x: `git-filter-list-free` (memory management shouldn't be exposed to Emacs)
- :x: `git-filter-list-length` (in `sys`)
- :interrobang: `git-filter-list-load`
- :x: `git-filter-list-new` (in `sys`)
- :x: `git-filter-list-push` (in `sys`)
- :interrobang: `git-filter-list-stream-blob`
- :interrobang: `git-filter-list-stream-data`
- :interrobang: `git-filter-list-stream-file`
- :x: `git-filter-lookup` (in `sys`)
- :x: `git-filter-register` (in `sys`)
- :x: `git-filter-source-filemode` (in `sys`)
- :x: `git-filter-source-flags` (in `sys`)
- :x: `git-filter-source-id` (in `sys`)
- :x: `git-filter-source-mode` (in `sys`)
- :x: `git-filter-source-path` (in `sys`)
- :x: `git-filter-source-repo` (in `sys`)
- :x: `git-filter-unregister` (in `sys`)

### giterr

Probably none of these functions will be necessary, since we expose errors to Emacs as signals.

- :x: `giterr-clear`
- :x: `giterr-last`
- :x: `giterr-set-oom`
- :x: `giterr-set-str`

### graph

- :interrobang: `git-graph-ahead-behind`
- :interrobang: `git-graph-descendant-of`

### hashsig

- :x: `git-hashsig-compare` (in `sys`)
- :x: `git-hashsig-create` (in `sys`)
- :x: `git-hashsig-create-fromfile` (in `sys`)
- :x: `git-hashsig-free` (memory management shouldn't be exposed to Emacs)

### ignore

- :heavy_check_mark: `git-ignore-add-rule`
- :heavy_check_mark: `git-ignore-clear-internal-rules`
- :heavy_check_mark: `git-ignore-path-is-ignored`

### index

- :interrobang: `git-index-add`
- :interrobang: `git-index-add-all`
- :interrobang: `git-index-add-bypath`
- :interrobang: `git-index-add-frombuffer`
- :heavy_check_mark: `git-index-caps`
- :heavy_check_mark: `git-index-checksum`
- :interrobang: `git-index-clear`
- :interrobang: `git-index-conflict-add`
- :interrobang: `git-index-conflict-cleanup`
- :heavy_check_mark: `git-index-conflict-get`
- :heavy_check_mark: `git-index-conflict-iterator-free` (use `git-index-conflict-foreach`)
- :heavy_check_mark: `git-index-conflict-iterator-new` (use `git-index-conflict-foreach`)
- :heavy_check_mark: `git-index-conflict-next` (use `git-index-conflict-foreach`)
- :interrobang: `git-index-conflict-remove`
- :heavy_check_mark: `git-index-entry-is-conflict` (use `git-index-entry-stage`)
- :heavy_check_mark: `git-index-entry-stage`
- :heavy_check_mark: `git-index-entrycount`
- :interrobang: `git-index-find`
- :interrobang: `git-index-find-prefix`
- :x: `git-index-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-index-get-byindex`
- :heavy_check_mark: `git-index-get-bypath`
- :heavy_check_mark: `git-index-has-conflicts` (as `git-index-conflicts-p`)
- :interrobang: `git-index-new`
- :interrobang: `git-index-open`
- :heavy_check_mark: `git-index-owner`
- :heavy_check_mark: `git-index-path`
- :interrobang: `git-index-read`
- :interrobang: `git-index-read-tree`
- :interrobang: `git-index-remove`
- :interrobang: `git-index-remove-all`
- :interrobang: `git-index-remove-bypath`
- :interrobang: `git-index-remove-directory`
- :interrobang: `git-index-set-caps`
- :interrobang: `git-index-set-version`
- :interrobang: `git-index-update-all`
- :heavy_check_mark: `git-index-version`
- :interrobang: `git-index-write`
- :interrobang: `git-index-write-tree`
- :interrobang: `git-index-write-tree-to`

### indexer

- :interrobang: `git-indexer-append`
- :interrobang: `git-indexer-commit`
- :x: `git-indexer-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-indexer-hash`
- :interrobang: `git-indexer-new`

### libgit2

- :interrobang: `git-libgit2-features`
- :interrobang: `git-libgit2-init`
- :interrobang: `git-libgit2-opts`
- :interrobang: `git-libgit2-shutdown`
- :interrobang: `git-libgit2-version`

### mempack

- :x: `git-mempack-dump` (in `sys`)
- :x: `git-mempack-new` (in `sys`)
- :x: `git-mempack-reset` (in `sys`)

### merge

- :interrobang: `git-merge`
- :interrobang: `git-merge-analysis`
- :interrobang: `git-merge-base`
- :interrobang: `git-merge-base-many`
- :interrobang: `git-merge-base-octopus`
- :interrobang: `git-merge-bases`
- :interrobang: `git-merge-bases-many`
- :interrobang: `git-merge-commits`
- :interrobang: `git-merge-file`
- :interrobang: `git-merge-file-from-index`
- :interrobang: `git-merge-file-init-input`
- :interrobang: `git-merge-file-init-options`
- :interrobang: `git-merge-file-result-free`
- :interrobang: `git-merge-init-options`
- :interrobang: `git-merge-trees`

### message

- :interrobang: `git-message-prettify`
- :interrobang: `git-message-trailer-array-free`
- :interrobang: `git-message-trailers`

### note

- :interrobang: `git-note-author`
- :interrobang: `git-note-commit-create`
- :interrobang: `git-note-commit-iterator-new`
- :interrobang: `git-note-commit-read`
- :interrobang: `git-note-commit-remove`
- :interrobang: `git-note-committer`
- :interrobang: `git-note-create`
- :interrobang: `git-note-foreach`
- :x: `git-note-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-note-id`
- :interrobang: `git-note-iterator-free`
- :interrobang: `git-note-iterator-new`
- :interrobang: `git-note-message`
- :interrobang: `git-note-next`
- :interrobang: `git-note-read`
- :interrobang: `git-note-remove`

### object

- :x: `git-object--size` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-object-dup`
- :x: `git-object-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-object-id`
- :interrobang: `git-object-lookup`
- :heavy_check_mark: `git-object-lookup-bypath`
- :heavy_check_mark: `git-object-lookup-prefix`
- :heavy_check_mark: `git-object-owner`
- :interrobang: `git-object-peel`
- :heavy_check_mark: `git-object-short-id`
- :x: `git-object-string2type` (see below)
- :x: `git-object-type` (can be covered by a more general `git-typeof` for all opaque user pointers)
- :x: `git-object-type2string` (see above)
- :interrobang: `git-object-typeisloose`

### odb

- :interrobang: `git-odb-add-alternate`
- :interrobang: `git-odb-add-backend`
- :interrobang: `git-odb-add-disk-alternate`
- :interrobang: `git-odb-backend-loose`
- :interrobang: `git-odb-backend-one-pack`
- :interrobang: `git-odb-backend-pack`
- :interrobang: `git-odb-exists`
- :interrobang: `git-odb-exists-prefix`
- :interrobang: `git-odb-expand-ids`
- :interrobang: `git-odb-foreach`
- :x: `git-odb-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-odb-get-backend`
- :interrobang: `git-odb-hash`
- :interrobang: `git-odb-hashfile`
- :x: `git-odb-init-backend` (in `sys`)
- :interrobang: `git-odb-new`
- :interrobang: `git-odb-num-backends`
- :interrobang: `git-odb-object-data`
- :interrobang: `git-odb-object-dup`
- :x: `git-odb-object-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-odb-object-id`
- :interrobang: `git-odb-object-size`
- :interrobang: `git-odb-object-type`
- :interrobang: `git-odb-open`
- :interrobang: `git-odb-open-rstream`
- :interrobang: `git-odb-open-wstream`
- :interrobang: `git-odb-read`
- :interrobang: `git-odb-read-header`
- :interrobang: `git-odb-read-prefix`
- :interrobang: `git-odb-refresh`
- :interrobang: `git-odb-stream-finalize-write`
- :x: `git-odb-stream-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-odb-stream-read`
- :interrobang: `git-odb-stream-write`
- :interrobang: `git-odb-write`
- :interrobang: `git-odb-write-pack`

### oid

Probably none of these functions will be necessary, since we can expose OIDs to Emacs as strings.

- :x: `git-oid-cmp`
- :x: `git-oid-cpy`
- :x: `git-oid-equal`
- :x: `git-oid-fmt`
- :x: `git-oid-fromraw`
- :x: `git-oid-fromstr`
- :x: `git-oid-fromstrn`
- :x: `git-oid-fromstrp`
- :x: `git-oid-iszero`
- :x: `git-oid-ncmp`
- :x: `git-oid-nfmt`
- :x: `git-oid-pathfmt`
- :x: `git-oid-shorten-add`
- :x: `git-oid-shorten-free`
- :x: `git-oid-shorten-new`
- :x: `git-oid-strcmp`
- :x: `git-oid-streq`
- :x: `git-oid-tostr`
- :x: `git-oid-tostr-s`

### oidarray

- :x: `git-oidarray-free` (memory management shouldn't be exposed to Emacs)

### openssl

- :x: `git-openssl-set-locking` (in `sys`)

### packbuilder

- :interrobang: `git-packbuilder-foreach`
- :x: `git-packbuilder-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-packbuilder-hash`
- :interrobang: `git-packbuilder-insert`
- :interrobang: `git-packbuilder-insert-commit`
- :interrobang: `git-packbuilder-insert-recur`
- :interrobang: `git-packbuilder-insert-tree`
- :interrobang: `git-packbuilder-insert-walk`
- :interrobang: `git-packbuilder-new`
- :interrobang: `git-packbuilder-object-count`
- :interrobang: `git-packbuilder-set-callbacks`
- :interrobang: `git-packbuilder-set-threads`
- :interrobang: `git-packbuilder-write`
- :interrobang: `git-packbuilder-written`

### patch

- :x: `git-patch-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-patch-from-blob-and-buffer`
- :interrobang: `git-patch-from-blobs`
- :interrobang: `git-patch-from-buffers`
- :interrobang: `git-patch-from-diff`
- :interrobang: `git-patch-get-delta`
- :interrobang: `git-patch-get-hunk`
- :interrobang: `git-patch-get-line-in-hunk`
- :interrobang: `git-patch-line-stats`
- :interrobang: `git-patch-num-hunks`
- :interrobang: `git-patch-num-lines-in-hunk`
- :interrobang: `git-patch-print`
- :interrobang: `git-patch-size`
- :interrobang: `git-patch-to-buf`

### pathspec

- :x: `git-pathspec-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-pathspec-match-diff`
- :interrobang: `git-pathspec-match-index`
- :interrobang: `git-pathspec-match-list-diff-entry`
- :interrobang: `git-pathspec-match-list-entry`
- :interrobang: `git-pathspec-match-list-entrycount`
- :interrobang: `git-pathspec-match-list-failed-entry`
- :interrobang: `git-pathspec-match-list-failed-entrycount`
- :x: `git-pathspec-match-list-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-pathspec-match-tree`
- :interrobang: `git-pathspec-match-workdir`
- :interrobang: `git-pathspec-matches-path`
- :interrobang: `git-pathspec-new`

### proxy

- :interrobang: `git-proxy-init-options`

### push

- :interrobang: `git-push-init-options`

### rebase

- :interrobang: `git-rebase-abort`
- :interrobang: `git-rebase-commit`
- :interrobang: `git-rebase-finish`
- :x: `git-rebase-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-rebase-init`
- :interrobang: `git-rebase-init-options`
- :interrobang: `git-rebase-inmemory-index`
- :interrobang: `git-rebase-next`
- :interrobang: `git-rebase-open`
- :interrobang: `git-rebase-operation-byindex`
- :interrobang: `git-rebase-operation-current`
- :interrobang: `git-rebase-operation-entrycount`

### refdb

- :x: `git-refdb-backend-fs` (in `sys`)
- :interrobang: `git-refdb-compress`
- :x: `git-refdb-free` (memory management shouldn't be exposed to Emacs)
- :x: `git-refdb-init-backend` (in `sys`)
- :interrobang: `git-refdb-new`
- :interrobang: `git-refdb-open`
- :x: `git-refdb-set-backend` (in `sys`)

### reference

- :x: `git-reference--alloc` (in `sys`)
- :x: `git-reference--alloc-symbolic` (in `sys`)
- :interrobang: `git-reference-cmp`
- :heavy_check_mark: `git-reference-create`
- :heavy_check_mark: `git-reference-create-matching`
- :heavy_check_mark: `git-reference-delete`
- :heavy_check_mark: `git-reference-dup`
- :heavy_check_mark: `git-reference-dwim`
- :heavy_check_mark: `git-reference-ensure-log`
- :interrobang: `git-reference-foreach`
- :interrobang: `git-reference-foreach-glob`
- :interrobang: `git-reference-foreach-name`
- :x: `git-reference-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-reference-has-log`
- :heavy_check_mark: `git-reference-is-branch`
- :heavy_check_mark: `git-reference-is-note`
- :heavy_check_mark: `git-reference-is-remote`
- :heavy_check_mark: `git-reference-is-tag`
- :heavy_check_mark: `git-reference-is-valid-name`
- :interrobang: `git-reference-iterator-free`
- :interrobang: `git-reference-iterator-glob-new`
- :interrobang: `git-reference-iterator-new`
- :heavy_check_mark: `git-reference-list`
- :heavy_check_mark: `git-reference-lookup`
- :heavy_check_mark: `git-reference-name`
- :heavy_check_mark: `git-reference-name-to-id`
- :interrobang: `git-reference-next`
- :interrobang: `git-reference-next-name`
- :interrobang: `git-reference-normalize-name`
- :heavy_check_mark: `git-reference-owner`
- :heavy_check_mark: `git-reference-peel`
- :heavy_check_mark: `git-reference-remove`
- :interrobang: `git-reference-rename`
- :heavy_check_mark: `git-reference-resolve`
- :interrobang: `git-reference-set-target`
- :heavy_check_mark: `git-reference-shorthand`
- :interrobang: `git-reference-symbolic-create`
- :interrobang: `git-reference-symbolic-create-matching`
- :interrobang: `git-reference-symbolic-set-target`
- :heavy_check_mark: `git-reference-symbolic-target`
- :heavy_check_mark: `git-reference-target`
- :heavy_check_mark: `git-reference-target-peel`
- :heavy_check_mark: `git-reference-type`

### reflog

- :interrobang: `git-reflog-append`
- :interrobang: `git-reflog-delete`
- :interrobang: `git-reflog-drop`
- :interrobang: `git-reflog-entry-byindex`
- :interrobang: `git-reflog-entry-committer`
- :interrobang: `git-reflog-entry-id-new`
- :interrobang: `git-reflog-entry-id-old`
- :interrobang: `git-reflog-entry-message`
- :interrobang: `git-reflog-entrycount`
- :x: `git-reflog-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-reflog-read`
- :interrobang: `git-reflog-rename`
- :interrobang: `git-reflog-write`

### refspec

- :interrobang: `git-refspec-direction`
- :interrobang: `git-refspec-dst`
- :interrobang: `git-refspec-dst-matches`
- :interrobang: `git-refspec-force`
- :interrobang: `git-refspec-rtransform`
- :interrobang: `git-refspec-src`
- :interrobang: `git-refspec-src-matches`
- :interrobang: `git-refspec-string`
- :interrobang: `git-refspec-transform`

### remote

- :interrobang: `git-remote-add-fetch`
- :interrobang: `git-remote-add-push`
- :interrobang: `git-remote-autotag`
- :interrobang: `git-remote-connect`
- :interrobang: `git-remote-connected`
- :interrobang: `git-remote-create`
- :interrobang: `git-remote-create-anonymous`
- :interrobang: `git-remote-create-detached`
- :interrobang: `git-remote-create-with-fetchspec`
- :interrobang: `git-remote-default-branch`
- :interrobang: `git-remote-delete`
- :interrobang: `git-remote-disconnect`
- :interrobang: `git-remote-download`
- :interrobang: `git-remote-dup`
- :interrobang: `git-remote-fetch`
- :x: `git-remote-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-remote-get-fetch-refspecs`
- :interrobang: `git-remote-get-push-refspecs`
- :interrobang: `git-remote-get-refspec`
- :interrobang: `git-remote-init-callbacks`
- :interrobang: `git-remote-is-valid-name`
- :interrobang: `git-remote-list`
- :interrobang: `git-remote-lookup`
- :interrobang: `git-remote-ls`
- :interrobang: `git-remote-name`
- :interrobang: `git-remote-owner`
- :interrobang: `git-remote-prune`
- :interrobang: `git-remote-prune-refs`
- :interrobang: `git-remote-push`
- :interrobang: `git-remote-pushurl`
- :interrobang: `git-remote-refspec-count`
- :interrobang: `git-remote-rename`
- :interrobang: `git-remote-set-autotag`
- :interrobang: `git-remote-set-pushurl`
- :interrobang: `git-remote-set-url`
- :interrobang: `git-remote-stats`
- :interrobang: `git-remote-stop`
- :interrobang: `git-remote-update-tips`
- :interrobang: `git-remote-upload`
- :interrobang: `git-remote-url`

### repository

- :x: `git-repository--cleanup` (in `sys`)
- :heavy_check_mark: `git-repository-commondir`
- :heavy_check_mark: `git-repository-config`
- :interrobang: `git-repository-config-snapshot`
- :heavy_check_mark: `git-repository-detach-head`
- :heavy_check_mark: `git-repository-discover`
- :interrobang: `git-repository-fetchhead-foreach`
- :x: `git-repository-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-repository-get-namespace`
- :interrobang: `git-repository-hashfile`
- :heavy_check_mark: `git-repository-head`
- :heavy_check_mark: `git-repository-head-detached`
- :heavy_check_mark: `git-repository-head-for-worktree`
- :heavy_check_mark: `git-repository-head-unborn`
- :heavy_check_mark: `git-repository-ident`
- :heavy_check_mark: `git-repository-index`
- :heavy_check_mark: `git-repository-init`
- :interrobang: `git-repository-init-ext`
- :interrobang: `git-repository-init-init-options`
- :heavy_check_mark: `git-repository-is-bare`
- :heavy_check_mark: `git-repository-is-empty`
- :heavy_check_mark: `git-repository-is-shallow`
- :heavy_check_mark: `git-repository-is-worktree`
- :interrobang: `git-repository-item-path`
- :interrobang: `git-repository-mergehead-foreach`
- :heavy_check_mark: `git-repository-message`
- :heavy_check_mark: `git-repository-message-remove`
- :x: `git-repository-new` (in `sys`)
- :interrobang: `git-repository-odb`
- :heavy_check_mark: `git-repository-open`
- :heavy_check_mark: `git-repository-open-bare`
- :interrobang: `git-repository-open-ext`
- :interrobang: `git-repository-open-from-worktree`
- :heavy_check_mark: `git-repository-path`
- :interrobang: `git-repository-refdb`
- :x: `git-repository-reinit-filesystem` (in `sys`)
- :x: `git-repository-set-bare` (in `sys`)
- :x: `git-repository-set-config` (in `sys`)
- :heavy_check_mark: `git-repository-set-head`
- :heavy_check_mark: `git-repository-set-head-detached`
- :interrobang: `git-repository-set-head-detached-from-annotated`
- :heavy_check_mark: `git-repository-set-ident`
- :x: `git-repository-set-index` (in `sys`)
- :heavy_check_mark: `git-repository-set-namespace`
- :x: `git-repository-set-odb` (in `sys`)
- :x: `git-repository-set-refdb` (in `sys`)
- :heavy_check_mark: `git-repository-set-workdir`
- :heavy_check_mark: `git-repository-state`
- :heavy_check_mark: `git-repository-state-cleanup`
- :x: `git-repository-submodule-cache-all` (in `sys`)
- :x: `git-repository-submodule-cache-clear` (in `sys`)
- :heavy_check_mark: `git-repository-workdir`
- :interrobang: `git-repository-wrap-odb`

### reset

- :interrobang: `git-reset`
- :interrobang: `git-reset-default`
- :interrobang: `git-reset-from-annotated`

### revert

- :interrobang: `git-revert`
- :interrobang: `git-revert-commit`
- :interrobang: `git-revert-init-options`

### revparse

- :interrobang: `git-revparse`
- :interrobang: `git-revparse-ext`
- :heavy_check_mark: `git-revparse-single`

### revwalk

- :interrobang: `git-revwalk-add-hide-cb`
- :x: `git-revwalk-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-revwalk-hide`
- :interrobang: `git-revwalk-hide-glob`
- :interrobang: `git-revwalk-hide-head`
- :interrobang: `git-revwalk-hide-ref`
- :interrobang: `git-revwalk-new`
- :interrobang: `git-revwalk-next`
- :interrobang: `git-revwalk-push`
- :interrobang: `git-revwalk-push-glob`
- :interrobang: `git-revwalk-push-head`
- :interrobang: `git-revwalk-push-range`
- :interrobang: `git-revwalk-push-ref`
- :interrobang: `git-revwalk-repository`
- :interrobang: `git-revwalk-reset`
- :interrobang: `git-revwalk-simplify-first-parent`
- :interrobang: `git-revwalk-sorting`

### signature

- :heavy_check_mark: `git-signature-default`
- :interrobang: `git-signature-dup`
- :x: `git-signature-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-signature-from-buffer`
- :interrobang: `git-signature-new`
- :interrobang: `git-signature-now`

### smart

- :interrobang: `git-smart-subtransport-git`
- :interrobang: `git-smart-subtransport-http`
- :interrobang: `git-smart-subtransport-ssh`

### stash

- :interrobang: `git-stash-apply`
- :interrobang: `git-stash-apply-init-options`
- :interrobang: `git-stash-drop`
- :interrobang: `git-stash-foreach`
- :interrobang: `git-stash-pop`

### status

- :interrobang: `git-status-byindex`
- :heavy_check_mark: `git-status-file`
- :heavy_check_mark: `git-status-foreach`
- :heavy_check_mark: `git-status-foreach-ext`
- :x: `git-status-init-options`
- :interrobang: `git-status-list-entrycount`
- :interrobang: `git-status-list-free`
- :x: `git-status-list-get-perfdata` (in `sys`)
- :interrobang: `git-status-list-new`
- :heavy_check_mark: `git-status-should-ignore`

### strarray

- :x: `git-strarray-copy`
- :x: `git-strarray-free` (memory management shouldn't be exposed to Emacs)

### stream

- :x: `git-stream-register-tls` (in `sys`)

### submodule

- :interrobang: `git-submodule-add-finalize`
- :interrobang: `git-submodule-add-setup`
- :interrobang: `git-submodule-add-to-index`
- :interrobang: `git-submodule-branch`
- :interrobang: `git-submodule-fetch-recurse-submodules`
- :interrobang: `git-submodule-foreach`
- :x: `git-submodule-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-submodule-head-id`
- :interrobang: `git-submodule-ignore`
- :interrobang: `git-submodule-index-id`
- :interrobang: `git-submodule-init`
- :interrobang: `git-submodule-location`
- :interrobang: `git-submodule-lookup`
- :interrobang: `git-submodule-name`
- :interrobang: `git-submodule-open`
- :interrobang: `git-submodule-owner`
- :interrobang: `git-submodule-path`
- :interrobang: `git-submodule-reload`
- :interrobang: `git-submodule-repo-init`
- :interrobang: `git-submodule-resolve-url`
- :interrobang: `git-submodule-set-branch`
- :interrobang: `git-submodule-set-fetch-recurse-submodules`
- :interrobang: `git-submodule-set-ignore`
- :interrobang: `git-submodule-set-update`
- :interrobang: `git-submodule-set-url`
- :interrobang: `git-submodule-status`
- :interrobang: `git-submodule-sync`
- :interrobang: `git-submodule-update`
- :interrobang: `git-submodule-update-init-options`
- :interrobang: `git-submodule-update-strategy`
- :interrobang: `git-submodule-url`
- :interrobang: `git-submodule-wd-id`

### tag

- :interrobang: `git-tag-annotation-create`
- :interrobang: `git-tag-create`
- :interrobang: `git-tag-create-frombuffer`
- :interrobang: `git-tag-create-lightweight`
- :interrobang: `git-tag-delete`
- :interrobang: `git-tag-dup`
- :heavy_check_mark: `git-tag-foreach`
- :x: `git-tag-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-tag-id`
- :heavy_check_mark: `git-tag-list`
- :heavy_check_mark: `git-tag-list-match`
- :heavy_check_mark: `git-tag-lookup`
- :heavy_check_mark: `git-tag-lookup-prefix`
- :heavy_check_mark: `git-tag-message`
- :heavy_check_mark: `git-tag-name`
- :heavy_check_mark: `git-tag-owner`
- :heavy_check_mark: `git-tag-peel`
- :heavy_check_mark: `git-tag-tagger`
- :heavy_check_mark: `git-tag-target`
- :heavy_check_mark: `git-tag-target-id`
- :heavy_check_mark: `git-tag-target-type`

### time

- :x: `git-time-monotonic` (in `sys`)

### trace

- :interrobang: `git-trace-set`

### transaction

- :heavy_check_mark: `git_transaction_commit`
- :x: `git_transaction_free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git_transaction_lock_ref`
- :interrobang: `git_transaction_new`
- :interrobang: `git_transaction_remove`
- :interrobang: `git_transaction_set_reflog`
- :interrobang: `git_transaction_set_symbolic_target`
- :interrobang: `git_transaction_set_target`

### transport

- :x: `git-transport-dummy` (in `sys`)
- :x: `git-transport-init` (in `sys`)
- :x: `git-transport-local` (in `sys`)
- :x: `git-transport-new` (in `sys`)
- :x: `git-transport-register` (in `sys`)
- :x: `git-transport-smart` (in `sys`)
- :x: `git-transport-smart-certificate-check` (in `sys`)
- :x: `git-transport-smart-credentials` (in `sys`)
- :x: `git-transport-smart-proxy-options` (in `sys`)
- :x: `git-transport-ssh-with-paths` (in `sys`)
- :x: `git-transport-unregister` (in `sys`)

### tree

- :interrobang: `git-tree-create-updated`
- :interrobang: `git-tree-dup`
- :heavy_check_mark: `git-tree-entry-byid`
- :heavy_check_mark: `git-tree-entry-byindex`
- :heavy_check_mark: `git-tree-entry-byname`
- :heavy_check_mark: `git-tree-entry-bypath`
- :x: `git-tree-entry-cmp` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-dup` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-filemode` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-filemode-raw` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-free` (memory management shouldn't be exposed to Emacs)
- :x: `git-tree-entry-id` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-name` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-to-object` (tree entries are exposed to emacs as lists)
- :x: `git-tree-entry-type` (tree entries are exposed to emacs as lists)
- :heavy_check_mark: `git-tree-entrycount`
- :x: `git-tree-free` (memory management shouldn't be exposed to Emacs)
- :heavy_check_mark: `git-tree-id`
- :heavy_check_mark: `git-tree-lookup`
- :heavy_check_mark: `git-tree-lookup-prefix`
- :heavy_check_mark: `git-tree-owner`
- :heavy_check_mark: `git-tree-walk`

### treebuilder

- :interrobang: `git-treebuilder-clear`
- :interrobang: `git-treebuilder-entrycount`
- :interrobang: `git-treebuilder-filter`
- :x: `git-treebuilder-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-treebuilder-get`
- :interrobang: `git-treebuilder-insert`
- :interrobang: `git-treebuilder-new`
- :interrobang: `git-treebuilder-remove`
- :interrobang: `git-treebuilder-write`
- :interrobang: `git-treebuilder-write-with-buffer`

### worktree

- :interrobang: `git-worktree-add`
- :interrobang: `git-worktree-add-init-options`
- :x: `git-worktree-free` (memory management shouldn't be exposed to Emacs)
- :interrobang: `git-worktree-is-locked`
- :interrobang: `git-worktree-is-prunable`
- :interrobang: `git-worktree-list`
- :interrobang: `git-worktree-lock`
- :interrobang: `git-worktree-lookup`
- :interrobang: `git-worktree-open-from-repository`
- :interrobang: `git-worktree-prune`
- :interrobang: `git-worktree-prune-init-options`
- :interrobang: `git-worktree-unlock`
- :interrobang: `git-worktree-validate`
