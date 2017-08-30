# gmailieer

<img src="doc/demo.png">

This program can pull email and labels (and changes to labels) from your GMail
account and store them locally in a maildir with the labels synchronized with a
[notmuch](https://notmuchmail.org/) database. The changes to tags in the
notmuch database may be pushed back remotely to your GMail account.

Other mail pullers (offlineimap, isync/mbsync, etc.) are not needed and will not be used.

If you already have a maildir downloaded, it will not be used.  You have to start from scratch (download from Gmail again) when using gmailieer. Do not try to use a previous maildir. In fact, if you have one, you'll need to use a different directory.

## disclaimer

Gmailieer will not and can not:

* Add or delete messages on your remote account (except syncing the `trash` or `spam` label to messages, and those messages will eventually be [deleted](https://support.google.com/mail/answer/7401?co=GENIE.Platform%3DDesktop&hl=en))
* Modify messages other than their labels

While Gmailieer has been used to successfully synchronize millions of messages and tags by now, it comes with **NO WARRANTIES**.

## requirements

* Python 3
* `tqdm`
* `google_api_python_client` (sometimes `google-api-python-client`)
* `oauth2client`
* `notmuch` python bindings: latest from [git://notmuchmail.org/git/notmuch](https://git.notmuchmail.org/git/notmuch) or `>= 0.25` (when released)
* `setuptools`

In Debian 9 ("Stretch"), for instance, these can be obtained this way:
```sh
$ sudo apt-get -y install python3-tqdm python3-googleapi python3-oauth2client python3-notmuch python3-setuptools
```

## installation

After cloning this repository, symlink `gmi` to somewhere on your path, or use `python setup.py`.

# usage

This assumes your root mail folder is in `~/mail`.  All commands
should be run from the local mail repository unless otherwise specified.

1. First, if you already have a notmuch config, you'll need to move that out of the way:
```sh
$ mv ~/.notmuch-config ~/.notmuch-config.bak
```
(If you want to manage more than one not-much managed maildir, it is possible, but these instructions don't accomodate that. You'll have to edit your ~/.notmuch-config by hand in such a case.)

2. Second, make sure that you make a new directory for your gmailieer-managed maildir.  We're going to do this as a subdirectory of ~/mail, so that you can have more than one maildir. But ours will be only one of them. Example:
```
~/mail <- root mail
~/mail/.notmuch
~/mail/arbitrary.imap.account <- other account, maybe synced with offlineimap
~/mail/gmail.account1 <- first gmailieer-managed account. This can be whatever name you want.
~/mail/gmail.account2 <- second gmailieer-managed acount
```
So let's assume my account is 'tomtomorrow@gmail.com', just for the example.

3. Make a directory for the gmailieer storage and state files
```sh
$ mkdir ~/mail
```

4. Make a subdirectory for your gmail account's maildir
```sh
$ cd ~/mail
$ mkdir gmail-tom  (this could be any directory name you want)
```

5. Now -- again, we're going to assume you do not already have a notmuch config set up. That is, no ~/.notmuch-config file. We need to set up notmuch. 
```sh
$ cd gmail-tom
$ notmuch setup
Your full name [Tom Tomorrow]: _
Your primary email address [xxx]: tomtomorrow@gmail.com
Additional email address [Press 'Enter' if none]:
Top-level directory of your email archive [/home/tt/mail]:
Tags to apply to all new messages (separated by spaces) [unread inbox]:
Tags to exclude when searching messages (separated by spaces) [deleted spam]:
```

6. Now we need to set up your gmailieer-managed subdir (we are still sitting in ~/mail).
```sh
$ mkdir gmail-tom
```

7. And alter notmuch just a little:
  - edit ~/.notmuch-config
  - change the section```
[new]
tags=unread;inbox;
ignore=
```
to read:
```
[new]
tags=new
ignore=*.json;
#tags=unread;inbox;
#ignore=
```
  - and save the file and get back out of your editor
This will allow notmuch to ignore the .json files that gmailieer will create in its maildir directory, and will also tag all new mails with 'new' tag.
  
8. Set up a 'post-new' hook file
```sh
cd .notmuch
mkdir -p hooks
cd hooks
```
  - use your editor to create a 'post-new' file ('hook'). This is a file which instructs notmuch what to do after it has updated its database. (More info on the notmuch docs under [initial tagging](https://notmuchmail.org/initial_tagging/). This will essentially remove the `new` tag each time after notmuch has run. Here is an example ~/mail/.notmuch/hooks/post-new:
```
#!/bin/bash

# immediately archive all messages i myself sent out
notmuch tag -new -- tag:new and from:tomtomorrow@gmail.com

# finally, retag all (remaining) "new" messages "inbox" and "unread"
notmuch tag +inbox +unread -new -- tag:new
```

9. Now initialize notmuch's database even though it will be empty for now since we have not yet pulled down any mail.
```sh
$ cd ~/mail/
$ notmuch new
```

10. Now let's initialize the gmailieer mail storage:
```sh
$ cd gmail-tom # (we should now be sitting in ~/mail/gmail-tom)
$ gmi init tomtomorrow@gmail.com
```
 - `gmi init` will now open your browser and request limited access to your e-mail.
 - Sign in if you need to.
 - Click `Allow`.

> The access token is stored in `.credentials.gmailieer.json` in the local mail repository (i.e., in ~/mail/gmail-tom). If you wish, you can specify [your own api key](#using-your-own-api-key) that should be used.

4. You're now set up, and you can do the initial pull.

> Use `gmi -h` or `gmi command -h` to get more usage information.

# pull

will pull down all remote changes since last time, overwriting any local tag
changes of the affected messages.

Make sure you are sitting in the appropriate directory (in this example case, ~/mail/gmail-tom), before pulling. In other words, you want to be sitting in the directory you want mail pulled into:
```sh
$ pwd ~/mail/gmail-tom
$ gmi pull 
```

the first time you do this, or if a full synchronization is needed it will take longer.

# push

will push up all changes since last push, conflicting changes will be ignored
unless `-f` is specified. these will be overwritten with the remote changes at
the next `pull`.

```sh
$ gmi push
```

# normal synchronization routine

```sh
$ cd ~/.mail/account.gmail
$ gmi sync
```

This effectively does a `push` followed by a `pull`. Any conflicts detected
with the remote in `push` will not be pushed. After the next `pull` has been
run the conflicts should be resolved, overwriting the local changes with the
remote changes. You can force the local changes to overwrite the remote changes
by using `push -f`.

## using your own API key

gmailieer ships with an API key that is shared openly, this key shares API quota, but [cannot be used to access data](https://github.com/gauteh/gmailieer/pull/9) unless access is gained to your private `access_token` or `refresh_token`.

You can get an [api key](https://console.developers.google.com/flows/enableapi?apiid=gmail) for a CLI application to use for yourself. Store the `client_secret.json` file somewhere safe and specify it to `gmi auth -c`. You can do this on a repository that is already initialized.


# caveats

* The GMail API does not let you sync `muted` messages. Until [this Google
bug](https://issuetracker.google.com/issues/36759067) is fixed, the `mute` and `muted` tags are not synchronized with the remote.

* Only one of the tags `inbox`, `spam`, and `trash` may be added to an email. For
the time being, `trash` will be prefered over `spam`, and `spam` over inbox.
