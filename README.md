# setup-ruby

This action downloads a prebuilt ruby and adds it to the `PATH`.

It is very efficient and takes about 5 seconds to download, extract and add the given Ruby to the `PATH`.
No extra packages need to be installed.

Compared to [actions/setup-ruby](https://github.com/actions/setup-ruby),
this actions supports many more versions and features.

### Supported Versions

This action currently supports these versions of MRI, JRuby and TruffleRuby:

| Interpreter | Versions |
| ----------- | -------- |
| Ruby | 2.2, 2.3.0 - 2.3.8, 2.4.0 - 2.4.10, 2.5.0 - 2.5.8, 2.6.0 - 2.6.6, 2.7.1, head, debug, mingw, mswin |
| JRuby | 9.1.17.0, 9.2.9.0 - 9.2.11.1, head |
| TruffleRuby | 19.3.0 - 20.0.0, head |
| Rubinius | 4.14 |

`ruby-debug` is the same as `ruby-head` but with assertions enabled (`-DRUBY_DEBUG=1`).  
On Windows, `mingw` and `mswin` are `ruby-head` builds using the MSYS2/MinGW and the MSVC toolchains respectively.

Ruby 2.2 resolves to 2.2.6 on Windows (last build from RubyInstaller) and 2.2.10 otherwise.  
Ruby 2.3 on Windows only has builds for 2.3.0, 2.3.1 and 2.3.3 (same as RubyInstaller).

Note that Ruby ≤ 2.3 and the OpenSSL version it needs (1.0.2) are both end-of-life,
which means Ruby ≤ 2.3 is unmaintained and considered insecure.
On Windows, Ruby 2.4 uses OpenSSL 1.0.2, which is no longer maintained.

### Supported Platforms

The action works for all [GitHub-hosted runners](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/virtual-environments-for-github-hosted-runners).

| Operating System | Versions |
| ----------- | -------- |
| Ubuntu  | `ubuntu-latest` (= `ubuntu-18.04`), `ubuntu-16.04` |
| macOS   | `macos-latest` |
| Windows | `windows-latest` |

Rubinius is only available on `ubuntu-latest`.

The prebuilt releases are generated by [ruby-builder](https://github.com/ruby/ruby-builder)
and on Windows by [RubyInstaller2](https://github.com/oneclick/rubyinstaller2).
`mingw` and `mswin` builds are generated by [ruby-loco](https://github.com/MSP-Greg/ruby-loco).
`ruby-head` is generated by [ruby-dev-builder](https://github.com/ruby/ruby-dev-builder),
`jruby-head` is generated by [jruby-dev-builder](https://github.com/ruby/jruby-dev-builder)
and `truffleruby-head` is generated by [truffleruby-dev-builder](https://github.com/ruby/truffleruby-dev-builder).
The full list of available Ruby versions can be seen in [ruby-builder-versions.js](ruby-builder-versions.js)
for Ubuntu and macOS and in [windows-versions.js](windows-versions.js) for Windows.

## Usage

### Single Job

```yaml
name: My workflow
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: ruby/setup-ruby@v1
      with:
        ruby-version: 2.6 # Not needed with a .ruby-version file
    - run: bundle install
    - run: bundle exec rake
```

### Matrix

This matrix tests all stable releases and `head` versions of MRI, JRuby and TruffleRuby on Ubuntu and macOS.

```yaml
name: My workflow
on: [push]
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu, macos]
        ruby: [2.5, 2.6, 2.7, head, debug, jruby, jruby-head, truffleruby, truffleruby-head]
    runs-on: ${{ matrix.os }}-latest
    continue-on-error: ${{ endsWith(matrix.ruby, 'head') || matrix.ruby == 'debug' }}
    steps:
    - uses: actions/checkout@v2
    - uses: ruby/setup-ruby@v1
      with:
        ruby-version: ${{ matrix.ruby }}
    - run: bundle install
    - run: bundle exec rake
```

See the GitHub Actions documentation for more details about the
[workflow syntax](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions)
and the [condition and expression syntax](https://help.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions).

### Supported Version Syntax

* engine-version like `ruby-2.6.5` and `truffleruby-19.3.0`
* short version like `2.6`, automatically using the latest release matching that version (`2.6.5`)
* version only like `2.6.5`, assumes MRI for the engine
* engine only like `truffleruby`, uses the latest stable release of that implementation
* `.ruby-version` reads from the project's `.ruby-version` file
* `.tool-versions` reads from the project's `.tool-versions` file
* If the `ruby-version` input is not specified, `.ruby-version` is tried first, followed by `.tool-versions`

### Bundler

By default, if there is a `Gemfile.lock` file with a `BUNDLED WITH` section,
the latest version of Bundler with the same major version will be installed.
Otherwise, the latest Bundler version is installed (except for Ruby 2.2 and 2.3 where only Bundler 1 is supported).

This behavior can be customized, see [action.yml](action.yml) for details about the `bundler` input.

### Caching `bundle install`

You can cache the installed gems with these two steps:

```yaml
    - uses: actions/cache@v1
      with:
        path: vendor/bundle
        key: bundle-use-ruby-${{ matrix.os }}-${{ matrix.ruby }}-${{ hashFiles('**/Gemfile.lock') }}
        restore-keys: |
          bundle-use-ruby-${{ matrix.os }}-${{ matrix.ruby }}-
    - name: bundle install
      run: |
        bundle config deployment true
        bundle config path vendor/bundle
        bundle install --jobs 4
```

When using a single OS, replace `${{ matrix.os }}` with the OS.  
When using a single job with a Ruby version, replace `${{ matrix.ruby }}` with the Ruby version.  
When using `.ruby-version`, replace `${{ matrix.ruby }}` with `${{ hashFiles('.ruby-version') }}`.  
When using `.tool-versions`, replace `${{ matrix.ruby }}` with `${{ hashFiles('.tool-versions') }}`.

This uses the [cache action](https://github.com/actions/cache).
The code above is a more complete version of the [Ruby - Bundler example](https://github.com/actions/cache/blob/master/examples.md#ruby---bundler).
Make sure to include `use-ruby` in the `key` to avoid conflicting with previous caches.

### Working Directory

The `working-directory` input can be set to resolve `.ruby-version`, `.tool-versions` and `Gemfile.lock`
if they are not at the root of the repository, see [action.yml](action.yml) for details.

## Windows

Note that running CI on Windows can be quite challenging if you are not very familiar with Windows.
It is recommended to first get your build working on Ubuntu and macOS before trying Windows.

* The default shell on Windows is not Bash but [PowerShell](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#using-a-specific-shell).
  This can lead issues such as multi-line scripts [not working as expected](https://github.com/ruby/setup-ruby/issues/13).
* The `PATH` contains [multiple compiler toolchains](https://github.com/ruby/setup-ruby/issues/19). Use `where.exe` to debug which tool is used.
* For Ruby ≥ 2.4, MSYS2 is prepended to the `Path`, similar to what RubyInstaller2 does.
* For Ruby < 2.4, the DevKit MSYS tools are installed and prepended to the `Path`.
* JRuby on Windows has a known bug that `bundle exec rake` [fails](https://github.com/ruby/setup-ruby/issues/18).

## Versioning

It is highly recommended to use `ruby/setup-ruby@v1` for the version of this action.
This will provide the best experience by automatically getting bug fixes, new Ruby versions and new features.

If you instead choose a specific version (v1.2.3) or a commit sha, there will be no automatic bug fixes and
it will be your responsibility to update every time the action no longer works.
Make sure to always use the latest release before reporting an issue on GitHub.

This action follows semantic versioning with a moving `v1` branch.
This follows the [recommendations](https://github.com/actions/toolkit/blob/master/docs/action-versioning.md) of GitHub Actions.

## Limitations

* This action currently only works with GitHub-hosted runners, not private runners.

## History

This action used to be at `eregon/use-ruby-action` and was moved to the `ruby` organization.
Please [update](https://github.com/ruby/setup-ruby/releases/tag/v1.13.0) if you are using `eregon/use-ruby-action`.

## Credits

The current maintainer of this action is @eregon.
Most of the Windows logic is based on work by MSP-Greg.
Many thanks to MSP-Greg and Lars Kanis for the help with Ruby Installer.
