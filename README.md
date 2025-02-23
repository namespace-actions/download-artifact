# `namespace-actions/download-artifact`

Namespace's version of Download [Actions Artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts) compatible with the [upload-artifact](https://github.com/actions/upload-artifact).

The artifacts are stored in Namespace internal storage. They will not be visible and billed in GitHub.

Upload artifacts with [upload-artifact](https://github.com/namespace-actions/upload-artifact).

## Usage

### Inputs

```yaml
- uses: namespace-actions/download-artifact@v0
  with:
    # Name of the artifact to download.
    # If unspecified, all artifacts for the run are downloaded.
    # Optional.
    name:

    # Destination path. Supports basic tilde expansion.
    # Optional. Default is $GITHUB_WORKSPACE
    path:

    # A glob pattern to the artifacts that should be downloaded.
    # Ignored if name is specified.
    # Optional.
    pattern:

    # When multiple artifacts are matched, this changes the behavior of the destination directories.
    # If true, the downloaded artifacts will be in the same directory specified by path.
    # If false, the downloaded artifacts will be extracted into individual named directories within the specified path.
    # Optional. Default is 'false'
    merge-multiple:

    # The GitHub token used to authenticate with the GitHub API.
    # This is required when downloading artifacts from a different repository or from a different workflow run.
    # Optional. If unspecified, the action will download artifacts from the current repo and the current workflow run.
    github-token:

    # The repository owner and the repository name joined together by "/".
    # If github-token is specified, this is the repository that artifacts will be downloaded from.
    # Optional. Default is ${{ github.repository }}
    repository:

    # The id of the workflow run where the desired download artifact was uploaded from.
    # If github-token is specified, this is the run that artifacts will be downloaded from.
    # Optional. Default is ${{ github.run_id }}
    run-id:
```

### Outputs

| Name | Description | Example |
| - | - | - |
| `download-path` | Absolute path where the artifact(s) were downloaded | `/tmp/my/download/path` |

## Examples

### Download Single Artifact

Download to current working directory (`$GITHUB_WORKSPACE`):

```yaml
steps:
- uses: namespace-actions/download-artifact@v0
  with:
    name: my-artifact
- name: Display structure of downloaded files
  run: ls -R
```

Download to a specific directory (also supports `~` expansion):

```yaml
steps:
- uses: namespace-actions/download-artifact@v0
  with:
    name: my-artifact
    path: your/destination/dir
- name: Display structure of downloaded files
  run: ls -R your/destination/dir
```

### Download All Artifacts

If the `name` input parameter is not provided, all artifacts will be downloaded. To differentiate between downloaded artifacts, by default a directory denoted by the artifacts name will be created for each individual artifact. This behavior can be changed with the `merge-multiple` input parameter.

Example, if there are two artifacts `Artifact-A` and `Artifact-B`, and the directory is `etc/usr/artifacts/`, the directory structure will look like this:

```
etc/usr/artifacts/
    Artifact-A/
        ... contents of Artifact-A
    Artifact-B/
        ... contents of Artifact-B
```

Download all artifacts to the current working directory:

```yaml
steps:
- uses: namespace-actions/download-artifact@v0
- name: Display structure of downloaded files
  run: ls -R
```

Download all artifacts to a specific directory:

```yaml
steps:
- uses: namespace-actions/download-artifact@v0
  with:
    path: path/to/artifacts
- name: Display structure of downloaded files
  run: ls -R path/to/artifacts
```

To download them to the _same_ directory:

```yaml
steps:
- uses: namespace-actions/download-artifact@v0
  with:
    path: path/to/artifacts
    merge-multiple: true
- name: Display structure of downloaded files
  run: ls -R path/to/artifacts
```

Which will result in:

```
path/to/artifacts/
    ... contents of Artifact-A
    ... contents of Artifact-B
```

### Download multiple (filtered) Artifacts to the same directory

In multiple arch/os scenarios, you may have Artifacts built in different jobs. To download all Artifacts to the same directory (or matching a glob pattern), you can use the `pattern` and `merge-multiple` inputs.

```yaml
jobs:
  upload:
    strategy:
      matrix:
        runs-on: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.runs-on }}
    steps:
    - name: Create a File
      run: echo "hello from ${{ matrix.runs-on }}" > file-${{ matrix.runs-on }}.txt
    - name: Upload Artifact
      uses: actions/upload-artifact@v4
      with:
        name: my-artifact-${{ matrix.runs-on }}
        path: file-${{ matrix.runs-on }}.txt
  download:
    needs: upload
    runs-on: ubuntu-latest
    steps:
    - name: Download All Artifacts
      uses: namespace-actions/download-artifact@v0
      with:
        path: my-artifact
        pattern: my-artifact-*
        merge-multiple: true
    - run: ls -R my-artifact
```

This results in a directory like so:

```
my-artifact/
  file-macos-latest.txt
  file-ubuntu-latest.txt
  file-windows-latest.txt
```

### Download Artifacts from other Workflow Runs or Repositories

It may be useful to download Artifacts from other workflow runs, or even other repositories. By default, the permissions are scoped so they can only download Artifacts within the current workflow run. To elevate permissions for this scenario, you can specify a `github-token` along with other repository and run identifiers:

```yaml
steps:
- uses: namespace-actions/download-artifact@v0
  with:
    name: my-other-artifact
    github-token: ${{ secrets.GH_PAT }} # token with actions:read permissions on target repo
    repository: actions/toolkit
    run-id: 1234
```

## Limitations

### Permission Loss

File permissions are not maintained during artifact upload. All directories will have `755` and all files will have `644`. For example, if you make a file executable using `chmod` and then upload that file, post-download the file is no longer guaranteed to be set as an executable.

If you must preserve permissions, you can `tar` all of your files together before artifact upload. Post download, the `tar` file will maintain file permissions and case sensitivity.

```yaml
- name: 'Tar files'
  run: tar -cvf my_files.tar /path/to/my/directory

- name: 'Upload Artifact'
  uses: actions/upload-artifact@v4
  with:
    name: my-artifact
    path: my_files.tar
```
