---
title: "ATutor 2.2.1 Authentication Bypass"
date: 2019-10-01
tags: [ATutor, oswe, awae, ATutor RCE, ATutor sqli, ATutor auth bypass,atutor login bypass, ATutor authentication bypass,atutor exploit auth,offsec, blind sqli, sql injection, Atutor 2.2.1]
header:
  image: "/images/ATutor/header.jpg"
---

Hi! Two days ago i wrote a [post](https://rebraws.github.io/ATutor/) analyzing an old SQL Injection in ATutor 2.2.1, today i'm going to show how we can chain that SQL Injection to bypass authentication, this is going to be a short post because this is very easy to exploit, once we have SQL injection. 


### ATutor from SQL Injection to Bypass Authentication.

![login](/images/ATutor/login.png)
<sub><sup>Figure 1. Login page from ATutor 2.2.1</sup></sub>

At figure 1 we can see the Login page from ATutor, but before reading the source code of the application i want to see how does the login requests looks, so i set up burp and intercepted a login request (See figure 2.)

![](/images/ATutor/loginReq.png)

<sub><sup>Figure 2. Login request intercepted with Burp</sup></sub>

As we can see in Figure 2, the following parameters are sent to the web server
```
form_login_action=true&form_course_id=0&form_password_hidden=ef54344395c598213ae8345db480c6916a25c75a&p=&form_login=test&form_password=&submit=Login
```
From those parameters three of them look interesting, those are "form_password_hidden", "form_login" and "form_password"

Note that the form_password_hidden is not a plaintext password (i used te credentials test:test), so we should take a look about how that hash is generated.

Now that we know how the login request looks, we can start reading the source code to see how the login process works. Below we have a picture of the contents of login.php file

![login.php](/images/ATutor/loginCode.png)
<sub><sup>Figure 3. Code from login.php</sup></sub>

As we see in figure 3. there's not too many things there but we can see that is including two files, vitals.inc.php and login_functions.inc.php, since we are trying to understand how the login proccess works, let's go and take a look at the login_functions file.

![Loginfunctions](/images/ATutor/login_functions.png)
<sub><sup>Figure 4. Code from login_functions.inc.php</sup></sub>


At figure 4 we have the first lines of code from login_functions.inc.php and the first three lines are very important to exploit this authentication bypass
```
if (isset($_POST['token']))
{
    $_SESSION['token'] = $_POST['token'];
}
else
{
    if (!isset($_SESSION['token']))
        $_SESSION['token'] = sha1(mt_rand() . microtime(TRUE));
}
```
Basically is checking if the post parameter "token" is set and if it is, then does: $_SESSION['token'] = $_POST['token'];

If we look back at the login request from Figure 2. there isn't any token parameter there. This means that if we change the request and add that parameter we can set it to whatever we want. 

The next lines of code from Figure 4. are not that important to exploit the vulnerability, so let's just jump to the next important piece of code, see figure 5.

![LoginFunctions2](/images/ATutor/login_functions2.png)
<sub><sup>Figure 5. Code from login_functions.inc.php</sup></sub>

In figure 5. we have the next important lines of code, as we see there, first checks if the POST parameter is set, if we look back at figure 2 we can see that it is, then stores the values from the parameters "form_login_hidden" and "form_login" at the variables $this_password and $this_login and finally sets the variable $used_cookie to false

After the code showed in figure 5, the app uses $addslashes() on both variables ($this_password and $this_login) but as we saw in the previous post this does not do anything.

Finally we can see the next important piece of code in the image from below.

![vulnCode](/images/ATutor/login_functions3.png)
<sub><sup>Figure 5. Code from login_functions.inc.php</sup></sub>

In figure 5. we see that the app calls the queryDB function and if you remember in the previous post we saw that this function creates a query and executes it, but if we pay attention the the array that the app is passing to the function, we will see that is sending the $_SESSION['token']
```
array(TABLE_PREFIX, $this_login, $this_login, $_SESSION['token'], $this_password)
```
And remember that we can set the token to whatever we want.

Now if we pay attention to the query, we'll see that it's only checking if the password we provided is correct or not, but the problem is how is doing that.

```
SHA1(CONCAT(password, '%s'))='%s'
```
Above we have the vulnerable code, first is concatenating the password (FROM THE DATABASE) with the session token, then is creating a SHA1 hash and comparing it to the hashed password from our request, maybe the expression from above looks a little confusing, but basically is doing this:

```
SHA1(CONCAT(password, '$_SESSION['token']'))='$_POST['form_password_hidden']'
```
Since we can set the $_SESSION['token'] to be anything, if we know the hashed password from the database (remember that we have a sql injection, that means that we can get the hashed password) then we can bypass the authentication.

Let's see an example to make it more clear, the hashed password from the Administrator account is the next one

```
78b946e75265cd70834815e3bd922741abdfe2d6
```
We can get this hashed password by exploiting the SQL injection explained in the other post

Now i'm going to craft a POST request and set the token to be "REBRAWS", since i'm going to set the token then the app is going to do this:
```
SHA1(CONCAT('78b946e75265cd70834815e3bd922741abdfe2d6', 'REBRAWS'))='$_POST['form_password_hidden']'
```
Is going to concatenate the hashed password from the DB with the token, then is going to create a SHA1 hash from that string and compare it to "form_password_hidden", in order to bypass the authentication we also have to set the value of form_password_hidden

Below we have an image showing the value of SHA1(CONCAT('78b946e75265cd70834815e3bd922741abdfe2d6', 'REBRAWS'))

![hash](/images/ATutor/hash.png)
<sub><sup>Figure 6. SHA1 Hash generated to bypass auth</sup></sub>

Now that we have the value for from_password_hidden we only have to send a login request with the parameters token=REBRAWS and form_password_hidden=4c3174453fcff6b123d5542b0c0cf163258862a6, see Figure 7.

![bypass](/images/ATutor/bypass.png)
<sub><sup>Figure 7. Crafted request to bypass authentication </sup></sub>

![bypass2](/images/ATutor/bypass2.png)
<sub><sup>Figure 8. Succesfully logged in as Administrator</sup></sub>

Finally in figure 7 we can see the crafted request and in figure 8 we have succesfully got in as Administrator, but remember that this is only exploitable if we know the hashed password of the user and in this case we know it because of the previous SQL injection vulnerability.

Thanks for reading and if you have any suggestion or question you can send me a PM to Twitter! [@Rebraws1](https://twitter.com/Rebraws1)

