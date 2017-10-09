---
layout: post
header:
title: Web Scraping III - Data Synchronization 
categories: [Python, Web Scraping]
tags: [Python, Web Scraping, Ubuntu, Database, Bash]
fullview: true
comments: true
---


In this article, I would continue from the previous example - [crawling betting odds from Jockery Club](https://chrisckwong821.github.io/python/web%20scraping/2017/09/21/Splinter.html), to update on the data synchronization using **scp** and **sshpass**.

Previously, the data was stored in csv format, which is quite simple and nice. If the data has a much bigger size, or meaning for more sophisticated usage, we may want to store the data in a database so retrieval of part of the data can be more efficient. With some modification, the output can be formatted into a database one using **sqlite3**.



```python
#output in csv
#recall that rows is just [(data1columnA,data1columnB.....),(another row)]
import csv
with open('football.odds.csv','a') as f:
    writer = csv.writer(f)
    for row in rows:
        writer.writerow(row)
    f.close()
    
#output in database
import sqlite3
conn = sqlite3.connect('football_odds.db')
c = conn.cursor()
c.execute('CREATE TABLE if not exists odds (\
 matchtime TEXT,\
 date TEXT,\
 refresh TEXT,\
 home_team TEXT,\
 away_team TEXT,\
 odd_home INTEGER,\
 odd_draw INTEGER,\
 odd_away INTEGER)')

c.executemany("INSERT INTO odds VALUES (?,?,?,?,?,?,?,?)", rows)
conn.commit()
''' 
dump to a .sql
with open('dump.sql', 'w') as f:
    for line in conn.iterdump():
        f.write('%s\n' % line)
'''
conn.close()

```

For your reference, sqlite3 is a built-in, light-weight library in python to access database and execute SQL. After adjusting the data type, we may proceed to synchronising the data from the remote server to our desktop. In order schedule the update on a regular basis, **cron** is again applied, in addition to **scp** as a utility for file transfer, and **sshpass** for password-enabled login. Assuming your have followed the previous articles, and have the data readily updated in the remote server, steps to grab the data back to your desktop are following:


1. Installing sshpass:
 - brew install https://raw.githubusercontent.com/kadwanev/bigboybrew/master/Library/Formula/sshpass.rb

2. Create a dummy file and give right to modify it:
 - chmod 0755 pathto/filename.db
 
3. To retrieve of file:
 - sudo sshpass -p pwdofyourserver scp -r username@ip:path/filename.db /localpathtostore
 
4. Give right to sshpass so it can be automated in cron.
 - sudo visudo
   As you get into the editor, type this:
 - yourusername ALL = NOPASSWD: path/to/sshpass
   Check the path to sshpass by `which sshpass`. Basiclly this authorizes sshpass to be executed without superuser password.
 
5. Schedule updates in Crontab:
 - crontab -e to get into the editor
 - type * 1 * * * bin/bash /path/to/syncdb.sh
   where syncdb.sh contains the command in Step 3, update is run each hour now.
   
6. Two tips:
 - PATH=outputtopath
   in case system path in cron fails to access sshpass
 - MAILTO=username.com
   to get feedback for debugging at /var/mail/username
 


Coming up next, I would discuss the use of a more production-level scraping tool - Scrapy, on a new problem.
