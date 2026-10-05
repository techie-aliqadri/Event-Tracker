# Brownie: Event Sign-up App

A small Rails app for creating events and signing people up for them. When an event is full, new sign-ups go on a waitlist. When someone cancels, the longest-waiting person is confirmed automatically.

- Ruby 4.0.7, Rails 8.1.4, SQLite, Hotwire (Turbo), Propshaft, Jbuilder
- Tests: RSpec (`rspec-rails`)

## Models

### `Event`

An event people can sign up for.

| Column | Type | Constraints |
|---|---|---|
| `title` | string | required |
| `description` | text | optional |
| `starts_at` | datetime | required |
| `location` | string | optional |
| `capacity` | integer | required, whole number greater than 0 |

Associations:

- `has_many :registrations, dependent: :destroy`: deleting an event deletes its registrations.
- `has_many :attendees, through: :registrations`

Methods:

- `spots_left`: capacity minus confirmed registrations, never below 0.
- `full?`: true when `spots_left` is 0.
- `promote_from_waitlist!`: confirms waitlisted registrations, oldest first, one per free spot.

### `Attendee`

A person who can register for events.

| Column | Type | Constraints |
|---|---|---|
| `name` | string | required |
| `email` | string | required, valid format, unique (unique index) |

- Emails are stripped of whitespace and lowercased before saving and before lookups, so `ADA@Example.com` and `ada@example.com` are the same person.
- `has_many :registrations, dependent: :destroy`
- `has_many :events, through: :registrations`

### `Registration`

The join model between `Event` and `Attendee`. It carries its own data and rules, which is why it is a model and not a bare join table.

| Column | Type | Constraints |
|---|---|---|
| `event_id` | reference | required, foreign key |
| `attendee_id` | reference | required, foreign key |
| `status` | integer (enum) | `waitlisted` (0), `confirmed` (1), `cancelled` (2); default 0 |

- Unique index on `[event_id, attendee_id]`: one registration per person per event.
- A new registration is **confirmed** while the event has room, otherwise **waitlisted** (set in a `before_validation` callback on create).
- `cancel!` marks the registration cancelled and then calls `event.promote_from_waitlist!`, inside one transaction.
- A cancelled attendee cannot register for the same event again. This is intentional, and the cancel button warns about it.

### Relationships

```
Event 1 ──< Registration >── 1 Attendee
Event  >──────── through ────────<  Attendee   (many-to-many)
```

## Routes

| Method | Path | Action | Purpose |
|---|---|---|---|
| GET | `/` | `events#index` | Home page (the event list) |
| GET | `/events` | `events#index` | List events (HTML or JSON) |
| GET | `/events/new` | `events#new` | New event form |
| POST | `/events` | `events#create` | Create an event |
| GET | `/events/:id` | `events#show` | Event page with the sign-up form, attendees and waitlist (HTML or JSON) |
| GET | `/events/:id/edit` | `events#edit` | Edit form |
| PATCH/PUT | `/events/:id` | `events#update` | Update an event |
| DELETE | `/events/:id` | `events#destroy` | Delete an event and its registrations |
| POST | `/events/:event_id/registrations` | `registrations#create` | Sign up with a name and email |
| DELETE | `/events/:event_id/registrations/:id` | `registrations#destroy` | Cancel a registration |
| GET | `/up` | `rails/health#show` | Health check |

### Registering and cancelling

- **Sign-up form** (on the event page): takes `name` and `email`. If the email already exists, that attendee is reused and the submitted name is ignored. The response is a redirect back to the event with a message: confirmed, or on the waitlist with an explanation of how promotion works.
- **Cancel button**: asks for confirmation (the person will not be able to register again), marks the registration cancelled, and promotes the first waitlisted person.
- Registrations are looked up through the event, so a registration id from another event returns 404.

### JSON

`GET /events` and `GET /events/:id` respond with JSON when the URL ends in `.json` (for example `/events/1.json`) or the request sends `Accept: application/json`. Without either, you get HTML. Rails picks the template whose format matches the request (`index.json.jbuilder` or `index.html.erb`).

Each event includes `id`, `title`, `description`, `location`, `starts_at`, `capacity`, `spots_left`, `confirmed_count`, `waitlisted_count` and `url`. The single-event response also lists `registrations` with each person's `name`, `status` and `registered_at`. Emails are left out on purpose.

Other actions do not have JSON templates.

## How to run it

Prerequisites: Ruby 4.0.7 (see `.ruby-version`) and Bundler.

```
bin/setup --skip-server     # install gems and prepare the database
bin/rails server            # start the app at http://localhost:3000
```

Open the home page, create an event with "New event", then use the form on the event page to register people. Register more people than the capacity to see the waitlist, and cancel a confirmed registration to see the first waitlisted person promoted.

Run the commands from a folder whose name matches the real casing on disk (`Brownie`). On Windows, starting Rails from a differently cased path makes it lose its view templates. After adding a new `app/views/<folder>/`, restart the server.

### Tests and checks

```
bundle exec rspec                                   # full suite
bundle exec rspec spec/models/event_spec.rb         # one file
bundle exec rspec spec/models/event_spec.rb:12      # one example
bin/rubocop                                         # style (rails-omakase)
bin/brakeman --no-pager                             # security scan
```

The suite has model specs (`spec/models/`) and request specs for every controller action (`spec/requests/`).

## Project tooling for Claude Code

- `.claude/settings.json` and `.claude/hooks/run_rspec.rb`: a hook that runs the RSpec suite after Claude edits Ruby, ERB or Jbuilder files under `app/`, `spec/`, `config/` or `db/`. If the suite fails, the failure is sent back to Claude.
- `.claude/skills/new-resource/`: a skill with a script that generates a tested CRUD resource (model, migration, controller, views, JSON, route and specs).
- `CLAUDE.md`: notes for Claude Code sessions in this repository.

## Known gaps

- **CI is not updated yet.** `.github/workflows/ci.yml` and `config/ci.rb` still run `bin/rails test` and trigger on `master`; they need to switch to RSpec and `main`.
- **No authentication.** Anyone can create, edit or delete events and cancel any registration.
- **Not deployed yet.**
