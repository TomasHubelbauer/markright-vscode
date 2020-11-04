# MarkRight VS Code

This repository hosts the source code for the MarkRight VS Code extension. It is
written using MarkRight itself, so it also serves as a test bench for MarkRight.

To run MarkRight on this document, run `npx tomashubelbauer/markright watch`.
This will fetch the latest development version of MarkRight and start it in
watch mode, so any changes to `readme.md` will be reflected by MarkRight.

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

# Ensure `generator-code` in installed locally as a direct dependency
# Print the `generator-code` package version for comparison with expected output
(($(npm ls 2> $null) | where { $_ -match "`-- generator-code" }) -split "@")[1]

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
- Initialize a Git repository?
- Which package manager to use?

Finding the names of the command line arguments corresponding to these questions
is possible by using `yo code --help`. However, not all of the questions were
covered there. By inspecting the source code of the generator:

https://github.com/microsoft/vscode-generator-code/blob/master/generators/app/generate-command-ts%20.js

I found that it uses the methods for asking the questions from this file:

https://github.com/microsoft/vscode-generator-code/blob/master/generators/app/prompts.js

In here it is apparent that the Git initialization and package manager selection
questions do not check for if the options are already provided.

I've filed a question asking about this:

https://github.com/microsoft/vscode-generator-code/issues/226

And after noticing this (the missing check to see if the answers were given), I
have also filed a PR which fixes that:

https://github.com/microsoft/vscode-generator-code/pull/227

Until that PR is fixed, we cannot scaffold the extension. But we can prepare the
command line that should work once the PR is merged:

```
yo code `
  --extensionType command-ts `
  --extensionDisplayName MarkRight `
  --extensionName vscode-markright `
  --extensionDescription "MarkRight support for VS Code" `
  --gitInit false `
  --pkgManager npm `

```
