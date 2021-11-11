# setup-texlive-action

[![Ubuntu CI tests status badge][ubuntu-ci-shield]][ubuntu-ci-url]
[![macOS CI tests status badge][macos-ci-shield]][macos-ci-url]
[![Windows CI tests status badge][windows-ci-shield]][windows-ci-url]

[![Latest release badge][release-shield]][release-url]
[![License badge][license-shield]][license-url]

[ubuntu-ci-url]: https://github.com/paolobrasolin/setup-texlive-action/actions/workflows/ubuntu.yml "Ubuntu CI tests"
[macos-ci-url]: https://github.com/paolobrasolin/setup-texlive-action/actions/workflows/macos.yml "macOS CI tests"
[windows-ci-url]: https://github.com/paolobrasolin/setup-texlive-action/actions/workflows/windows.yml "Windows CI tests"

[ubuntu-ci-shield]: https://img.shields.io/github/workflow/status/paolobrasolin/setup-texlive-action/ubuntu/main?label=Ubuntu&logo=ubuntu&logoColor=white
[macos-ci-shield]: https://img.shields.io/github/workflow/status/paolobrasolin/setup-texlive-action/macos/main?label=macOS&logo=apple&logoColor=white
[windows-ci-shield]: https://img.shields.io/github/workflow/status/paolobrasolin/setup-texlive-action/windows/main?label=Windows&logo=windows&logoColor=white

[release-url]: https://github.com/marketplace/actions/setup-texlive-action "Latest release"
[license-url]: https://github.com/paolobrasolin/setup-texlive-action/blob/main/LICENSE "License"

[release-shield]: https://img.shields.io/github/v/release/paolobrasolin/setup-texlive-action?display_name=tag&sort=semver
[license-shield]: https://img.shields.io/github/license/paolobrasolin/setup-texlive-action

This action provides a smooth way to setup a TeX Live distribution tailored to your needs. Its features:

* preconfigured cache for blazing-fast builds,
* same customization level of a local installation,
* compatible with Ubuntu, macOS and Windows runners.

# Getting started

Here is the minimal workflow to see the action running:

```yaml
# .github/workflows/barebones.yml
on: push
jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - uses: paolobrasolin/setup-texlive-action@v1
      - run: tlmgr version
```

This will install the `infraonly` scheme and check that `tlmgr` is available.

That's nice, but too bare-bones to actually compile a document.

To pick a richer scheme and a few custom packages you will need to setup two configuration files and feed them to the action:

```ini
# .github/texlive.profile
# Install the scheme minimal:
selected_scheme scheme-minimal
# Omit documentation files:
tlpdbopt_install_docfiles 0
# Omit source files:
tlpdbopt_install_srcfiles 0
# Avoid doing backups:
tlpdbopt_autobackup 0
```

```ini
# .github/texlive.packages
latex-bin
```

```yaml
# .github/workflows/barebones.yml
on: push
jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - uses: paolobrasolin/setup-texlive-action@v1
        with:
          profile-path: ${{ github.workspace }}/.github/texlive.profile
          packages-path: ${{ github.workspace }}/.github/texlive.packages
      - run: pdflatex sample2e
```

You just compiled `sample2e` in your CI!
The world is your oyster.

# Usage

Here is a full rundown of options for advanced usage.

## Inputs

### `profile-path`

* Optional
* Default: the path of the [default profile][default-profile], which provides just the `infraonly` scheme

This option is the file path of the profile that the action will feed to [`install-tl`][install-tl-man] to setup your distribution.

The [default profile][default-profile] is an excellent starting point to write your own.
It contains general instructions and most likely you'll just need to copy it and pick a different scheme.

**PLEASE be a responsible citizen and avoid waste!**

* Use the smallest setup which is suitable for your purpose.
* Omit documentation, source files and backups (the [default profile][default-profile] already contains the correct flags for that).

If more details are needed, you can find everything about writing profiles in the [official documentation][install-tl-profiles-man].

[default-profile]: https://github.com/paolobrasolin/setup-texlive-action/blob/main/texlive.profile
[install-tl-man]: https://www.tug.org/texlive/doc/install-tl.html
[install-tl-profiles-man]: https://www.tug.org/texlive/doc/install-tl.html#PROFILES

### `packages-path`

* Optional
* Default: the path of the [default list][default-packages], which is empty

This option is the file path of the package list that the action will feed to [`tlmgr`][tlmgr-man] to install them.

The empty [default list][default-packages] might be enough if the scheme you picked already covers all your bases.
If that's not the case, the file format is simple: list the packages one per line. Lines starting with `#` are comments and will be ignored.

To search or browse for packages you can use [CTAN][ctan].

[default-packages]: https://github.com/paolobrasolin/setup-texlive-action/blob/main/texlive.packages
[tlmgr-man]: https://www.tug.org/texlive/doc/tlmgr.html
[ctan]: https://www.ctan.org/

### `cache-key`

* Optional
* Default: `texlive`

This option is the key which will identify the cached installation.

**Irrespectively of how you set this, the cache will expire when your profile or your package list change.**

You won't need to change this unless your CI requires finely configured cache reuse and segregation.

For the sake of concreteness, let's make an example.
Imagine you need to run your daily tests on three different platforms, while keeping their caches separate and forcing their expiration every week.
Here is how to do that:

```yaml
# .github/workflows/example.yml
name: example
on:
  schedule:
    - cron: '0 0 * * *' # daily at midnight
jobs:
  main:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ 'ubuntu-latest', 'macos-latest', 'windows-latest' ]
    steps:
      - name: Generate a YYYY-WW timestamp
        id: timestamp
        run: ${{ runner.os == 'Windows' && 'Write-Output "::set-output name=yyyy-ww::$(Get-Date -UFormat %Y-%W)"' || 'echo "::set-output name=yyyy-ww::$(date +%Y-%W)"' }}
      - uses: paolobrasolin/setup-texlive-action@v1
        with:
          cache-key: texlive-${{ matrix.os }}-${{ steps.timestamp.output.yyyy-ww }}
      - run: tlmgr version
```

The caching mechanism is implemented using the excellent [`cache`][cache-action] action. It has some [limitations][cache-action-limits]:

* up to 5GB per repo,
* older caches are evicted upon reaching the limit,
* caches are evicted after a week if unused.

[cache-action]: https://github.com/actions/cache
[cache-action-limits]: https://github.com/actions/cache#cache-limits

### `cache-enabled`

* Optional
* Default: `true`

This option can disable caching.

**PLEASE be a responsible citizen and always use caching!**

Not only this will avoid uselessly burdening CTAN mirrors, but your build will also be **much faster**.

### `installation-path`

* Optional
* Default: the `texlive` folder inside the runner's tool cache path

This option is the path where the TeX Live is installed on the runner.

There should be no need to change it.

## Outputs

The action currently provides no outputs.
