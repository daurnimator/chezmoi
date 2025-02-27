# chezmoi How-To Guide

* [Use a hosted repo to manage your dotfiles across multiple machines](#use-a-hosted-repo-to-manage-your-dotfiles-across-multiple-machines)
* [Use templates to manage files that vary from machine to machine](#use-templates-to-manage-files-that-vary-from-machine-to-machine)
* [Create a config file on a new machine automatically](#create-a-config-file-on-a-new-machine-automatically)
* [Keep data private](#keep-data-private)
  * [Use Bitwarden to keep your secrets](#use-bitwarden-to-keep-your-secrets)
  * [Use gpg to keep your secrets](#use-gpg-to-keep-your-secrets)
  * [Use KeePassXC to keep your secrets](#use-keepassxc-to-keep-your-secrets)
  * [Use a keyring to keep your secrets](#use-a-keyring-to-keep-your-secrets)
  * [Use LastPass to keep your secrets](#use-lastpass-to-keep-your-secrets)
  * [Use 1Password to keep your secrets](#use-1password-to-keep-your-secrets)
  * [Use pass to keep your secrets](#use-pass-to-keep-your-secrets)
  * [Use Vault to keep your secrets](#use-vault-to-keep-your-secrets)
  * [Use a generic tool to keep your secrets](#use-a-generic-tool-to-keep-your-secrets)
  * [Use templates variables to keep your secrets](#use-templates-variables-to-keep-your-secrets)
* [Use scripts to perform actions](#use-scripts-to-perform-actions)
  * [Understand how scripts work](#understand-how-scripts-work)
  * [Install packages with scripts](#install-packages-with-scripts)
* [Import archives](#import-archives)
* [Export archives](#export-archives)
* [Use a non-git version control system](#use-a-non-git-version-control-system)
* [Use a merge tool other than vimdiff](#use-a-merge-tool-other-than-vimdiff)
* [Migrate from a dotfile manager that uses symlinks](#migrate-from-a-dotfile-manager-that-uses-symlinks)

## Use a hosted repo to manage your dotfiles across multiple machines

chezmoi relies on your version control system and hosted repo to share changes
across multiple machines. You should create a repo on the source code repository
of your choice (e.g. [Bitbucket](https://bitbucket.org),
[GitHub](https://github.com/), or [GitLab](https://gitlab.com), many people call
their repo `dotfiles`) and push the repo in the source directory here. For
example:

    chezmoi cd
    git remote add origin https://github.com/username/dotfiles.git
    git push -u origin master
    exit

On another machine you can checkout this repo:

    chezmoi init https://github.com/username/dotfiles.git

You can then see what would be changed:

    chezmoi diff

If you're happy with the changes then apply them:

    chezmoi apply

The above commands can be combined into a single init, checkout, and apply:

    chezmoi init --apply --verbose https://github.com/username/dotfiles.git

You can pull the changes from your repo and apply them in a single command:

    chezmoi update

This runs `git pull --rebase` in your source directory and then `chezmoi apply`.

## Use templates to manage files that vary from machine to machine

The primary goal of chezmoi is to manage configuration files across multiple
machines, for example your personal macOS laptop, your work Ubuntu desktop, and
your work Linux laptop. You will want to keep much configuration the same across
these, but also need machine-specific configurations for email addresses,
credentials, etc. chezmoi achieves this functionality by using
[`text/template`](https://godoc.org/text/template) for the source state where
needed.

For example, your home `~/.gitconfig` on your personal machine might look like:

    [user]
      email = "john@home.org"

Whereas at work it might be:

    [user]
      email = "john@company.com"

To handle this, on each machine create a configuration file called
`~/.config/chezmoi/chezmoi.toml` defining what might change. For your home
machine:

    [data]
      email = "john@home.org"

If you intend to store private data (e.g. access tokens) in
`~/.config/chezmoi/chezmoi.toml`, make sure it has permissions `0600`. See
"Keeping data private" below for more discussion on this.

If you prefer, you can use any format supported by
[Viper](https://github.com/spf13/viper) for your configuration file. This
includes JSON, YAML, and TOML.

Then, add `~/.gitconfig` to chezmoi using the `-T` flag to automatically turn
it in to a template:

    chezmoi add -T ~/.gitconfig

You can then open the template (which will be saved in the file
`~/.local/share/chezmoi/dot_gitconfig.tmpl`):

    chezmoi edit ~/.gitconfig

The file should look something like:

    [user]
      email = "{{ .email }}"

chezmoi will substitute the variables from the `data` section of your
`~/.config/chezmoi/chezmoi.toml` file when calculating the target state of
`.gitconfig`.

For more advanced usage, you can use the full power of the
[`text/template`](https://godoc.org/text/template) language to include or
exclude sections of file. For a full list of variables, run:

    chezmoi data

For example, in your `~/.local/share/chezmoi/dot_bashrc.tmpl` you might have:

    # common config
    export EDITOR=vi

    # machine-specific configuration
    {{- if eq .chezmoi.hostname "work-laptop" }}
    # this will only be included in ~/.bashrc on work-laptop
    {{- end }}

chezmoi includes all of the hermetic text functions from
[sprig](http://masterminds.github.io/sprig/).

If, after executing the template, the file contents are empty, the target file
will be removed. This can be used to ensure that files are only present on
certain machines. If you want an empty file to be created anyway, you will need
to give it an `empty_` prefix. See "Under the hood" below.

For coarser-grained control of files and entire directories are managed on
different machines, or to exclude certain files completely, you can create
`.chezmoiignore` files in the source directory. These specify a list of patterns
that chezmoi should ignore, and are interpreted as templates. An example
`.chezmoiignore` file might look like:

    README.md
    {{- if ne .chezmoi.hostname "work-laptop" }}
    .work # only manage .work on work-laptop
    {{- end }}

Patterns can be excluded by prefixing them with a `!`, for example:

    f*
    !foo

will ignore all files beginning with an `f` except `foo`.

## Create a config file on a new machine automatically

`chezmoi init` can also create a config file automatically, if one does not
already exist. If your repo contains a file called `.chezmoi.<format>.tmpl`
where *format* is one of the supported config file formats (e.g. `json`, `toml`,
or `yaml`) then `chezmoi init` will execute that template to generate your
initial config file.

Specifically, if you have `.chezmoi.toml.tmpl` that looks like this:

    {{- $email := promptString "email" -}}
    [data]
        email = "{{ $email }}"

Then `chezmoi init` will create an initial `chezmoi.toml` using this template.
`promptString` is a special function that prompts the user (you) for a value.

## Keep data private

chezmoi automatically detects when files and directories are private when adding
them by inspecting their permissions. Private files and directories are stored
in `~/.local/share/chezmoi` as regular, public files with permissions `0644` and
the name prefix `private_`. For example:

    chezmoi add ~/.netrc

will create `~/.local/share/chezmoi/private_dot_netrc` (assuming `~/.netrc` is
not world- or group- readable, as it should be). This file is still private
because `~/.local/share/chezmoi` is not group- or world- readable or executable.
chezmoi checks that the permissions of `~/.local/share/chezmoi` are `0700` on
every run and will print a warning if they are not.

It is common that you need to store access tokens in config files, e.g. a
[GitHub access
token](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/).
There are several ways to keep these tokens secure, and to prevent them leaving
your machine.

### Use Bitwarden to keep your secrets

chezmoi includes support for [Bitwarden](https://bitwarden.com/) using the
[Bitwarden CLI](https://github.com/bitwarden/cli) to expose data as a template
function.

Log in to Bitwarden using:

    bw login <bitwarden-email>

Unlock your Bitwarden vault:

    bw unlock

Set the `BW_SESSION` environment variable, as instructed.

The structured data from `bw get` is available as the `bitwarden` template
function in your config files, for example:

    username = {{ (bitwarden "item" "example.com").login.username }}
    password = {{ (bitwarden "item" "example.com").login.password }}

### Use gpg to keep your secrets

chezmoi supports encrypting files with [gpg](https://www.gnupg.org/). Encrypted
files are stored in the source state and automatically be decrypted when
generating the target state or printing a file's contents with `chezmoi cat`.
`chezmoi edit` will transparently decrypt the file before editing and re-encrypt
it afterwards.

#### Asymmetric (private/public-key) encryption

Specify the encryption key to use in your configuration file (`chezmoi.toml`)
with the `gpg.recipient` key:

    [gpg]
      recipient = "..."

Add files to be encrypted with the `--encrypt` flag, for example:

    chezmoi add --encrypt ~/.ssh/id_rsa

chezmoi will encrypt the file with:

    gpg --armor --recipient ${gpg.recipient} --encrypt

and store the encrypted file in the source state. The file will automatically be
decrypted when generating the target state.

#### Symmetric encryption

Specify symmetric encryption in your configuration file:

    [gpg]
      symmetric = true

Add files to be encrypted with the `--encrypt` flag, for example:

    chezmoi add --encrypt ~/.ssh/id_rsa

chezmoi will encrypt the file with:

    gpg --armor --symmetric

### Use KeePassXC to keep your secrets

chezmoi includes support for [KeePassXC](https://keepassxc.org) using the
KeePassXC CLI (`keepassxc-cli`) to expose data as a template function.

Provide the path to your KeePassXC database in your configuration file:

    [keepassxc]
      database = "/home/user/Passwords.kdbx"

The structured data from `keepassxc-cli show $database` is available as the
`keepassxc` template function in your config files, for example:

    username = {{ (keepassxc "example.com").UserName }}
    password = {{ (keepassxc "example.com").Password }}

### Use a keyring to keep your secrets

chezmoi includes support for Keychain (on macOS), GNOME Keyring (on Linux), and
Windows Credentials Manager (on Windows) via the
[`zalando/go-keyring`](https://github.com/zalando/go-keyring) library.

Set passwords with:

    $ chezmoi keyring set --service=<service> --user=<user>
    Password: xxxxxxxx

The password can then be used in templates using the `keyring` function which
takes the service and user as arguments.

For example, save a GitHub access token in keyring with:

    $ chezmoi keyring set --service=github --user=<github-username>
    Password: xxxxxxxx

and then include it in your `~/.gitconfig` file with:

    [github]
      user = "{{ .github.user }}"
      token = "{{ keyring "github" .github.user }}"

You can query the keyring from the command line:

    chezmoi keyring get --service=github --user=<github-username>

### Use LastPass to keep your secrets

chezmoi includes support for [LastPass](https://lastpass.com) using the
[LastPass CLI](https://lastpass.github.io/lastpass-cli/lpass.1.html) to expose
data as a template function.

Log in to LastPass using:

    lpass login <lastpass-username>

Check that `lpass` is working correctly by showing password data:

    lpass show --json <lastpass-entry-id>

where `<lastpass-entry-id>` is a [LastPass Entry
Specification](https://lastpass.github.io/lastpass-cli/lpass.1.html#_entry_specification).

The structured data from `lpass show --json id` is available as the `lastpass`
template function. The value will be an array of objects. You can use the
`index` function and `.Field` syntax of the `text/template` language to extract
the field you want. For example, to extract the `password` field from first the
"GitHub" entry, use:

    githubPassword = "{{ (index (lastpass "GitHub") 0).password }}"

chezmoi automatically parses the `note` value of the Lastpass entry, so, for
example, you can extract a private SSH key like this:

    {{ (index (lastpass "SSH") 0).note.privateKey }}

Keys in the `note` section written as `CamelCase Words` are converted to
`camelCaseWords`.

### Use 1Password to keep your secrets

chezmoi includes support for [1Password](https://1password.com/) using the
[1Password CLI](https://support.1password.com/command-line-getting-started/) to
expose data as a template function.

Log in and get a session using:

    eval $(op signin <subdomain>.1password.com <email>)

The structured data from `op get item <uuid>` is available as the `onepassword`
template function, for example:

    {{ (onepassword "<uuid>").details.password }}

### Use pass to keep your secrets

chezmoi includes support for [pass](https://www.passwordstore.org/) using the
pass CLI.

The first line of the output of `pass show <pass-name>` is available as the
`pass` template function, for example:

    {{ pass "<pass-name>" }}

### Use Vault to keep your secrets

chezmoi includes support for [Vault](https://www.vaultproject.io/) using the
[Vault CLI](https://www.vaultproject.io/docs/commands/) to expose data as a
template function.

The vault CLI needs to be correctly configured on your machine, e.g. the
`VAULT_ADDR` and `VAULT_TOKEN` environment variables must be set correctly.
Verify that this is the case by running:

    vault kv get -format=json <key>

The stuctured data from `vault kv get -format=json` is available as the `vault`
template function. You can use the `.Field` syntax of the `text/template`
language to extract the data you want. For example:

    {{ (vault "<key>").data.data.password }}

### Use a generic tool to keep your secrets

You can use any command line tool that outputs secrets either as a string or in
JSON format. Choose the binary by setting `genericSecret.command` in your
configuration file. You can then invoke this command with the `secret` and
`secretJSON` template functions which return the raw output and JSON-decoded
output respectively. All of the above secret managers can be supported in this
way:

| Secret Manager  | `genericSecret.command` | Template skeleton                                 |
| --------------- | ----------------------- | ------------------------------------------------- |
| 1Password       | `op`                    | `{{ secretJSON "get" "item" <id> }}`              |
| Bitwarden       | `bw`                    | `{{ secretJSON "get" <id> }}`                     |
| Hashicorp Vault | `vault`                 | `{{ secretJSON "kv" "get" "-format=json" <id> }}` |
| LastPass        | `lpass`                 | `{{ secretJSON "show" "--json" <id> }}`           |
| KeePassXC       | `keepassxc-cli`         | Not possible (interactive command only)           |
| pass            | `pass`                  | `{{ secret "show" <id> }}`                        |

### Use templates variables to keep your secrets

Typically, `~/.config/chezmoi/chezmoi.toml` is not checked in to version control
and has permissions 0600. You can store tokens as template values in the `data`
section. For example, if your `~/.config/chezmoi/chezmoi.toml` contains:

    [data]
      [data.github]
        user = "<github-username>"
        token = "<github-token>"

Your `~/.local/share/chezmoi/private_dot_gitconfig.tmpl` can then contain:

    {{- if (index . "github") }}
    [github]
      user = "{{ .github.user }}"
      token = "{{ .github.token }}"
    {{- end }}

Any config files containing tokens in plain text should be private (permissions
`0600`).

## Use scripts to perform actions

### Understand how scripts work

chezmoi supports scripts, which are executed when you run `chezmoi apply`. The
scripts can either run every time you run `chezmoi apply`, or only when their
contents have changed.

In verbose mode, the script's contents will be printed before executing it. In
dry-run mode, the script is not executed.

Scripts are any file in the source directory with the prefix `run_`, and are
executed in alphabetical order. Scripts that should only be run when their
contents change have the prefix `run_once_`.

Scripts break chezmoi's declarative approach, and as such should be used
sparingly. Any script should be idempotent, even `run_once_` scripts.

Scripts must be created manually in the source directory, typically by running
`chezmoi cd` and then creating a file with a `run_` prefix.

Scripts with the suffix `.tmpl` are treated as templates, with the usual
template variables available. If, after executing the template, the result is
only whitespace or an empty string, then the script is not executed. This is
useful for disabling scripts.

### Install packages with scripts

Change to the source directory and create a file called
`run_once_install-packages.sh`:

    chezmoi cd
    $EDITOR run_once_install-packages.sh

In this file create your package installation script, e.g.

    #!/bin/sh
    sudo apt install ripgrep

The next time you run `chezmoi apply` or `chezmoi update` this script will be
run. As it has the `run_once_` prefix, it will not be run again unless its
contents change, for example if you add more packages to be installed.

This script can also be a template. For example, if you create
`run_once_install-packages.sh.tmpl` with the contents:

    {{ if eq .chezmoi.os "linux" -}}
    #!/bin/sh
    sudo apt install ripgrep
    {{ else if eq .chezmoi.os "darwin" -}}
    #!/bin/sh
    brew install ripgrep
    {{ end -}}

This will install `ripgrep` on both Debian/Ubuntu Linux systems and macOS.

## Import archives

It is occasionally useful to import entire archives of configuration into your
source state. The `import` command does this. For example, to import the latest
version
[`github.com/robbyrussell/oh-my-zsh`](https://github.com/robbyrussell/oh-my-zsh)
to `~/.oh-my-zsh` run:

    curl -s -L -o oh-my-zsh-master.tar.gz https://github.com/robbyrussell/oh-my-zsh/archive/master.tar.gz
    chezmoi import --strip-components 1 --destination ~/.oh-my-zsh oh-my-zsh-master.tar.gz

Note that this only updates the source state. You will need to run

    chezmoi apply

to update your destination directory.

## Export archives

chezmoi can create an archive containing the target state. This can be useful
for generating target state on a different machine or for simply inspecting the
target state. A particularly useful command is:

    chezmoi archive | tar tvf -

which lists all the targets in the target state.

## Use a non-git version control system

By default, chezmoi uses git, but you can use any version control system of your
choice. In your config file, specify the command to use. For example, to use
Mercurial specify:

    [sourceVCS]
      command = "hg"

The source VCS command is used in the chezmoi commands `init`, `source`, and
`update`, and support for VCSes other than git is limited but easy to add. If
you'd like to see your VCS better supported, please [open an issue on
GitHub](https://github.com/twpayne/chezmoi/issues/new).

## Use a merge tool other than vimdiff

By default, chezmoi uses vimdiff, but you can use any merge tool of your choice.
In your config file, specify the command and args to use. For example, to use
neovim's diff mode specify:

    [merge]
      command = "nvim"
      args = "-d"

## Migrate from a dotfile manager that uses symlinks

Many dotfile managers replace dotfiles with symbolic links to files in a common
directory. If you `chezmoi add` such a symlink, chezmoi will add the symlink,
not the file. To assist with migrating from symlink-based systems, use the
`--follow` / `-f` option to `chezmoi add`, for example:

    chezmoi add --follow ~/.bashrc

This will tell `chezmoi add` that the target state of `~/.bashrc` is the target
of the `~/.bashrc` symlink, rather than the symlink itself. When you run
`chezmoi apply`, chezmoi will replace the `~/.bashrc` symlink with the file
contents.