# MarkRight VS Code

This repository hosts the source code for the MarkRight VS Code extension. It is
written using MarkRight itself, so it also serves as a test bench for MarkRight.

If you do not know what MarkRight is, check out its main repository at
https://github.com/tomashubelbauer/markright.

To run MarkRight on this document, run `npx tomashubelbauer/markright watch`.
This will fetch the latest development version of MarkRight and start it in
watch mode, so any changes to `readme.md` will be reflected by MarkRight.

You can also just install MarkRight globally using NPM by running this command:
`npm install --global tomashubelbauer/markright` and use it using `markright` or
`markright watch` in this directory.

While prototyping, I'll start off by ensuring the directory is completely empty
except for the MarkRight entry document (this `readme.md`). Later on once the
code is more complete, we'll have more code blocks which reuse stuff if it exist
and optimize the runtime that way. But for now it is more beneficial to know we
always start off from the same spot.

```sh
# Preserve node_modules and .vscode-test because they are ignored so that their
# content doesn't matter and keeping them around preserves installations of some
# dependencies so it also speeds up the duration of MarkRight processing

# Do not use `Remove-Item -Exclude` as it does not work properly after PS 3.0+
Get-ChildItem -Exclude readme.md,node_modules,.vscode-test | Remove-Item -Recurse
```

I'll be following the official *Your First Extension* guide on the VS Code site.
https://code.visualstudio.com/api/get-started/your-first-extension

As we can see, the first step is to install Yeoman and its generator which is
used to scaffold VS Code extension repository, `generator-code`. Yeoman requires
that the `generator-code` package is installed locally, not globally, otherwise
it won't see it (test using `yo --generators`). Let's create a `.gitignore` for
`node_modules` then:

```shell .gitignore
# Node
node_modules
```

Now we're ready to check out Yeoman installation and install if it's missing.
Same with the `code-generator` package:

```sh
# Install Yeoman if it is not installed already
if ($null -eq (Get-Command "yo" -ErrorAction SilentlyContinue)) {
  # Supress output - this also supresses errors, but we'll find out later anyway
  [void] (npm install --global yo --silent)
}

# Print Yeoman version number for comparison with the expected output
yo --version

# Install generator-code as a local dependency so that Yeoman sees it
$ls = $(npm ls 2> $null) | where { $_ -match "`-- generator-code" }
if ($ls -eq $null) {
  # Supress output - this also supresses errors, but we'll find out later anyway
  [void] (npm install generator-code --silent)
  $ls = $(npm ls 2> $null) | where { $_ -match "`-- generator-code" }
}

# Print the `generator-code` package version for comparison with expected output
# Split by space in case we already have the Git PR version which has extra info
(($ls -split "@")[1] -split " ")[0]

