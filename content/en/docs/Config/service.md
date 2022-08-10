---
title: "Service"
linkTitle: "Service"
weight: 5
description: >
  Configure the services to monitor and what to notify/action when a new release is found.
---

config.yml
```yaml
service:
  ...
  # As many of these (below) as you like, just ensure they have unique ID's
  EXAMPLE_GITHUB_SERVICE:
    comment: 'something about the service maybe?' # Optional comment about the service
    options:
      active: true               # Disable the service without removing it from config
      interval: 30m              # Interval between both `latest_version` and `deployed_version` queries
      semantic_versioning: true  # Repo follow semantic MAJOR.MINOR.PATCH versioning
    latest_version:                      # Where and how to track the latest version of the service
      type: github
      url: release-argus/Argus           # Monitor this github repo
      access_token: GITHUB_ACCESS_TOKEN  # Useful when you want to exceed the public rate-limit, or want to query a private repo
      use_prerelease: false              # Whether a 'prerelease' tag can be used
      url_commands:
        - type: regex_submatch
          regex: ^v?([0-9.]+)$           # This searches the tag_names. The '$' is used to ensure the tag name
                                         # ends in this RegEx and doesn't just omit a '-beta' or similar details
      require:
        regex_content: 'argus-{{ version }}.linux-amd64'  # Ensure the linux binary has been released before we
                                                          # are alerted about the new version
        regex_version: ^[0-9.]+[0-9]$                     # Version found must match this RegEx to be considered valid
    deployed_version:                   # Get the `current_version` from a deployed service
      url: https://example.com/version  # URL to use
      allow_invalid_certs: false        # Accept invalid HTTPS certs/not
      basic_auth:                       # Credentials for BasicAuth
        username: user
        password: 123
      headers:                          # Headers to send to the URL (Usually an API Key)
        - key: Authorization
          value: 'Bearer <API_KEY>'
      json: data.version                # Use the value of this JSON key as the `current_version`
                                        # (Full path to the key, e.g. `data.version`, not `version`)
      regex: 'v?([0-9.]+)'              # Regex to apply to the data retrieved. Will run after the
                                        # JSON value fetch, or alone (if no JSON)
    notify:                             # Notifiers for this Service
      EXAMPLE_GOTIFY_ID:
        message: 'overriding template'
      EXAMPLE_SLACK_ID:
        message: 'overriding template'
    command:                         # Commands for this Service
      - ["COMMAND", "ARG1", "ARG2"]
    webhook:                               # WebHooks for this Service
      EXAMPLE_WEBHOOK_ID:
        secret: 'service-specific secret'
    dashboard:
      auto_approve: false                           # Whether approval is required for new versions in the Web UI.
      web_url: 'https://example.com/{{ version }}'  # Overrides URL in the Web UI and can be used in the notifiers
      icon: https://example.com/icon.png            # Icon to use on the Web UI
      icon_link_to: https://service.com             # Make the Web UI icon a clickable link to this
```
{{< alert title="Note" >}}
the number of `notify`/`webhook`'s you give the Service can be any number (you can even omit the var altogether).
For these vars, if you provide a var with the same ID as in the globals for that var, the options provided in the service will override those globals. e.g. if you have `slack.main` defined in the `config.yml` with all the required fields, but would like to override the message on some services for example, you can simply set `slack.main.message` inside the service, and it will use that var in messages for this service.
{{< /alert >}}

## Options

```yaml
service:
  example:
    ...
    options:
      active: false              # Disable a service without removing it from the config
      interval: 1h5m             # Query for a version change every 65 minutes
                                # y=years, w=weeks, d=days, h=hours, m=minutes, s=seconds
      semantic_versioning: true  # Whether to enforce semantic versioning on versions queried
                                # (`url_commands` can potentially be used to format versions semantically - https://semver.org)
```

## Latest Version

### GitHub

`lastest_version`'s of type `github` will monitor the most recent `tag_name` that matches both your `regex_content` and `regex_version` at
https://api.github.com/repos/OWNER/REPO/releases.

It will go through each item in that list and try using `tag_name` as the version. It will run the `url_commands` on this version, check it with `regex_version` and then check the assets against `regex_content`. If all of these pass, that version will be used.

```yaml
service:
  example:
    ...
    latest_version:
      type: github                       # The type of service to monitor
      url: OWNER/REPO                    # The GitHub repo to monitor
      allow_invalid_certs: false         # Whether invalid HTTPS certificates on the query site are allowed
      access_token: GITHUB_ACCESS_TOKEN  # Useful when you want to exceed the public rate-limit, or want to query a private repo
      use_prerelease: false              # Whether a 'prerelease' tag can be used
      url_commands:
        - type: regex_submatch
          regex: ^v?([0-9.]+)$  # Since the `type` is 'github', this searches the tag_names, so the '$' is used to ensure
                                # the tag name ends in this RegEx and doesn't just omit a '-beta' or similar details
      require:
        regex_content: 'example-{{ version }}-amd64'  # Release assets of a tag much match this RegEx for the new version to be
                                                      # considered valid (meaning alerts wil fire). This RegEx runs against the
                                                      # version assets `name` and `browser_download_url`
        regex_version: ^[0-9.]+[0-9]$                 # Version found must match this RegEx to be considered valid
```

