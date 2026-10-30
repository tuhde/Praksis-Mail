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
| `mail.move-to` | moves a mail to a folder of its account (what dropping a mail on a folder does) | `mail.move` |
| `mail.purge` | deletes a mail for good, **it cannot be undone** | `mail.delete` |
| `mail.send-message` | sends a mail (`to`, `cc`, `bcc`, `subject`, `text`, attachments), keeps a copy in `params.sent_folder` and remembers the people it went to | `mail.send`, `mail.append`, `contacts.add` (optional) |
| `mail.save-draft` | saves a mail that is not finished in `params.drafts_folder`; nothing is sent | `mail.compose`, `mail.append` |

`mail.send-message` and `mail.save-draft` are only there if the `smtp` module is.

## Which modules

The `maildir` module ships with the runtime image of Praksis. `imap`, `smtp`, `contacts` and `notify`
are in [Praksis-Modules-Mail](https://github.com/tuhde/Praksis-Modules-Mail): build it, then install
what you use and restart the daemon:

```sh
praksis modules install ../Praksis-Modules-Mail/praksis-module-imap/target/imap.praksis-module
praksis modules install ../Praksis-Modules-Mail/praksis-module-smtp/target/smtp.praksis-module
```

(`praksis modules list` shows what is there.) What needs a module that is not installed is left out.

## Sorting by what a mail says (the ai module)

With the `ai` module from [Praksis-Modules-AI](https://github.com/tuhde/Praksis-Modules-AI) the rule
`mail.45-ai-sort` asks a language model which of your labels (`ai_labels`) fits a mail, and moves mail
with one of the labels in `ai_move_labels` to `ai_folder_prefix` + label (`Auto/invoice`). It runs
after the rules for spam, lists and bulk mail and is **off until you switch it on**:

```toml
[param]
ai_sort = true
ai_profile = "local"      # a profile of the ai module: the text of a mail stays on this machine
```

A model can be wrong, and a mail that is moved by mistake is hard to find, so:

1. **Try the model first.** The command **Evaluate the model for sorting mail** (`mail.ai-evaluate`)
   labels the examples in `ai_examples` (put in mails of your own that you know the answer for, with
   the label that is right) and a notification says how many it got right and which ones it got wrong.
   A small model on a CPU is slow (seconds per mail) and gets some wrong; a bigger one does better.
2. **Only labels of your list can move a mail,** whatever the text of the mail says: the answer is
   checked against the list, anything else leaves the mail where it is, and a grant should list the
   folders the rule may move to (`folder = { one_of = ["Auto/invoice", ...] }`) and the profile it may
   ask (`profile = { one_of = ["local"] }`).
3. Mail that is not moved is handled by the rules after it (the notification) as before.

The text of the mail goes to the model as data, apart from the instruction (see the README of the
module). A mail that says "ignore your instructions" does not get far, but **no rule should do
something important because of text that a model has read.**

## Sending mail

Sending leaves the machine and cannot be taken back, so `mail.send` has the effect class `external`:
it always needs a grant, and a rule that sends (an automatic reply, say) is held to the same grants as
a command that a person starts. The copy in the Sent folder is made **after** the mail was sent, so a
failure of the copy never keeps a mail from going out, and a command that is repeated after a crash
puts the copy in place without sending the mail again.

The SMTP module (`smtp`) needs its settings in `praksis.toml` and a credential binding for the password:

```toml
enabled_modules = ["imap", "smtp"]

[module.smtp]
host = "smtp.example.org"
security = "starttls"        # ssl (the default), starttls or none (this machine only)
user = "me@example.org"
credential = "smtp:me"
from = "Me <me@example.org>"
max_per_hour = 30            # no more than this many mails in any hour, in total
```

```toml
# policy.toml
[credential."smtp:me"]
command = ["pass", "show", "mail/example"]
capabilities = ["mail.send"]

[[grant]]
id = "send"
layer = "mail"
capability = "mail.send"

[[grant]]
id = "keep-sent"
layer = "mail"
capability = "mail.append"
[grant.args]
folder = { equals = "Sent" }
```

**A grant only limits the arguments it lists.** To let `mail.send` write to certain people only, list
`to`, `cc` **and** `bcc` in the grant (`[grant.args]`, with `equals`, `one_of` or `prefix`); a grant
that limits `to` alone does not stop a mail from going to anyone on `bcc`. A call that leaves out an
argument that the grant lists is not covered, so a rule that sends should always give `cc` and `bcc`
(an empty text will do), as these commands do.

## Views

| View | What it shows |
|---|---|
| `mail.reader` | three panes: the folders (with the number of unread mails), the mails of the selected folder or of a search (unread in bold, a "More" button for long folders) and the selected mail with its headers, text and attachments. Opening a mail marks it read; buttons mark it unread, star it or move it to the trash |

A view needs a mail module that provides `mail.folders`, `mail.list`, `mail.search`, `mail.get` and
`mail.attachment` (the `maildir` and `imap` modules do), and grants for them in the layer that owns
the view. The buttons also need grants for `mail.mark` and `mail.move` (folder `Trash`). For example:

```toml
[[grant]]
id = "read-folders"
layer = "mail"
capability = "mail.folders"
```

(and the same for `mail.list`, `mail.search`, `mail.get` and `mail.attachment`). Open it in Praksis Desktop on the tab Views.

## Several accounts

Give every account a table of its own in `praksis.toml`, in the module that you use (and the same
names for `smtp`, so that an answer goes out from the account of the mail):

```toml
[module.imap.work]
host = "imap.example.org"
user = "me@example.org"
credential = "imap:work"

[module.imap.private]
host = "imap.example.net"
user = "me"
credential = "imap:private"
```

A mail names its account (in its id), so a rule that moves it, or a command, needs no more. A list
or a search says which account it means with `account`; the reader has a list of accounts above the
folders when there are several. Grants can limit `account` like any other argument. The view
**Accounts** shows how each account is doing (with a button that logs in to every one now), what can
send and how many mails it sent in the last hour, and writes out the configuration for a new
account for you to copy: it changes nothing itself, because your configuration and your grants are
yours to change.

## Daily use

- **Keys** (when you are not typing): `r` reply, `a` reply all, `f` forward, `e` edit a draft, `c`
  new mail, `u` mark unread, `s` star, `S` unstar, `Delete` trash, `t` conversations or single mails,
  `m` more, `/` search, `Ctrl+Enter` send (in the writing view). The keys are written in the views;
  change them there.
- **Drag and drop:** drag a mail onto a folder to move it (the command `mail.move-to`, which needs
  a grant for `mail.move`).
- **Conversations:** the button `Conversations` puts the mails of a conversation together (by
  `References` and `In-Reply-To`), the oldest first, the answers indented. It works on the page that
  is shown, so a conversation whose start is further back starts at the oldest mail on the page.
- **Notifications:** with the `notify` module, mail from people (not spam, lists or bulk mail) shows
  a notification with the sender and the subject, naming the account if there are several, even if
  no window is open. At most five a minute are shown (`max_per_minute`); the others are counted and
  the next one says how many were left out.

```toml
[module.notify]
command = ["notify-send", "--app-name=Praksis"]
max_per_minute = 5
```

## Writing mail

With the `smtp` module the reader has the buttons **New mail**, **Reply**, **Reply all** and **Forward**
(and **Edit draft** in the drafts folder), and the view `mail.compose` for writing:

- the recipients are completed as you type from the address book (the `contacts` module, optional),
  which learns from mail that people send you (not spam, lists or bulk mail: the rule `mail.40-learn`
  comes after the rules that stop those) and from the mail you send;
- a reply goes to the `Reply-To` address if there is one, to nobody twice and not to you; the subject
  has one `Re:` however many there were, the thread is kept in `In-Reply-To` and `References`, and the
  mail you answer is quoted below your signature;
- a forward has the original below a line that says so, with its headers, and starts no thread;
- **Attach...** copies the file you choose into a private directory of Praksis; the mail is built
  from that copy, so no rule can attach a file that you did not choose;
- **Send** asks first (it leaves your computer) and then returns to the reader; **Save draft** keeps
  what is written and stays where it is. Saving again makes another draft (the old one is trashed by
  hand), and a draft that is edited again does not bring its attachments back.

Your signature is the ruleset parameter `signature`:

```toml
[param]
signature = "Till\nhttps://example.org"
```

What a ruleset offers that needs a module you did not enable is **left out, not an error**: without
`smtp` there is no Write view and no Reply button, without `contacts` there is no completion and
nobody is remembered, and everything else works. (A rule, a view, a part of a view or a step of a rule
can say `requires_modules = ["smtp"]`.) The daemon says what it left out in its log.

For several accounts, for the notifications and for the list of accounts, add grants for
`mail.accounts`, `mail.senders` and `notify.show` in the same way. For writing, add to your grants (the layer is the one you gave this ruleset):

```toml
# reading what an answer starts from, and the completion
[[grant]]
id = "mail-get"
layer = "mail"
capability = "mail.get"
[[grant]]
id = "mail-reply"
layer = "mail"
capability = "mail.reply"
[[grant]]
id = "mail-forward"
layer = "mail"
capability = "mail.forward"
[[grant]]
id = "contacts-search"
layer = "mail"
capability = "contacts.search"
# the commands: building, sending and keeping the mail, and the address book
[[grant]]
id = "mail-compose"
layer = "mail"
capability = "mail.compose"
[[grant]]
id = "contacts-add"
layer = "mail"
capability = "contacts.add"
```

(`mail.send` and `mail.append` as above.) The address book is in `~/.praksis/data/contacts/`, and
`praksis grants` shows `contacts.remove` as destructive if you want to be able to forget an address.

## Reading mail safely

Mail is untrusted input, and the reader is built so that a mail cannot do anything:

- **HTML** is turned into text. Only paragraphs, headings, lists, quotes, bold, italic, code and links
  are kept; scripts, styles, frames, forms and everything else are dropped with their content. The
  window has no browser engine, so there is nothing that could run a script.
- **Links** are shown as links only if they go to `http`, `https` or `mailto`. Clicking one shows the
  whole address and asks before it is opened in your browser.
- **Remote images** are not loaded. The mail says how many there are, and a button loads them for
  that mail only (the next mail is blocked again). Nothing is fetched before you press it.
- **Attachments** are never opened or run. `Save...` asks where to put the file and writes it there.

## Parameters

| Name | Default |
|---|---|
| `spam_folder` | `Spam` |
| `lists_folder` | `Lists` |
| `bulk_folder` | `Bulk` |
| `trash_folder` | `Trash` |
| `sent_folder` | `Sent` |
| `drafts_folder` | `Drafts` |
| `signature` | empty |

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
