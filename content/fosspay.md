---
title: "Fosspay"
tags: ['service']
icon: 'devault.jpg'
short_desc: "A self-hosted payment and donation gateway interfaced with Stripe."
draft: true
---
[Fosspay](https://sr.ht/~sircmpwn/fosspay/) is a free-software web frontend for receiving donations and
subscriptions, similar to Patreon or Liberapay, but which can be hosted
on your own server. It can also interface with Patreon or Github
Sponsors to aggregate all your donations.

## Stripe Setup

Fosspay uses [Stripe](https://stripe.com) as a payment processor, which
is a less annoying and more serious and extensible equivalent of PayPal.
You must set up an account with them to be able to receive card payments
through Fosspay.

Note that with Stripe, these payments can be arranged to go directly to
a bank account within a day or so.

Be sure to check out or configure all the Stripe settings you want
before hand. For example, Stripe will automatically include your phone
number on invoices by default, so you might want to change that if you
are not using a dummy number.

### Note

LandChad.net strives for free software, privacy and internet
independence. Using Stripe is better than using companies with a bad
reputation like PayPal, but it is still a large company with can
compromise the privacy of you and others and ban people.

Stripe does do a good job making payments very easy for you: you do not
have to do legally difficult things like storing credit card numbers
(they are sent directly to Stripe via Javascript code). They are also good at
catching fraudulent transactions and other issues that might arise.

But if you want a truly and fully free and open source monetary system
and will not compromise for less, please use Bitcoin or Monero
exclusively. Compared to any other fiat service, Stripe is about the
best, and Fosspay is a great way to self-host a payment gateway.

## Dependencies

We will need git, postgres and the ability to make a python virtual
environment:

```sh
apt install git python3-venv python3-dev postgresql libpq-dev
```

## Download and Installation

We will download fosspay to `/var/www/fosspay/`. This directory will
also serve as our virtual environement.

```sh
git clone https://git.sr.ht/~sircmpwn/fosspay /var/www/fosspay
python3 -m venv /var/www/fosspay
```

Activate the python environment with the command below, then we will
install the dependencies.

```sh
source /var/www/fosspay/bin/activate
cd /var/www/fosspay
pip install -r requirements.txt
```

The `install` command, at the time of this writing, *might* produce many
errors related to the `psycopg2` package. If it does, you can ignore
them, let the command complete and run `apt install python3-psycopg2` to
get the package globally.

Be sure you are still in `/var/www/fosspay`, then we will build the
package and create the configuration file.

```sh
make
cp config.ini.example config.ini
```

## Create a Database

Fosspay uses a postgresql database to store donation information, so
let\'s create a database and user for it.

First, become the `postgres` user and run the `psql` command:

```sh
su postgres
psql
```

We will create a database named `fosspay` controled by a user named
`fosspay` (also identified by a a password `fosspay`).

```sql
create database fosspay ;
create user fosspay with encrypted password 'fosspay' ;
grant all privileges on database fosspay to fosspay ;
\q
```

Note that if you want to use a different username or password for
whatever reason, change them in the command above, but also in the
`connection-string` variable in the configuration file.

## Configuration

Now open up `/var/www/fosspay/config.ini` and we will set things up.
Here are a list of things to edit.

-   `domain` should be set to `donate.example.org`, with your domain.
-   `protocol` can be set to `https`.
-   Get or create an email account to use as a mailer and add the
    account/server information to the email settings.
-   Add your public and secret Stripe keys to the information.
-   Change the `connection-string` to
    `postgresql://fosspay:fosspay@localhost/fosspay` as set up above.

**An important note:** mail ports *must* be opened on the server you\'re
using, or else Fosspay will silently fail to send mails when someone
tries to donate or reset their password. You do not have to run a mail
server on the same server as Fosspay, but either way, mail submission
ports must be opened. This usually requires contacting your VPS provider
and requesting it from them.

### Optional Integration with Patreon, Github, Liberapay

Note that if you have a previous Patreon, Github Sponsors or Liberapay
account, you can create an access token for Fosspay, so that you can
display your income from those sources along side Fosspay monthly
donations.

For Liberapay, you only need to include your username. You must create a
[Github access token](https://github.com/settings/tokens) with the
\"user\" access to interface with it, and you have to add several
[Patreon client
parameters](https://www.patreon.com/portal/registration/register-clients)
for it.

## Nginx configuration

Fosspay runs on port 5000, so we can have Nginx show the site. Create an
Nginx configuration file modeled as below:

```nginx
server {
        listen 80 ;
        listen [::]:80 ;
        server_name donate.example.org ;
        location / {
                proxy_pass http://localhost:5000 ;
        }
}
```

After that, [remember to get HTTPS for the subdomain!](/basic/certbot)
HTTPS is absolutely required for using Stripe as a payment processor.

## Systemd File

We can now create a systemd service file for Fosspay. Create a file in
`/etc/systemd/system/fosspay.service` as below:

```systemd
[Unit]
Description=fosspay website
Wants=network.target
Wants=postgresql.target
Before=network.target
Before=postgresql.target
[Service]
Type=simple
WorkingDirectory=/var/www/fosspay
VIRTUAL_ENV=/var/www/fosspay
Environment=PATH=$VIRTUAL_ENV/bin:$PATH
ExecStart=/var/www/fosspay/bin/gunicorn app:app -b 127.0.0.1:5000
ExecStop=/var/www/fosspay/bin/gunicorn
[Install]
WantedBy=multi-user.target
```

Note that for safety, we are running fosspay through `gunicorn` in our
virtual environment.

We can now run `systemctl start fosspay` to start the service, and it
should appear at the URL you designated above.

## Customizing the Page

Within `/var/www/fosspay/templates`, there are various files that you
can change to add text and other features to the page. The main file is
`summary.html`, where you can add a description and other information
that will appear. Restart the service after updating files to make
changes live.
