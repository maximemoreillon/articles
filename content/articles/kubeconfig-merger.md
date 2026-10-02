---
date: "2025-11-12T00:00:00+09:00"
title: "Kubeconfig-merger: merging kubeconfig files into ~/.kube/config"
tags: ["Kubernetes", "Go"]
---

Each new Kubernetes cluster usually comes with its own kubeconfig file, and getting it into `~/.kube/config` means copying its cluster, user and context entries over by hand. Kubeconfig-merger is a small Go CLI that removes this step: kubeconfig files are simply dropped into a directory, and the application merges all of them into `~/.kube/config`.

## How it works

By default, kubeconfig files are kept as separate files in `~/.kube/config.d`, one per cluster. The application:

1. Walks the source directory recursively and collects every `.yaml` and `.yml` file
2. Sets `KUBECONFIG` to the list of those files, using the path separator of the current OS (`;` on Windows, `:` elsewhere)
3. Runs `kubectl config view --merge --flatten`, which merges the files and embeds certificates and keys inline instead of referencing them by path
4. Writes the output to `~/.kube/config`

The merging itself is delegated to `kubectl`, so `kubectl` must be installed and available in the `PATH`.

## Usage

```bash
./kubeconfig-merger
```

A source directory other than `~/.kube/config.d` can be specified with the `-source` flag:

```bash
./kubeconfig-merger -source /path/to/config/dir
```

Adding or removing a cluster then comes down to adding or removing a file in the source directory and running the merger again. Note that `~/.kube/config` is overwritten each time, so it should only be edited through the files in the source directory.

## Source code

The source code is available on [GitHub](https://github.com/maximemoreillon/kubeconfig-merger). Binaries are built with GoReleaser through GitHub Actions.
