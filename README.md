# Sthack 2019 - CTF - Ticket Checker
*Creator of the challenge:* Ghozt

*Category:* WEB

*Difficulty:* Medium

## **Step 1: Follow the errors**

Reaching the website, we get a webpage allowing you to scan a ticket (QRCode) using your camera.

Once done, we get the following error:

![error_json](/error_json.png)

Hum...

From Burp, we intercept the request and save it for later.

We use the tool "qrencode" to create a QRCode to put the content I want inside.

For all the next steps, the process will be like the following:
* Create a QRCode using qrencode, containing our data to be tested inside
* Base64 encoded the output of the command
* Send it throught the repeater

Let's start with an empty json:

![json_empty](/qrencode_base.png)
+  *qrencode '{}' -o - | base64 -w 0*
+  *-o -*: output directly on the standard output
+  *-w 0* : disable line wrapping

We put the output into the raw-data of the data image in the "**comment**" parameter.

Be carefull to encode the base64 just added to avoid the following error:
![error_2_encoding](/error_2_encoding.png)

Once encoded:

![parameter](/json_empty.png)

OK ! Missing the key '<b>uid</b>' in the JSON.

![json_uid](/json_uid.png)

## **Step 2: Find the entry point**

Let's add it...

![qrencode_uid](/qrencode_uid.png)

Now we get a correct output telling us that the ticket has already been used:

![output_uid](/output_uid.png)

But if we use one which may be unknown, we get a database error.

![error_db](/error_db.png)

## **Step 3: Injection SQL - Integer Based**

Let's try for some Injection SQL as we are talking to the database!

I will not detail all SQL Injection tested as one using 'single quote', "double quote" and so on... Here, it is an interger based!

We need to know the right number used by the database. Using "order by" with 5 columns, we get an error: 

![order_by](/order.png)

But using 4, it is ok meaning that we have 4 columns used. Output "2" and "3" are displayed, we can use it to exploit our SQL Injection:

![test_1_2_3_4](/union_1_2_3_4.png)

![reflect_value](/reflected_value.png)

Finding the current user, the database... 

![output_sqli](/output_sqli.png)

**Nice** but I can not do anything better with that? Actually yes, let's try to read some local files!

![load_file](/load_file.png)

![check](/check.png)

**Great!!!**
We can leak the source code!

The usefull files to continue the challenge are:
* check.php
* ticket.php

## **Step 4: Read the source code and understand it**

By analyzing the check.php and the Ticket.php files, our goal is to reach the call to the log() function from the Ticket class. In this function, there is a call to the system function with one user input:

![system](/system.png)

It may be a **PHP Object Injection!**. Let's verify it...

Back to the check.php file, if there is no **t_uid, object, and sign** keys in the JSON, the program terminates is execution.

![die4](/die4.png)

We add those keys to continue the execution and pass the condition.

The program get a KEY got from the database which will be used to create a Signature object.

Let's grab it! It would be usefull:

![key](/KEY.png)

![KEY_2](/KEY_2.png)

After getting the key, the program create a Signature Object. Call the method Check(), dies if the check is wrong.

If not, we continue to the unserialize call and the log function from the Ticket class. All leading to the call to the system function!

To pass the signature, we get the source code of the Ticket.php file on our machine to modify it and pass the check. We need to change some part:

* Changing the filename in the History class to avoid our php object to be different at every try on our local test
![filename](/filename.png)
* Hardcoding the key in the Signature class
* Adding the following source code to pass the right value of the **object** and **sign** keys:

![code_modified](/code_modified.png)

I will explain the added part:

* Creation of a new Ticket instance that will be used on the unserialize call
* Creation of a new History instance needed in the Ticket class
* Changing the value of data in the History class. This value will be injected to the system function leading to a some **CODE EXECUTION**
* Linking the new History instance into the Ticket instance
* Serialize the Ticket instance
* Base64 encode the output
* Sign it using the hmac methode took from the Signature class

## **Step 5: Create your most beautiful object to test exploitation!**

The output will be like:

![test_id_rce](/test_id_rce.png)

![output_rce](/id.png)

Let's analyze the server in order to get the flag and finaly have a break to drink a bit!

![laroot](/laroot.png)

![outputflag](/outputflag.png)

## **Step 6: Get the flag, and go drink a beer!** //RCE shell

Get the flag!

![flag](/flag.png)

![flag_output](/flag_output.png)

Flag is: QrSql1_t0_uNs3r1i4lIz3_S0_L0l

![Cheers](/verre-chope.jpg)
