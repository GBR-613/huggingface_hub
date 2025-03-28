<!--⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.
-->

# Download files from the Hub

The `huggingface_hub` library provides functions to download files from the repositories
stored on the Hub. You can use these functions independently or integrate them into your
own library, making it more convenient for your users to interact with the Hub. This
guide will show you how to:

* Download and cache a single file.
* Download and cache an entire repository.
* Download files to a local folder. 

## Download a single file

The [`hf_hub_download`] function is the main function for downloading files from the Hub.
It downloads the remote file, caches it on disk (in a version-aware way), and returns its local file path.

<Tip>

The returned filepath is a pointer to the HF local cache. Therefore, it is important to not modify the file to avoid
having a corrupted cache. If you are interested in getting to know more about how files are cached, please refer to our
[caching guide](./manage-cache).

</Tip>

### From latest version

Select the file to download using the `repo_id`, `repo_type` and `filename` parameters. By default, the file will
be considered as being part of a `model` repo.

```python
>>> from huggingface_hub import hf_hub_download
>>> hf_hub_download(repo_id="lysandre/arxiv-nlp", filename="config.json")
'/root/.cache/huggingface/hub/models--lysandre--arxiv-nlp/snapshots/894a9adde21d9a3e3843e6d5aeaaf01875c7fade/config.json'

# Download from a dataset
>>> hf_hub_download(repo_id="google/fleurs", filename="fleurs.py", repo_type="dataset")
'/root/.cache/huggingface/hub/datasets--google--fleurs/snapshots/199e4ae37915137c555b1765c01477c216287d34/fleurs.py'
```

### From specific version

By default, the latest version from the `main` branch is downloaded. However, in some cases you want to download a file
at a particular version (e.g. from a specific branch, a PR, a tag or a commit hash).
To do so, use the `revision` parameter:

```python
# Download from the `v1.0` tag
>>> hf_hub_download(repo_id="lysandre/arxiv-nlp", filename="config.json", revision="v1.0")

# Download from the `test-branch` branch
>>> hf_hub_download(repo_id="lysandre/arxiv-nlp", filename="config.json", revision="test-branch")

# Download from Pull Request #3
>>> hf_hub_download(repo_id="lysandre/arxiv-nlp", filename="config.json", revision="refs/pr/3")

# Download from a specific commit hash
>>> hf_hub_download(repo_id="lysandre/arxiv-nlp", filename="config.json", revision="877b84a8f93f2d619faa2a6e514a32beef88ab0a")
```

**Note:** When using the commit hash, it must be the full-length hash instead of a 7-character commit hash.

### Construct a download URL

In case you want to construct the URL used to download a file from a repo, you can use [`hf_hub_url`] which returns a URL.
Note that it is used internally by [`hf_hub_download`].

## Download an entire repository

[`snapshot_download`] downloads an entire repository at a given revision. It uses internally [`hf_hub_download`] which
means all downloaded files are also cached on your local disk. Downloads are made concurrently to speed-up the process.

To download a whole repository, just pass the `repo_id` and `repo_type`:

```python
>>> from huggingface_hub import snapshot_download
>>> snapshot_download(repo_id="lysandre/arxiv-nlp")
'/home/lysandre/.cache/huggingface/hub/models--lysandre--arxiv-nlp/snapshots/894a9adde21d9a3e3843e6d5aeaaf01875c7fade'

# Or from a dataset
>>> snapshot_download(repo_id="google/fleurs", repo_type="dataset")
'/home/lysandre/.cache/huggingface/hub/datasets--google--fleurs/snapshots/199e4ae37915137c555b1765c01477c216287d34'
```

[`snapshot_download`] downloads the latest revision by default. If you want a specific repository revision, use the
`revision` parameter:

```python
>>> from huggingface_hub import snapshot_download
>>> snapshot_download(repo_id="lysandre/arxiv-nlp", revision="refs/pr/1")
```

### Filter files to download

[`snapshot_download`] provides an easy way to download a repository. However, you don't always want to download the
entire content of a repository. For example, you might want to prevent downloading all `.bin` files if you know you'll
only use the `.safetensors` weights. You can do that using `allow_patterns` and `ignore_patterns` parameters.

