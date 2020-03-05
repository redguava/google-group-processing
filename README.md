## Google Group Message Processing

# Summary

This process has several steps.

First, all messages from the group need to be downloaded locally.

Next, we iterated through these messages and processed them. For us, this meant discarding the HTML verions of the messages, and inserting the information we cared about into an SQLite database.

After that, we could query and filter them, and export the information we needed for a bulk mail to CSV.

## Downloading

Messages were downloaded with a modification to this project: https://github.com/geniusgordon/go-google-group-crawler

Our fork of this project, allowing access to private groups and fixing a couple of bugs, can be found here: https://github.com/redguava/go-google-group-crawler

### Preparation

Our fork expects a file named `cookies.txt` to exist in its directory to allow it access to your private group.

To create this file, visit your group in Chrome (while logged in), with the developer console open. In the network tab, find the main page request, right click it and select `Copy as cURL (bash)` from the context menu.

This curl request has a number of parameters, but the one we want is a header (`-H`) parameter beginning with `cookie:`. Copy this entire parameter after the word `cookie: ` into your cookie.txt file, excluding the closing single quote (`'`).

If you don't have it installed already, you'll need to install Go. Instructions can be found here: https://golang.org/doc/install

### Let's get started

run:

```sh
go run crawler.go -o [YOUR_GOOGLE_GROUP_ORGANIZATION] -g [YOUR_GROUP_ID] -t 4
```

This will download all the messages from this group into a subdirectory.

First it will find and download the URLs of all of the threads in the group to `<your group name/threads`.  
Next, it will find and download the URLs of all of the messages in the group to `<your group name/msgs`.  
Finally, the messages themselves, in [RFC 822](https://tools.ietf.org/html/rfc822) format, will be downloaded to `<your group name>/mbox`.

Depending on the size of your group, this can take hours (or longer), and occupy many gigabytes.

If these files are all you need, you can stop here.

## Processing messages

We used the following project to import these output files into a static database: https://github.com/redguava/group-message-parser

[Install Ruby](https://www.ruby-lang.org/en/documentation/installation/) and [bundler](https://bundler.io/), and the sqlite3 development packages, and run

```ruby
bundle install
```

to install the project dependencies.

Next run:

```sh
bin/parser import --dir /path/to/your/mbox/directory --cutoffdate [CUTOFF_DATE] --ignore [IGNORE_EMAIL_DOMAIN]
```

Any messages sent before `CUTOFF_DATE` will be ignored  
Any messages sent from `IGNORE_EMAIL_DOMAIN` will be ignored.

Example:

```sh
bin/parser import --dir ~/my_group/mbox --cutoffdate "20-03-2019" --ignore hotmail.com
```

Will import files from `~/my_group/mbox`, ignoring any messages sent before March 20, 2019, and skip messages sent from the hotmail.com domain.

By default, files will be written to a sqlite file named `messages.db`.

### Importing Intercom contacts

To be able to filter these messages based on whether the sending user was already in Intercom, we exported a CSV of our Intercom contacts, and imported these into a new table in the `messages.db` database file.

In Intercom, click the `Platform` sidebar button, and navigate to `People > All users`.  

In the column list dropdown, deselect all attributes except for `Email` (some cannot be deselected), and from the `More` dropdown, choose `Export users`.

You will receive an email from Intercom when this export is ready to download. Place it in the group-message-parser directory, and run:

```sh
bin/parser import_contacts --contacts=[/path/to/contacts/csv]
```

## Reading and exporting

You can read from the database using the sqlite3 client


```sh
sqlite3 messages.db
```

Some example queries:


### Find unique email addresses
```sql
SELECT distinct from_address from messages;
```

### Some column info

```sql
SELECT
  from_display AS full_email_address,
  from_address AS email_address,
  strftime('%d/%m/%Y', datetime(date, "unixepoch")) as date,
  subject,
  ('https://groups.google.com/a/YOUR_ORGANIZATION/forum/#!topic/YOUR_GROUP_NAME/' || messages.id) AS message_url
  FROM messages;
```

### Find only email addresses which are not in Intercom

```sql
SELECT distinct messages.from_address
FROM messages
LEFT JOIN contacts ON contacts.email = messages.from_address
WHERE contacts.email IS NULL;
```

### Filter messages you probably don't care about

```sql
SELECT * from messages
WHERE
  lower(body) NOT LIKE "%unsubscribe%"
  AND lower(from_address) NOT LIKE "%no-reply";
```

### Creating CSVs

You can write the output of any query to a CSV file with the following:

```sql
headers on
.mode csv
.output your_filename.csv
```
