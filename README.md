# TestProf docs in Chinese

This repo contains (will contain) Chinese documentation for [TestProf][].

The docs will be available at the main [documentation website](https://test-prof.evilmartians.io/).

[Translations guide](https://github.com/test-prof/docs/blob/master/TRANSLATIONS.md).

## Linters

We try to keep our documentation both correct and _stylish_ using the following tools:

- [mdl](https://github.com/markdownlint/markdownlint)—Markdown linter, Ruby edition.
- [liche](https://github.com/raviqqe/liche)—links linter.
- [forspell](https://github.com/kkuprikov/forspell)—spelling checker.
- RuboCop with [rubocop-md](https://github.com/rubocop-hq/rubocop-md) and [standard](https://github.com/testdouble/standard)—Ruby code snippets style checking.

To run these tools locally we use [Lefthook](https://github.com/Arkweid/lefthook) (runs linters automatically for every commit).

To sum up:

- Install `mdl`:

```sh
gem install mdl
```

- Install `liche`:

```sh
go get -u github.com/raviqqe/liche
```

- Install Hunspell and Forspell:

```sh
# for MacOS (for other OS see Forspell documentation)
brew install hunspell

gem install forspell
```

- Install StandardRB and `rubocop-md`:

```sh
gem install standard
gem install rubocop-md
```

- Install `lefthook`:

```sh
# for MacOS (for other OS see Lefthook documentation)
brew install lefthook
```

- Initialize `lefthook`:

```sh
lefthook install
```

Or you can skip all of these and rely on our CI, which can do all the checks for you!

[TestProf]: https://github.com/test-prof/test-prof