These parameters accept either a single pattern or a list of patterns. Patterns are Standard Wildcards (globbing
patterns) as documented [here](https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/x11655.htm). The pattern matching is
based on [`fnmatch`](https://docs.python.org/3/library/fnmatch.html).

For example, you can use `allow_patterns` to only download JSON configuration files:

```python
>>> from huggingface_hub import snapshot_download
>>> snapshot_download(repo_id="lysandre/arxiv-nlp", allow_patterns="*.json")
```

On the other hand, `ignore_patterns` can exclude certain files from being downloaded. The
following example ignores the `.msgpack` and `.h5` file extensions:

```python
>>> from huggingface_hub import snapshot_download
>>> snapshot_download(repo_id="lysandre/arxiv-nlp", ignore_patterns=["*.msgpack", "*.h5"])
```

Finally, you can combine both to precisely filter your download. Here is an example to download all json and markdown
files except `vocab.json`.

```python
>>> from huggingface_hub import snapshot_download
>>> snapshot_download(repo_id="gpt2", allow_patterns=["*.md", "*.json"], ignore_patterns="vocab.json")
```

## Download file(s) to local folder

The recommended (and default) way to download files from the Hub is to use the [cache-system](./manage-cache).
You can define your cache location by setting `cache_dir` parameter (both in [`hf_hub_download`] and [`snapshot_download`]).

However, in some cases you want to download files and move them to a specific folder. This is useful to get a workflow
closer to what `git` commands offer. You can do that using the `local_dir` and `local_dir_use_symlinks` parameters:
- `local_dir` must be a path to a folder on your system. The downloaded files will keep the same file structure as in the
repo. For example if `filename="data/train.csv"` and `local_dir="path/to/folder"`, then the returned filepath will be
`"path/to/folder/data/train.csv"`.
- `local_dir_use_symlinks` defines how the file must be saved in your local folder.
  - The default behavior (`"auto"`) is to duplicate small files (<5MB) and use symlinks for bigger files. Symlinks allow
    to optimize both bandwidth and disk usage. However manually editing a symlinked file might corrupt the cache, hence
    the duplication for small files. The 5MB threshold can be configured with the `HF_HUB_LOCAL_DIR_AUTO_SYMLINK_THRESHOLD`
    environment variable.
  - If `local_dir_use_symlinks=True` is set, all files are symlinked for an optimal disk space optimization. This is
    for example useful when downloading a huge dataset with thousands of small files.
  - Finally, if you don't want symlinks at all you can disable them (`local_dir_use_symlinks=False`). The cache directory
    will still be used to check wether the file is already cached or not. If already cached, the file is **duplicated**
    from the cache (i.e. saves bandwidth but increases disk usage). If the file is not already cached, it will be
    downloaded and moved directly to the local dir. This means that if you need to reuse it somewhere else later, it
    will be **re-downloaded**.

Here is a table that summarizes the different options to help you choose the parameters that best suit your use case.

<!-- Generated with https://www.tablesgenerator.com/markdown_tables -->
| Parameters | File already cached | Returned path | Can read path? | Can save to path? | Optimized bandwidth | Optimized disk usage |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `local_dir=None` |  | symlink in cache | ✅ | ❌<br>_(save would corrupt the cache)_ | ✅ | ✅ |
| `local_dir="path/to/folder"`<br>`local_dir_use_symlinks="auto"` |  | file or symlink in folder | ✅ | ✅ _(for small files)_ <br> ⚠️ _(for big files do not resolve path before saving)_ | ✅ | ✅ |
| `local_dir="path/to/folder"`<br>`local_dir_use_symlinks=True` |  | symlink in folder | ✅ | ⚠️<br>_(do not resolve path before saving)_ | ✅ | ✅ |
| `local_dir="path/to/folder"`<br>`local_dir_use_symlinks=False` | No | file in folder | ✅ | ✅ | ❌<br>_(if re-run, file is re-downloaded)_ | ⚠️<br>(multiple copies if ran in multiple folders) |
| `local_dir="path/to/folder"`<br>`local_dir_use_symlinks=False` | Yes | file in folder | ✅ | ✅ | ⚠️<br>_(file has to be cached first)_ | ❌<br>_(file is duplicated)_ |

**Note:** if you are on a Windows machine, you need to enable developer mode or run `huggingface_hub` as admin to enable
symlinks. Check out the [cache limitations](../guides/manage-cache#limitations) section for more details.