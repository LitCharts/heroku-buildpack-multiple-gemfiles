# Multiple Gemfiles for Heroku

A buildpack that gives your Ruby app the ability to run with multiple Gemfiles. It works
alongside Heroku's ruby buildpack.

You may want to use this during Rails upgrades to run two Rails versions simultaneously in the
same Heroku environment. For example, you could have some of your dynos running the current Rails version, and the rest of your dynos running the next Rails version.

- Capable of running 2 different gemfiles `Gemfile` and `Gemfile_next` from one Heroku slug
- Configurable to run tests against both gemfiles when using Heroku CI parallel
- Allows you to run two Rails versions in production
- Uses an env var to specify which dynos you want to use `Gemfile_next`

## Setup

1. **WARNING** You should not assume my copy of this repo to continue to exist, or be stable, or secure for your application in the future. Fork this repo so you have a copy under your control under your own GitHub account.

2. Add a buildpack after your existing heroku/ruby buildpack. For `app.json` see below. Replace `YOUR-GITHUB-ACCOUNT` with your GitHub user/org name.

```
"buildpacks": [
  { "url": "heroku/ruby" },
  { "url": "https://github.com/YOUR-GITHUB-ACCOUNT/heroku-buildpack-multiple-gemfiles" }
]
```

If you want Heroku CI support, you'll also need to include this buildpack in the `environments > test > buildpacks` array in `app.json`.

3. Create `Gemfile_next` with its dependencies in your project root. This is the same directory as your existing `Gemfile`.

4. Generate `Gemfile_next.lock` by running:
```sh
BUNDLE_GEMFILE=Gemfile_next bundle install
```

5. Commit `Gemfile_next` and `Gemfile_next.lock` to your git repository.

6. For Heroku CI support, set the `DEPENDENCIES_NEXT_CI_NODES` environment variable (in the `environments > test > env` object in `app.json`) to a comma-separated string of node indexes you want to use `Gemfile_next`.

For example, if you have 4 CI nodes in total, and you want the first 2 nodes to use `Gemfile` (the default), and the last 2 nodes to use `Gemfile_next`, set the following:

```
"DEPENDENCIES_NEXT_CI_NODES": "2,3",
```

(CI nodes are zero-indexed.)

7. To specify which dynos use `Gemfile_next`, set the `DEPENDENCIES_NEXT_DYNOS` environment variable to a comma-separated string of dyno names.

`DEPENDENCIES_NEXT_DYNOS=web.5,web.6,worker.3` would cause dynos `web.5`, `web.6`, and `worker.3` to use `Gemfile_next`. All other dynos would use `Gemfile` by default.


## How it works

When Heroku builds your app, these additional steps are performed by this buildpack:

- Installs gems specified in `Gemfile_next.lock` into your slug.

- Writes a shell script in your dyno's `.profile.d/` directory which sets environment variables on startup `BUNDLE_GEMFILE=Gemfile_next` and `DEPENDENCIES_NEXT=true` for the dynos you specify in the `DEPENDENCIES_NEXT_DYNOS` environment variable (or `DEPENDENCIES_NEXT_CI_NODES` for Heroku CI).


## Rollback to Gemfile

If you need to rollback to using `Gemfile` on a dyno instead of `Gemfile_next`, change the value of `DEPENDENCIES_NEXT_DYNOS`. All dynos will restart and only those dynos specified in `DEPENDENCIES_NEXT_DYNOS` will use `Gemfile_next`. All other dynos will use `Gemfile`. If you want no dynos to use `Gemfile_next`, you can delete the `DEPENDENCIES_NEXT_DYNOS` environment variable.
