# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Brownie" is a small Rails 8.1 app (Ruby 4.0.7, SQLite, Hotwire, Propshaft, importmap) built as a capstone: an event sign-up app. Domain: `Event`, `Attendee`, and a `Registration` join model (`has_many :through`, status enum, capacity rule). It is being built in stages on a feature branch; models and routes may not exist yet. Check `app/models` and `config/routes.rb` for the current state.

The capstone checklist requires: green RSpec suite, a `has_many :through` third model, JSON on one resource, CI, a live deploy, a Claude Code hook that runs RSpec after edits, and a script-bearing skill. The README, the "refactor one ugly thing" step and the explanations of what each spec protects against are written by the user, not by Claude.

## Commands

```
bin/setup --skip-server        # install gems, prepare the DB
bin/rails server               # run the app (also: bin/dev)
bundle exec rspec              # full suite
bundle exec rspec spec/models/event_spec.rb        # one file
bundle exec rspec spec/models/event_spec.rb:12     # one example
bin/rubocop                    # lint (rubocop-rails-omakase)
bin/brakeman --no-pager        # static security scan
bin/bundler-audit              # gem vulnerability audit
bin/ci                         # runs config/ci.rb steps locally
```

## Testing

RSpec (`spec/`, `rspec-rails`) replaces the default Minitest suite; the `test/` directory was deleted. `.github/workflows/ci.yml` and `config/ci.rb` still run `bin/rails test` and `test:system` and need to be switched to `bundle exec rspec`. Until that is done they will not run the specs.

## Architecture notes

- Production uses four SQLite databases (primary, cache, queue, cable) under `storage/`; see `config/database.yml`. Development and test use a single database each. Solid Cache, Solid Queue and Solid Cable are enabled, so no Redis is needed.
- Deployment is set up for Docker, Kamal and Thruster (`Dockerfile`, `kamal` gem). A host must persist `storage/` or the SQLite data is lost.
- JSON responses are intended to use `jbuilder` (already in the Gemfile).
- `config/master.key` is present locally and is gitignored; CI would need `RAILS_MASTER_KEY` only if credentials are read in tests.

## Git

Work happens on feature branches (current: `feature/project-tasks`) pushed to `techie-aliqadri/Event-Tracker`, with small, meaningful commits, then merged to `main`. The default branch is `main`, but the generated CI workflow triggers on `master`; fix that when working on CI.