### URL (Monitor outside GitHub API)

The following is how you'd define a service to be monitored without using the GitHub API. (The key difference is that `type` is 'web' and `url` is a full HTTP(S) address with `url_commands` to scrape the version).

config.yml
```yaml
service:
  example:
    ...
    latest_version:
      type: web                                 # Regular URL, not GitHub API
      url: https://golang.org/dl/               # URL to monitor
      url_commands:                             # Commands to grab the latest version number
        type: regex                    # RegEx type
        regex: go([0-9.]+[0-9]+)\.src\.tar\.gz  # RegEx to find the version. The most recent version download  is linked first
      require:
        regex_content: 'example-{{ version }}-amd64'  # URL queried must contain content with this RegEx for any new version
                                                      # to be considered valid (meaning alerts wil fire)
        regex_version: ^[0-9.]+[0-9]$                 # Version found must match this RegEx to be considered valid
```

### Filters

#### url_commands
`url_commands` are defined in a YAML list, with them being executed in order starting from the top.
There are (currently) four different types of url_command that can be performed:
- [regex](#regex)
- [replace](#replace)
- [split](#split)

{{< alert title="Note" >}}
For a service of type 'github', 'url_commands' will run against every release 'tag_name' until a version is found that matches no 'url_command' fails on and both 'regex_version' and 'regex_content' pass. ('regex_content' acts on every assets 'name' and 'browser_download_url')
{{< /alert >}}

##### regex
```yaml
latest_version:
  ...
  url_commands:
    - type: regex
      regex: v([0-9.]+)$
```
{{< alert title="Note" >}}
This RegEx will return the submatch (the match in the bracket), so `regex` of 'v([0-9])' and text of `v1 v2 v3...` would return 1. To get the 2, you could either use an `index` of 1, or of -2 (second last match). The above RegEx is useful in places where they use the `v` prefix in their versions. Removing that helps in the majority of cases to make it follow semantic versioning.
{{< /alert >}}

##### replace
```yaml
latest_version:
  ...
  url_commands:
    - type: replace
      old: v
      new: ''
```
This command replaces `old` with `new`. e.g. the above would remove `v`

#### split
```yaml
latest_version:
  ...
  url_commands:
    - type: split
      text: test
      index: 0

```
This command will split on `text` and return `index` of that split. e.g. running this command
on '1.2.3test' with the above vars would return `1.2.3`


## Deployed Version

Track the version you have deployed and compare it to the latest_version.

```yaml
service:
  example:
    ...
    deployed_version:                   # Get the `current_version` from a deployed service
      url: https://example.com/version  # URL to use
      allow_invalid_certs: false        # Accept invalid HTTPS certs/not
      basic_auth:                       # Credentials for BasicAuth
        username: user
        password: 123
      headers:                          # Headers to send to the URL (Usually an API Key)
        - key: Authorization
          value: 'Bearer <API_KEY>'
      json: data.version                # Use the value of this JSON key as the `current_version`
                                        # (Full path to the key, e.g. `data.version`, not `version`)
      regex: 'v?([0-9.]+)'              # Regex to apply to the data retrieved. Will run after the
                                        # JSON value fetch, or alone (if no JSON)
```

## command
Under `command`, you can give a list of commands (with or without arguments) to approve and run when a new release is found.
The formatting of `command` is as a list of lists, for example:
```yaml
command:
  - ["COMMAND", "ARG1", "ARG2"...]
  - ["bash", "/opt/script.sh", "foo"]
  - ["bash", "/opt/other.sh", "bar"]
```

## Dashboard options

```yaml
service:
  example:
    ...
    dashboard:
      auto_approve: false                           # Whether approval is required for new versions in the Web UI, or whether
                                                    # WebHooks are automatically sent (required for their delay to be used)
      web_url: 'https://example.com/{{ version }}'  # Overrides URL in the Web UI and can be used in the notifiers
      icon: https://example.com/icon.png            # Icon to use on the Web UI
      icon_link_to: https://service.com             # Make the Web UI icon a clickable link to this
```

### web_url
Without defining this var, the Web UI will link to `url`, but when this var has a value, the Web UI will link to that value.
You could, for example use [message templating](/docs/help/templating), to use all/some variation of the version in this URL and make it link to the sevices changelog (which you could also link to in your update notifiers).
