---
layout: post
title: "The Perfect Remote Pair Environment"
date: 2013-06-25 10:37
comments: true
categories: tmux,vim,pair
---

I recently started working with a remote programmer and quickly realized we
needed an effective way to pair remotely. Using a ssh, vim, and tmux (along
with some other nifty tools) I was able to set up a powerful pair environment
in minutes. Here are the steps:

## 1. Have a box you can SSH into

I like using
[Linode](http://www.linode.com/?r=baecb63cb7bb6144d5328e91159225e45e876115)
(full disclosure: that link has a referral code), but any \*nix box you can SSH
into will do. All following instructions are to be performed on your remote
box, unless otherwise specified.

## 2. Create a new user account

The first thing I did was add a new user called 'pair' to a Linode I use for
development.

```
sudo add_user pair
```

I've seen ways to [share tmux sessions between two user
accounts](http://readystate4.com/2011/01/02/sharing-remote-terminal-session-between-two-users-with-tmux/),
but then I cannot let my pair wrap up work on *my* tmux session without
watching them (I have trust issues). But using an account just for pairing
means:

* no worries about pair partner messing with my personal files
* it's easy to have config files tweaked for pairing
* I can let pair partners log in via their public key (they don't need the
  password and I can remove their key at any time)

## 3. Configure tmux / vim

For those who aren't aware, [tmux](http://tmux.sourceforge.net/) is a termainal
multiplexer--it lets you easily switch between programs in one terminal. It's a
fantastic piece of software and if you haven't used it before I highly
recommend you take a look.

After setting up the user I copied over my vimrc and tmux configurations from
my dotfiles repo. One issue is that users (like myself) prefer their own
configss. The nice thing about having a separate pair account is that it's
easier to compromise on config changes. After all, it's not a change to *my*
vimrc, it's our shared vimrc.

My pair partner is a big fan of the [jk smash](http://vimbits.com/bits/180) to
switch back to normal mode.  When we were pairing, his instincts would take
over and he would add 'jk' to the end of lines of code. I told him just
to add it in to the vimrc, and that was that.

Note: Screen would also work fine instead of tmux. As would emacs or any other
terminal editor in place of vim. But tmux/vim is my preference.

## 4. SSH Keys

No need to give out the password to the pair account. Just ask your pair for
their SSH key and copy it over yourself. Paste it into ~/.ssh/authorized_keys
within your pair account. If you ever want to revoke their access, just remove
their key (note that this won't close an existing session).

## 5. Local SSH Config / Port Forwarding

I mostly work on Rails apps. Though we are often just looking at code and
running tests while in the pair environment, it's sometimes helpful to interact
with the app in the browser to diagnose issues and check behavior. With port
forwarding, we can run the dev Rails server in our pair environment and access
it using our browsers locally.

The ssh command to log in with forwarding looks like:

```
ssh pair@server.url -L 3000:127.0.0.1:3000
```

With this port forwarding, I can open my browser and go to `localhost:3000` to
access the dev Rails server running on the remote box. That's quite a bit to
type out every time, so I added an alias to my `.ssh/config`:

```
Host pair
  HostName server.url
  User pair
  LocalForward 3000 127.0.0.1:3000
```

Wit the alias I can log in with just `ssh pair`.

## 6. Git Pairing

When we pair it's nice to have the git log reflect that we worked together on a
piece of code. Fortunately, the
[git-pairing](https://github.com/glg/git-pairing) gem makes this easy. It
allows you to pre-define several pair partners who you can identify by their
initials. When you want to commit as a pair, just type:

```
git pair initials1 initials2
```
And when you're working solo:
```
git solo
```

## Conclusion

The pair environment + a phone call / skype makes working remotely incredibly
easy. I've found this setup to be just as easy as working side-by-side in
person. Perhaps even better, because when we're done pairing I can logout of
the pair server and get right back to working locally.
