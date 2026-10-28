# Praksis-Mail

General rules for sorting incoming mail, for [Praksis](https://github.com/tuhde/Praksis).
Use it as the bottom layer and put your own layer on top.

| Rule | What it does |
|---|---|
| `mail.10-spam` | Mail with `X-Spam-Flag: YES` goes to `params.spam_folder` |
| `mail.20-lists` | Mail with a `List-Id` header goes to `params.lists_folder` |
| `mail.30-bulk` | Mail with `Precedence: bulk` or `junk` goes to `params.bulk_folder` |

Each rule moves the mail and then ends with `stop`, because a moved mail cannot be handled by a
later rule. Rules fire in the order of their ids, so the numbers decide which rule wins when
several fit. The gaps between them leave room for rules in between.

## Parameters

| Name | Default |
|---|---|
| `spam_folder` | `Spam` |
| `lists_folder` | `Lists` |
| `bulk_folder` | `Bulk` |

Set your own values in `praksis.toml`:

```toml
[param]
lists_folder = "Mailing lists"
```

## Use

In `praksis.toml`:

```toml
[[layer]]
name = "mail"
git  = "https://github.com/tuhde/Praksis-Mail"
ref  = "main"
```

Then `praksis layers update` pins the current commit. Later updates are always your decision:
`praksis layers update` shows what changes and refuses to move the pin if a layer on top no longer
fits.

## Changing a rule in another layer

Do not edit this repository for personal wishes. Add `patch/<id>.toml`, `replace/<id>.toml` or
`disable/<id>.toml` in your own layer. `praksis effective` shows the `sha256:` value that
`based_on` needs.

## Needs

The module `maildir` (event `mail.received` with the fields `list_id`, `precedence` and
`x_spam_flag`, capability `mail.move`).

## License

Apache License 2.0, see [LICENSE](LICENSE). Every rule file carries an SPDX header.
