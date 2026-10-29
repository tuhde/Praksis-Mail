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

## Commands

A person starts these (from a view, `praksis run <id> message=<id>`, or the Commands tab). They are
not triggered by events.

| Command | What it does | Needs |
|---|---|---|
| `mail.read` / `mail.unread` | marks a mail read or unread | `mail.mark` |
| `mail.star` | stars a mail (`starred=false` takes the star away) | `mail.mark` |
| `mail.trash` | moves a mail to `params.trash_folder` | `mail.move` |
| `mail.purge` | deletes a mail for good, **it cannot be undone** | `mail.delete` |

## Views

| View | What it shows |
|---|---|
| `mail.reader` | three panes: the folders (with the number of unread mails), the mails of the selected folder or of a search (unread in bold, a "More" button for long folders) and the selected mail with its headers, text and attachments. Opening a mail marks it read; buttons mark it unread, star it or move it to the trash |

A view needs a mail module that provides `mail.folders`, `mail.list`, `mail.search` and `mail.get`
(the `maildir` and `imap` modules do), and grants for them in the layer that owns the view. The
buttons also need grants for `mail.mark` and `mail.move` (folder `Trash`). For example:

```toml
[[grant]]
id = "read-folders"
layer = "mail"
capability = "mail.folders"
```

(and the same for `mail.list`, `mail.search` and `mail.get`). Open it in Praksis Desktop on the tab Views.

## Parameters

| Name | Default |
|---|---|
| `spam_folder` | `Spam` |
| `lists_folder` | `Lists` |
| `bulk_folder` | `Bulk` |
| `trash_folder` | `Trash` |

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

## Tests

`tests/<rule id>.toml` holds the tests of each rule: a mail goes in, the calls the rule makes come
out. Run them with `praksis test`. They run with the defaults of this layer, whatever your
`praksis.toml` sets. Praksis refuses a rule that no test makes fire, so a change to a rule comes
with a change to its tests.

## Needs

A mail module (`maildir` or `imap`) that provides the event `mail.received` with the fields
`message`, `list_id`, `precedence` and `x_spam_flag`, and the capability `mail.move`.

## License

Apache License 2.0, see [LICENSE](LICENSE). Every rule file carries an SPDX header.