# Ensure `generator-code` is seem by Yeoman in its `yo --generators` output
if ($(yo --generators) -notcontains '  code') {
  throw "Yeoman does not see generator-code"
}
```

The Yeoman version we expect to work with here is `3.1.1`. We can ensure this is
honored (otherwise this document would error and force us to update it) by
checking the standard output of the previous script; it prints the version
numbers of both Yeoman and the `generator-code` package.

```stdout
3.1.1
1.3.5
```

MarkRight will throw here if the versions documented don't match the reality so
we can document the rest of the process knowing we are indeed working with the
right versions.

Next up we're ready to invoke the Yeoman generator and have it scaffold the VS
code extension code for us.

We'll make use of the Yeoman's ability to provide answers to the generator
prompts through command line arguments.

The Code generator asks for a few questions:

- What type of extension do you want to create? - `extensionType`
- What's the name of your extension? - `extensionDisplayName`
- What's the identifier of your extension? - `extensionName`
- What's the description of your extension? - `extensionDescription`

Finding the names of the command line arguments corresponding to these questions
is possible by using `yo code --help`. However, not all of the questions were
covered there. By inspecting the source code of the generator:

https://github.com/microsoft/vscode-generator-code/blob/master/generators/app/generate-command-ts%20.js

I found that it uses the methods for asking the questions from this file:

https://github.com/microsoft/vscode-generator-code/blob/master/generators/app/prompts.js

The questions not being documented in the `--help` command are:

- Initialize a Git repository? - `initGit`
- Which package manager to use? - `pkgManager`
- Bundle the source code with webpack? - `webpack`

Not only are the command line options for these questions not documented, they
are not even supported. The support for providing them through the CLI is
missing. I've submitted a PR to `microsoft/vscode-generator-code` to fix this:

https://github.com/microsoft/vscode-generator-code/pull/227

Until that PR is merged, we cannot scaffold the extension using the official
generator, but we can replace it with out fork's `patch-1` branch which was
created when I created the PR using the GitHub UI and has the updated code:

```sh
# Supress output - this also supresses errors, but we'll find out later anyway
[void] (npm install tomashubelbauer/vscode-generator-code#patch-1 --silent)
$(npm ls 2> $null) | where { $_ -match "`-- generator-code" }
```

This will replace the `microsoft/vscode-genrator-code` NPM package with
`tomashubelbauer/vscode-generator-code` Git repository package as it is in the
`patch-1` branch.

Let's verify that:

```stdout
`-- generator-code@1.3.5 (github:tomashubelbauer/vscode-generator-code#a962e050aa99e9af8e1b776b83df61966777f316)
```

One last detail to take care of is to prevent Yeoman from installing NPM
dependencies for us after scaffolding the extension code base. We'll do that
outselves to make that step explicit. To do this, we lean on `yo code --help`
again and use the `--skip-instal` command line argument.

```sh
yo code `
  --extensionType command-ts `
  --extensionDisplayName MarkRight `
  --extensionName vscode-markright `
  --extensionDescription "MarkRight support for VS Code" `
  --gitInit false `
  --pkgManager npm `
  --webpack false `
  --skip-install `

```

Unfortunately it looks like even though the generator now accepts the command
line arguments for the remaining questions, it does not honor the values as
`.git` is still being created in the `vscode-markright` directory and it looks
like the WebPack build is being set up, too.

We won't worry about that too much for now, we'll just wipe the `.git` from
`vscode-markright` and live with the fact we've using WebPack for bundle. At
least NPM is the default package manager so one of the three choices is the way
we wanted it.

```sh
Remove-Item -LiteralPath "vscode-markright/.git" -Force -Recurse
ls vscode-markright -n
```

The expected listing of `vscode-markright` now is:

```stdout
.vscode
build
src
.eslintrc.json
.gitignore
.vscodeignore
CHANGELOG.md
package.json
README.md
tsconfig.json
vsc-extension-quickstart.md
```

Let's also delete `node_modules` we have `generator-code` installed in and the
`package-lock.json` file created as a result of the `npm install` commands that
have been issued (if any have).

```sh
Remove-Item -LiteralPath "node_modules" -Force -Recurse
if (Test-Path "package-lock.json") {
  Remove-Item "package-lock.json"
}

# Exclude the directories we keep around if they exists for caching purposes
# (They do not exist on the first run but do on subsequent runs)
Get-ChildItem -Name -Exclude node_modules,.vscode-test
```

Now the reposotory directory is now and clean:

```stdout
vscode-markright
.gitignore
readme.md
```

We've cleared out the root directory of our repository because that's where we
are going to hoist the generated files from `vscode-markright` scaffolded by
`generator-code`. To make that go smoothly, we'll rename the generated
`README.md` to avoid conflict and then copy:

```sh
mv vscode-markright/README.md vscode-markright/README-code.md
Get-ChildItem -Path vscode-markright | Move-Item -Force -Destination .
rm vscode-markright
```

This move has replaced our `.gitignore`, but that's fine, we want the template
one.

With the extension template scaffolded, let's install the NPM dependencies so
that we get code completion and are able to build:

```sh
npm install
```

Opening `src/extension.ts` we can see the IDE now provides useful Intellisense.

We can also run tests:

```sh
npm test
```

Unfortunately running tests is not supported while VS Code is running.

For some reason, the VS Code extension template `.gitignore` does not include an
entry for `.vscode-test`. Let's fix that, too:

```gitignore .gitignore!
.vscode-test
```

## To-Do

### Use `npm list --global yo` to check Yeoman installation and version

### Use `npm list generator-code` to check `generator-code` local installation

### Consider using `generator-code` non-fork version and feeding its stdin

Looks like this is ugly to do in PowerShell though so maybe not:

https://stackoverflow.com/q/16098366/2715716

### Add CodeLens lines for each MarkRight code block with actions and notes

For shell scripts it would have a Run action so they can be developed in
isolation.

For other code blocks, it would describe what it does, like: *Create test.txt*
(this would resolve `_` to the last file name), *Update test.txt* (it would know
based on the contents of the directory) etc.

### Fix `generator-code` not being installed top-level all of the sudden

```
Actual:   "+-- generator-code@1.3.5 (github:tomashubelbauer/vscode-generator-code#a962e050a"… (1 lines, 113 chars)
Expected: "`-- generator-code@1.3.5 (github:tomashubelbauer/vscode-generator-code#a962e050a"… (1 lines, 113 chars)
```
