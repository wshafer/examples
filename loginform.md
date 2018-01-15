# Introduction
This tutorial will walk you through setting up a login and authentication
system using PHP.  We will use what is considered best practices and
built in methods that are current at the time of this writing.

While an authentication system may seem like a trivial task, it is in 
fact one of the more complicated things you will actually write for 
virtually all your websites.  For this reason I would always encourage
you to use a framework and built in authentication methods for that
framework.  If you can't use a framework, there are other authentication
systems out there, that are well built and maintained and you should
first try those.  Take a look at [Packagist](https://packagist.org/)
for some.

If you're still reading this, I assume that you either can't or won't
use a prebuilt solution, or you just want to learn how this done
under the covers.  If you fall into one of these categories please
continue on.

### Storing and managing passwords
Before we begin this tutorial we need to have a serious talk about how
to handle and store passwords.

Passwords need to be treated with care.  Remember these passwords will
grant users access to your system, sometime that access contains
even more incredibly sensitive data.  For this reason you should follow
these three very simple rules.

  * **NEVER STORE PASSWORDS**...  I mean it.  NEVER.  
  
I can already here you now.  "How can users login if we don't store the password?"
Well, while we're not going to store the password outright, we will be storing a 
representation of that password.  For this we'll use one way hashes, so
you'll be able to get a hash from a password, but you'll be unable to get
the password back, even if you have the stored hash.  More on this later.

  * **NEVER ECHO PASSWORDS BACK TO THE BROWSER.**

I know the user sent you the password, and it would be nice to have the password
fields pre-populated if the user entered in a bad email on the registration form.
Well, remember we said above to treat passwords with special care.  
What we don't want to have happen is someone sitting in the middle and 
intercepting passwords.  While we absolutely need the password from the user
and that can't be avoided, the user does not need a password from us.  So reduce 
the amount of times the password is transmitted down the wire, and just don't send 
them back.

### Handling user accounts
When it comes to a users account there's really only one thing I try
to watch out for when dealing with authentication:

  * **NEVER TELL A USER IF AN ACCOUNT EXISTS OR NOT**
  
Even though you'll know during the registration and login process if an account 
exists, it's important never let the user know.  If you tell an attacker if
an account exists, what they will do is simply try to brute force the users
password.  If we never tell the user or attacker if an account exists, it
helps to keep these people guessing.  It's not fool proof by any means but
does make it a little bit harder for an attacker to find valid accounts
to brute force.

During this tutorial we will add some additional security to also help with
brute force, but keeping attackers guessing is one of the best tactics you can use 
to help keep your site and your users secure.


### Hashing
Ok so now that we know what not do with passwords let's talk about hashing.
Hashing passwords is by far the most misunderstood thing in this whole process. I
can't tell you how many times I see questions about this on forums, facebook pages,
slack, etc.  Not only are people confused, the answers I see other developers 
give are absolutely horrifying.

First thing to know is that hashing **IS NOT** encryption.  Encryption is meant
to someday be unencrypted. So while storing an encrypted password is a little safer 
then storing the password outright, it's not much safer.  All it would take is 
to crack one password and your encryption would be completely compromised.  Once 
encryption is compromised it's no better then if you have the passwords stored 
in the clear since all the passwords would be encrypted the same way.  On the 
other hand, **hashing** is a generated string that represents the password.  
A hash can only go one-way, so you can get a hash from a password, but you'll 
never get the password from the hash.  If one password is hacked (brute forced), 
the hacker would only have one password, not all of them.

Be careful thought as hashing has it's own caveats that you need to be aware of.
While you can't get the exact password out of a hash, you can get a combination 
of letters that can also generate the same hash.  These are called "collision 
vulnerabilities" and some hashing algorithms are more vulnerable to this then 
others.

So which hashing algorithms should you use?  As of PHP 7.2 the recommendations 
are either using **Bcrypt** or **Argon2**, with **Argon2** being the preferred
right now.  However when dealing with hashing, remember that no matter what 
algorithm is recommended and considered "safe" at the moment, this will only be true 
until it's no longer "safe".  As an example when I began developing 
(a long log time ago) MD5 was the go to hashing algorithm for passwords.  As the years 
past, a quick way was discovered to exploits MD5's collision vulnerabilities in split 
seconds, showing us all how "unsafe" MD5 actually was.

So with this information the question now becomes, how to we secure our passwords
AND keep these passwords secure as new vulnerabilities are discovered and new
algorithms are released? As of PHP 5.5 we now have a solution to this problem.   
PHP now provides an API for hashing passwords AND figuring out if a hash is 
outdated and needs to be updated. These methods are `password_hash()` and 
`password_needs_rehash()` and we will be utilizing these methods in the example below.


### Buyer Beware
In order to show you the concepts of building a login system for your website
I will be taking a lot of shortcuts.  This will be done to get the idea across 
with the minimal amount of code.   I will try my best to mark when I take these 
shortcuts, but please do not use the code you see here in production.  Instead
use this as a starting point for building your own login system.


### Assumptions
This tutorial makes the following assumptions:
  * The reader already has a good understanding of PHP, SQL, and HTML
  * The reader already has PHP installed being served from a web server
  * The reader already has a MySql database available
  
  
# Setup
Before we can start building out the login system we need to get a few things
in place.

First lets create ourselves a users table we will use for logging users into 
the system.  There's nothing special here, just an id, email address, and 
password field.   In a real system, it's likely that you will want to
gather other data for the users, but that data won't be needed for this example.

Run the following SQL:

```mysql
CREATE DATABASE example;

USE example;

CREATE TABLE users (
  id MEDIUMINT NOT NULL AUTO_INCREMENT, 
  email VARCHAR(254) NOT NULL UNIQUE, 
  password VARCHAR(255) NOT NULL,
  locked TINYINT(1) NOT NULL DEFAULT 0,
  attempts TINYINT(1) NOT NULL DEFAULT 0,
  PRIMARY KEY (id)
);
```

<br />

Now that we have some space in the database, let's create some basic HTML
pages that we're going to need in order to handle authentication. 

`index.php`

In this example this is the page we will restrict to only logged in users.

```php
<?php

?>

<html>
<head>
    <title>Restricted Page</title>
</head>
<body>
    <h1>Restricted</h1>
    <p>Only logged in users should see this page.</p>
    <p><button onclick='window.location="logout.php"'>Logout</button></p>
</body>
</html>
```
<br />

`registration.php`

This is the registration form used to create new users

```php
<?php

?>

<html>
<head>
    <title>Registration</title>
</head>
<body>
<h1>Registration</h1>
<form action="registration.php" method="post">
    <p>
        <label>
            Email Address:
            <input type="text" name="email" value="">
        </label>
    </p>

    <p>
        <label>
            Password:
            <input type="password" name="password" value="">
        </label>
    </p>
    
    <p>
        <label>
            Verify Password:
            <input type="password" name="verify" value="">
        </label>
    </p>

    <p>
        <input type="submit" name="submit" value="Login">
    </p>
</form>
</body>
</html>
```

<br />

`login.php`

The login page.

```php
<?php

?>

<html>
<head>
    <title>Login</title>
</head>
<body>
<h1>Login</h1>
<form action="login.php" method="post">
    <p>
        <label>
            Email Address:
            <input type="text" name="email" value="">
        </label>
    </p>

    <p>
        <label>
            Password:
            <input type="password" name="password" value="">
        </label>
    </p>

    <p>
        <input type="submit" name="submit" value="Login">
    </p>

    <p><a href="forgot-password.php" >Forgot Password?</a> </p>
    <p><a href="registration.php" >New User?</a> </p>
</form>
</body>
</html>
```

<br />

`logout.php`

This page will be used to log users out of the system.  In most systems you
will not need a dedicated logout page.  But for simplicity we'll include one
to keep the logic separated out and easier to understand.

```php
<?php

?>

<html>
<head>
    <title>Logout</title>
</head>
<body>
    <p>You are logged out.</p>
    <p><button onclick='window.location="login.php"'>Login</button></p>
</body>
</html>
```

<br />

`forgot-password.php`

No matter how important your system is, it's inevitable that users will
forget their passwords.  This page will be used to begin the process
of resetting a users password.

```php
<?php

?>

<html>
<head>
    <title>Password Reset</title>
</head>
<body>
<h1>Password Reset</h1>
<form action="forgot-password.php" method="post">
    <p>
        <label>
            Email Address:
            <input type="text" name="email" value="">
        </label>
    </p>
    <p>
        <input type="submit" name="submit" value="Reset Password">
    </p>
</form>
</body>
</html>
```

<br />

`reset.php`

This page will be used to actually reset the users password.

```php
<?php

?>

<html>
<head>
    <title>Password Reset</title>
</head>
<body>
<h1>Password Reset</h1>
<form action="reset.php" method="post">
    <p>
        <label>
            New Password:
            <input type="password" name="password" value="">
        </label>
    </p>
    <p>
        <label>
            Verify Password:
            <input type="password" name="verify" value="">
        </label>
    </p>
    <p>
        <input type="submit" name="submit" value="Reset Password">
    </p>
</form>
</body>
</html>
```

Once these pages are completed, please navigate to pages and make sure they are rendering
and all the links are working correctly.


# Registration

Before we can add any kind of authentication or a way for the user to login we need 
a way for users to register with the site.  So let's begin building out the 
registration screen.

First thing we need to do is modify the page to have a default state and be able to display
error messages.

To that let's add some default variables to handle this.

`registration.php`

```php
<?php
/* Setup needed variables and default state */
// Error placeholders
$error = false;  // Was there an error processing the registration?
$errorMsg = '';  // Error message to display

// Fields
$email = '';  // Email Field. 

// Validation
$emailInvalidOrMissing    = false;  // Place holder to show that there's an error with the email 
$passwordInvalidOrMissing = false;  // Place holder to show that there's an error with the password

?>

<html>
<head>
...

```

One thing you might notice right off the bat, is we included a holder for the email
address and not the password.   There's a very good reason for this...
**We never want to send the password back through the web, EVER.**  So instead
of populating this field on an error, we're going to make them type
it in again.


Now lets modify the form to use these variables. We'll also sprinkle in just
a touch of css to help display the error messages

`registration.php`

```php
<html>
<head>
    <title>Registration</title>

    <style type="text/css">
        .error {
            font-weight: bold;
            color: #ff0000;
        }
    </style>
</head>
<body>
<h1>Registration</h1>

<?php if ($error): ?>
<div class="error"><?= $errorMsg; ?></div>
<?php endif; ?>

<form action="registration.php" method="post">
    <p>
        <label <?php if ($emailInvalidOrMissing): ?> class="error" <?php endif; ?>>
            Email Address:
            <input type="text" name="email" value="<?= $email ?>">
        </label>
    </p>

    <p>
        <label <?php if ($passwordInvalidOrMissing): ?> class="error" <?php endif; ?>>
            Password:
            <input type="password" name="password" value="">
        </label>
    </p>

    <p>
        <label <?php if ($passwordInvalidOrMissing): ?> class="error" <?php endif; ?>>
            Verify Password:
            <input type="password" name="verify" value="">
        </label>
    </p>

    <p>
        <input type="submit" name="submit" value="Login">
    </p>
</form>
</body>
</html>
```

Ok the form should be all wired up now.  If you change the default values the form should work
as expected.  Go ahead, give it a try.

Let's now wire up the logic to process the form data and show an error if the data is incorrect.

`registration.php`

```php
<?php
/* Setup needed variables and default state */
// Error placeholders
$error = false;
$errorMsg = '';

// Fields
$email = '';

// Validation
$emailInvalidOrMissing    = false;
$passwordInvalidOrMissing = false;


if (!empty($_POST)) {

    // Never trust data from the end users.  Always validate your inputs.

    /* Data Validation */
    $email = filter_var(
        $_POST['email'] ?? '', 
        FILTER_VALIDATE_EMAIL
    );
    
    $password = filter_var(
        $_POST['password'] ?? '', 
        FILTER_SANITIZE_STRING
    );
    
    $verify = filter_var(
        $_POST['verify'] ?? '', 
        FILTER_SANITIZE_STRING
    );

    /* Make sure we have all the data we need */
    if (empty($email)) {
        $emailInvalidOrMissing = true;
    }

    if (empty($password)) {
        $passwordInvalidOrMissing = true;
    }

    if (empty($verify)) {
        $passwordInvalidOrMissing = true;
    }
    
    if ($password !== $verify) {
        $passwordInvalidOrMissing = true;
    }

    if ($emailInvalidOrMissing || $passwordInvalidOrMissing) {
        $error = true;
        $errorMsg = 'Unable to complete registration. Please make sure to complete the questions below and try again';
    }

    // SHORTCUT.  Generally we would have abstracted out our logic from html, making
    // errors and validations a lot easier to deal with.  For this tutorial we are
    // not creating the usual abstractions, so we're cheating here a little.
    // Please don't do this in the real world.
    if ($error) {
        goto breaker;
    }
}

//  Cheating goto marker.  Please don't do this in the real world.
breaker:

?>

<html>
...
```

Ok, we've now validated the data being sent to us, and we are creating the
correct error if the data is not what we expect.  So what we are going to do now
is convert the password into something we can safely store into the database.
I am using the word "safely" very loosely.

To do this we're going to use the built in API from PHP 5.5 
[password_hash()](http://php.net/manual/en/function.password-hash.php)

Password hash accepts three parameters, the password, the algorithm to use,
and the options to use for the selected algorithm.  For this example we're
going to keep the default settings.  The default settings are deliberately set
low, so once you have your login system working, it's worth trying to increase
the default settings to secure your passwords even more.  Modifying these settings
is beyond the scope of this tutorial, but details on these options can be found 
[here](http://php.net/manual/en/function.password-hash.php)

`registration.php`

```php
...
    if ($error) {
        goto breaker;
    }

    $password = password_hash($password, PASSWORD_DEFAULT);
...
```

As of PHP 7.2 the following algorithms are currently supported: 
(Taken from [php.net](http://php.net/manual/en/function.password-hash.php))
              
  * PASSWORD_DEFAULT - Use the bcrypt algorithm (default as of PHP 5.5.0). Note that this constant is designed to change over time as new and stronger algorithms are added to PHP. For that reason, the length of the result from using this identifier can change over time. Therefore, it is recommended to store the result in a database column that can expand beyond 60 characters (255 characters would be a good choice).
  * PASSWORD_BCRYPT - Use the CRYPT_BLOWFISH algorithm to create the hash. This will produce a standard crypt() compatible hash using the "$2y$" identifier. The result will always be a 60 character string, or FALSE on failure.
  * PASSWORD_ARGON2I - Use the Argon2 hashing algorithm to create the hash.


As I stated above the current recommended algorithm is Argon2 but the default setting
is still using bcrypt.  If you're using PHP 7.2, there's no reason to wait for PHP to 
update it's default, you can just go ahead and update the password_hash to use Argon2
now. However, I would encourage you to go ahead and use default.   But if you change 
this be aware that you will have to maintain this setting on your own going forward, 
instead of letting PHP do it for you.

Ok so the password is now hashed, the only thing we have left to do is store the new
user in the database and redirect the user to the login page...  Ok in a real system
we'd log the user in and redirect to the restricted page.  But for this tutorial we're
going to make them login.

`registration.php`

```php
...
    $passwordHash = password_hash($password, PASSWORD_DEFAULT);
    
    /* Connect to the database */

    // SHORTCUT.  Generally the database connection should be abstracted out
    // into its own service so you can reuse the code over and over again in your app.
    // Better yet, instead of creating your own database abstraction, I'd highly
    // encourage you to use one that's already built and available through packagist.
    // Might I suggest Doctrine 2 or Zend DB?
    $connection = new mysqli('localhost', 'root', 'root', 'example');

    /* Handle connection errors */
    if ($connection->connect_error) {
        $error = true;
        
        // Generally you'd never output the actual database error to the user
        // but we'll do so here to make troubleshooting a little easier.
        // Don't do this in production
        $errorMsg = 'Unable to connect to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    /* Let's make sure this email doesn't already exist */
    $userCheckStatement = $connection->prepare('SELECT COUNT(1) FROM users WHERE `email` = ?');

    if (!$userCheckStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $userCheckStatement->bind_param('s', $email);
    $success = $userCheckStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Internal error communicating with the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $count = null;
    $userCheckStatement->bind_result($count);
    $userCheckStatement->fetch();

    if ($count > 0) {
        $error = true;
        
        // Remember we never tell the user if we have an account with that email address.
        $errorMsg = 'There was a problem creating your account.  Did you <a href="/forgot-password.php"> forget your password? </a>';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }
    
    $userCheckStatement->free_result();

    $statement = $connection->prepare(
        'INSERT INTO users (`email`, `password`) VALUES (?, ?)'
    );

    if (!$statement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $statement->bind_param('ss', $email, $passwordHash);

    $success = $statement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Unable to add new user to the database.';
    }

    if (!$error) {
        header('Location: login.php?success=1');
    }
...
```

Ok we now have a completed registration form.

`registration.php (final)`

```php
<?php
/* Setup needed variables and default state */
// Error placeholders
$error = false;
$errorMsg = '';

// Fields
$email = '';

// Validation
$emailInvalidOrMissing    = false;
$passwordInvalidOrMissing = false;


if (!empty($_POST)) {

    // Never trust data from the end users.  Always validate your inputs.

    /* Data Validation */
    $email = filter_var(
        $_POST['email'] ?? '',
        FILTER_VALIDATE_EMAIL
    );

    $password = filter_var(
        $_POST['password'] ?? '',
        FILTER_SANITIZE_STRING
    );

    $verify = filter_var(
        $_POST['verify'] ?? '',
        FILTER_SANITIZE_STRING
    );

    /* Make sure we have all the data we need */
    if (empty($email)) {
        $emailInvalidOrMissing = true;
    }

    if (empty($password)) {
        $passwordInvalidOrMissing = true;
    }

    if (empty($verify)) {
        $passwordInvalidOrMissing = true;
    }

    if ($emailInvalidOrMissing || $passwordInvalidOrMissing) {
        $error = true;
        $errorMsg = 'Unable to complete registration. Please make sure to complete the questions below and try again';
    }

    // SHORTCUT.  Generally we would have abstracted out our logic from html, making
    // errors and validations a lot easier to deal with.  For this tutorial we are
    // not creating the usual abstractions, so we're cheating here a little.  Please don't do this
    // in the real world.
    if ($error) {
        goto breaker;
    }

    $passwordHash = password_hash($password, PASSWORD_DEFAULT);

    /* Connect to the database */
    
    // SHORTCUT.  Generally the database connection should be abstracted out
    // into its own service so you can reuse the code over and over again in your app.
    // Better yet, instead of creating your own database abstraction, I'd highly
    // encourage you to use one that's already built and available through packagist.
    // Might I suggest Doctrine 2 or Zend DB?
    $connection = new mysqli('localhost', 'root', 'root', 'example');

    /* Handle connection errors */
    if ($connection->connect_error) {
        $error = true;
        
        // Generally you'd never output the actual database error to the user
        // but we'll do so here to make troubleshooting a little easier.
        // Don't do this in production.
        $errorMsg = 'Unable to connect to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    /* Let's make sure this email doesn't already exist */
    $userCheckStatement = $connection->prepare('SELECT COUNT(1) FROM users WHERE `email` = ?');

    if (!$userCheckStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $userCheckStatement->bind_param('s', $email);
    $success = $userCheckStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Internal error communicating with the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $count = null;
    $userCheckStatement->bind_result($count);
    $userCheckStatement->fetch();

    if ($count > 0) {
        $error = true;
        
        // Remember we never tell the user if we have an account with that email address.
        $errorMsg = 'There was a problem creating your account.  Did you <a href="/forgot-password.php"> forget your password? </a>';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }
    
    $userCheckStatement->free_result();

    $statement = $connection->prepare(
        'INSERT INTO users (`email`, `password`) VALUES (?, ?)'
    );

    if (!$statement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $statement->bind_param('ss', $email, $passwordHash);

    $success = $statement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Unable to add new user to the database.';
    }

    if (!$error) {
        header('Location: login.php?success=1');
    }
}

//  Cheating goto marker.  Please don't do this in the real world.
breaker:

?>

<html>
<head>
    <title>Registration</title>

    <style type="text/css">
        .error {
            font-weight: bold;
            color: #ff0000;
        }
    </style>
</head>
<body>
<h1>Registration</h1>

<?php if ($error): ?>
<div class="error"><?= $errorMsg; ?></div>
<?php endif; ?>

<form action="registration.php" method="post">
    <p>
        <label <?php if ($emailInvalidOrMissing): ?> class="error" <?php endif; ?>>
            Email Address:
            <input type="text" name="email" value="<?= $email ?>">
        </label>
    </p>

    <p>
        <label <?php if ($passwordInvalidOrMissing): ?> class="error" <?php endif; ?>>
            Password:
            <input type="password" name="password" value="">
        </label>
    </p>

    <p>
        <label <?php if ($passwordInvalidOrMissing): ?> class="error" <?php endif; ?>>
            Verify Password:
            <input type="password" name="verify" value="">
        </label>
    </p>

    <p>
        <input type="submit" name="submit" value="Login">
    </p>
</form>
</body>
</html>
```

# Login

Ok now that we have a user in the database let's work on logging the in.

To start let's wire up a default state and setup some placeholders to use, just
like we did above.

`login.php`
```php
<?php
/* Setup needed variables and default state */
// Error placeholders
$error = false;
$errorMsg = '';

// Fields
$email = '';
?>

<html>
<head>
    <title>Login</title>
    <style type="text/css">
        .error {
            font-weight: bold;
            color: #ff0000;
        }
    </style>
</head>
<body>
<h1>Login</h1>

<?php if ($error): ?>
    <div class="error"><?= $errorMsg; ?></div>
<?php endif; ?>

<form action="login.php" method="post">
    <p>
        <label>
            Email Address:
            <input type="text" name="email" value="<?= $email; ?>">
        </label>
    </p>

    <p>
        <label>
            Password:
            <input type="password" name="password" value="">
        </label>
    </p>

    <p>
        <input type="submit" name="submit" value="Login">
    </p>

    <p><a href="forgot-password.php" >Forgot Password?</a> </p>
    <p><a href="registration.php" >New User?</a> </p>
</form>
</body>
</html>
```

Let's now wire up the logic to process the form data and show an error if the data is incorrect.

`login.php`

```php
...
// Fields
$email = '';

if ($_POST) {
    // Never trust data from the end users.  Always validate your inputs.

    /* Data Validation */
    $email = filter_var(
        $_POST['email'] ?? '',
        FILTER_VALIDATE_EMAIL
    );

    $password = filter_var(
        $_POST['password'] ?? '',
        FILTER_SANITIZE_STRING
    );

    /* Make sure we have all the data we need */
    if (empty($email)
        || empty($password)
    ) {
        $error = true;
        $errorMsg = 'Unable to log you in.  Please check the form below and try again';

        // SHORTCUT.  Generally we would have abstracted out our logic from html, making
        // errors and validations a lot easier to deal with.  For this tutorial we are
        // not creating the usual abstractions, so we're cheating here a little.
        // Please don't do this in the real world.
        goto breaker;
    }
}

//  Cheating goto marker.  Please don't do this in the real world.
breaker:
...
```

Ok we now have the page wired up, now we need to get the hash for the user
the person claims they are.  This is a little different then what you
may have seen in the past, so it deserves some explanation.

Before the new password api was introduced, it was the general practice to hash the password
the user supplied in the form and then ask the database if this hash matches for the user.
With the new api the hash generated actually contains information on how it was created 
so it is needed in order to validate the password goes through the same process 
as the one stored.

While this may seem strange, and I admit is hard to really explain without getting to far
into the weeds, it does make sure that even if we change the hashing defaults or change the
algorithm we can still log the user in even if that hash uses an old out dated hash.  

Don't worry, one thing this does is allow us to find out if the hash itself is outdated
and needs to be updated.  So this step will actually help us down the road.

Let's fetch that hash now.

`login.php`
```php
...
    /* Make sure we have all the data we need */
    if (empty($email)
        || empty($password)
    ) {
        $error = true;
        $errorMsg = 'Unable to log you in.  Please check the form below and try again';

        // SHORTCUT.  Generally we would have abstracted out our logic from html, making
        // errors and validations a lot easier to deal with.  For this tutorial we are
        // not creating the usual abstractions, so we're cheating here a little.
        // Please don't do this in the real world.
        goto breaker;
    }

    /* Connect to the database */
    
    // SHORTCUT.  Generally the database connection should be abstracted out
    // into its own service so you can reuse the code over and over again in your app.
    // Better yet, instead of creating your own database abstraction, I'd highly
    // encourage you to use one that's already built and available through packagist.
    // Might I suggest Doctrine 2 or Zend DB?
    $connection = new mysqli('localhost', 'root', 'root', 'example');

    /* Handle connection errors */
    if ($connection->connect_error) {
        $error = true;

        // Generally you'd never output the actual database error to the user
        // but we'll do so here to make troubleshooting a little easier.
        // Don't do this in production.
        $errorMsg = 'Unable to connect to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    /* Let's fetch the hash if one exists for the user */
    $userHashStatement = $connection->prepare('SELECT `password` FROM users WHERE `email` = ?');

    if (!$userHashStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $userHashStatement->bind_param('s', $email);
    $success = $userHashStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Internal error communicating with the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $hash = null;
    $userHashStatement->bind_result($hash);
    $userHashStatement->fetch();

    //  No hash found
    if (empty($hash)) {
        $error = true;

        // Remember we don't tell the user if the account exists or not
        $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }
...
```

Ok we have the hash from the database, let's validate it.

`login.php`
```php
...
    //  No hash found
    if (empty($hash)) {
        $error = true;

        // Remember we don't tell the user if the account exists or not
        $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    // Validate the password against the hash.  Redirect on success
    if (password_verify($password, $hash)) {
        // Store that user is now logged in.
        $_SESSION['signedIn'] = true;
        
        // Redirect the user to the restricted page
        header('Location: /');
         
        // Send header and exit script 
        return;
    }
    
    $error = true;
    
    // Password invalid.  Remember we don't tell the user if the account exists or not so
    // let's reuse our generic error.
    $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';
...
```

Ok at this point the user is logged in.  But how do we keep this stored so the user
doesn't have to keep logging in?

There's a couple of ways to do this, but in this tutorial we're going to use sessions to
persist this info.   So we'll need to start the session at the top of the page.

`login.php`
```php
<?php

session_start();

...
```

In this tutorial, we'll simply store if the user is logged in.  In the real world, you'd 
probably also store the user they are logged in as and any privileges that user has.  That
is beyond the scope of this tutorial, so let's just set a variable to let us know
the user is already signed in and redirect them to the restricted page.

`login.php`
```php
...
    // Validate the password against the hash.  Redirect on success
    if (password_verify($password, $hash)) {
        // Store that user is now logged in.
        $_SESSION['signedIn'] = true;
        
        // Redirect the user to the restricted page
        header('Location: /');
         
        // Send header and exit script 
        return;
    }

    $error = true;

    // Password invalid.  Remember we don't tell the user if the account exists or not so
    // let's reuse our generic error.
    $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';
...
```

Ok we can now log users in.  That's great, but we still have one major hole we need
to address.

If we left the login form in it's current state, it would work and users would
be able to login and navigate the site.  However, we have nothing here
to prevent a possible attacker from simply brute forcing a users
password.  Brute forcing a users password means that an attacker will sit on the
login page and just keep trying different passwords until they find one that works.

There are many things you can do to help mitigate this problem.  For this tutorial
we're simply going to lock a users account if they fail to login 3 times.

In order for this to work, we need to keep track of login attempts in the database.
Because of the way sessions work, simply storing the login attempts via the session
var simply won't work because it relies on the browser to provide the previous session
number.  Since this is not very reliable, we'll just keep this in the db and when it
hits three attempts, we'll lock the account.

To begin, let's modify our login logic to make sure an account isn't locked
before logging them in.

`login.php`
```php
...
    /* Handle connection errors */
    if ($connection->connect_error) {
        $error = true;

        // Generally you'd never output the actual database error to the user
        // but we'll do so here to make troubleshooting a little easier.
        // Don't do this in production.
        $errorMsg = 'Unable to connect to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    // Let's fetch the hash if one exists for the user,
    // if the account is locked or not, and how many times they
    // have unsuccessfully attempted to login
    $userHashStatement = $connection->prepare('
        SELECT 
            `password`, 
            `locked`,
             `attempts`
        FROM users 
        WHERE `email` = ?
    ');

    if (!$userHashStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $userHashStatement->bind_param('s', $email);
    $success = $userHashStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Internal error communicating with the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $hash = null;
    $locked = false;
    $attempts = 0;

    $userHashStatement->bind_result($hash, $locked, $attempts);
    $userHashStatement->fetch();
    $userHashStatement->free_result();

    //  No hash found
    if (empty($hash)) {
        $error = true;

        // Remember we don't tell the user if the account exists or not
        $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    // Make sure account is not locked
    if ($locked) {
        $error = true;

        // Generally we don't want to give clues if an account exists or not.
        // Unfortunately accounts in a locked state need to reset their passwords
        // to get back into the system.   So let's be nice and let the user know
        // to reset their account.  If you don't you will flood your customer
        // service department with unneeded calls and email.
        $errorMsg = 'This account has been locked.  Please <a href="/forgot-password.php"> reset your password </a> in order to login';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }
...
```

Now we need to update the database on every failed login attempt. Or reset
the login attempts back to 0 on a successful login.

`login.php`

```php
...
    // Validate the password against the hash.  Redirect on success
    if (password_verify($password, $hash)) {
        // Reset login attempts
        $updateAttemptsStatement = $connection->prepare('
            UPDATE `users` 
            SET `attempts` = ?
            WHERE `email` = ?
        ');

        if (!$updateAttemptsStatement) {
            $error = true;
            $errorMsg = 'Unable to prepare lock statement to the database';

            //  Cheating goto.  Please don't do this in the real world.
            goto breaker;
        }
        
        $attempts = 0;
        $updateAttemptsStatement->bind_param('is', $attempts, $email);
        $success = $updateAttemptsStatement->execute();

        if (!$success) {
            $error = true;
            $errorMsg = 'Unable to update attempts to the database';

            //  Cheating goto.  Please don't do this in the real world.
            goto breaker;
        }

        $updateAttemptsStatement->free_result();
        
        // Store that user is now logged in.
        $_SESSION['signedIn'] = true;

        // Redirect the user to the restricted page
        header('Location: /');

        // Send header and exit script
        return;
    }

    $error = true;

    // Password invalid.  Remember we don't tell the user if the account exists or not so
    // let's reuse our generic error.
    $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';


    // Lock the account if they have tried over 3 times to login and failed
    $attempts++;

    if ($attempts >= 3) {
        $locked = true;
    }

    // Update the database with the locked and amount of login attempts
    $updateLockAndAttemptsStatement = $connection->prepare('
        UPDATE `users` 
        SET `locked` = ?, `attempts` = ?
        WHERE `email` = ?
    ');

    if (!$updateLockAndAttemptsStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare lock statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $updateLockAndAttemptsStatement->bind_param('iis', $locked, $attempts, $email);
    $success = $updateLockAndAttemptsStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Unable to update lock and attempts to the database';
    }
...
```

Great, we have now added some security to protect our users from 
brute force attacks.

Just when you thought we were done, there's still one more thing
we need to change on this login form.   Remember earlier we said
that we'd make our login system update hashes when they became
outdated and needed updated?  In order to do that we're going
to need to handle this during the users login.

The reason we update the hash on login is because it's only
time that we have the raw password for an existing user since hashes 
are one directional.

So let's make that happen using PHP's built in `password_needs_rehash`

`login.php`

```php
...
    $updateAttemptsStatement->free_result();

    //Check to see if the password needs to be rehashed
    if (password_needs_rehash($hash, PASSWORD_DEFAULT)) {
    
        // Generate a new hash using the password supplied.
        $newHash = password_hash($password, PASSWORD_DEFAULT);
    
        // Save the new hash to the database
        $updateStatement = $connection->prepare(
            'UPDATE users set `password` = ? WHERE `email` = ?'
        );
    
        if (!$updateStatement) {
            $error = true;
            $errorMsg = 'Unable to prepare statement to the database: '.$connection->error;
    
            //  Cheating goto.  Please don't do this in the real world.
            goto breaker;
        }
    
        $updateStatement->bind_param('ss', $newHash, $email);
    
        $success = $updateStatement->execute();
    
        if (!$success) {
            $error = true;
            $errorMsg = 'Unable to update user in the database: '.$connection->error;
    
            //  Cheating goto.  Please don't do this in the real world.
            goto breaker;
        }
    
        $updateStatement->free_result();
    }
...
```

As you can see, there is a lot to logging them into a system.  It's not as
clean cut as looking at the password and validating they typed in the data
correctly.  There's also a lot more you can do here.  We just did the bare 
minimum to get a relatively secure login form working.  

Other steps we might take:

  * Slow down invalid login returns with say a `sleep(3)` to help slow
    down brute force attacks.
    
  * Restrict the same IP to only 3 failed attempts every 15 minutes or so
  
  * Require passwords to be a set length, or only contain a certain set of
    characters.  <- This is actually a HORRIBLE idea, but I have seen some
    sites do this.  Please don't, as it actually makes your site less
    secure.
    
These are just some examples we can try to secure this page even more.  Since
nothing is ever 100% secure or foolproof, if you follow this guide, it's my 
opinion that it'll be secure enough.

Here's the final login page.

`login.php`
```php
<?php

session_start();

/* Setup needed variables and default state */
// Error placeholders
$error = false;
$errorMsg = '';

// Fields
$email = '';

if ($_POST) {
    // Never trust data from the end users.  Always validate your inputs.

    /* Data Validation */
    $email = filter_var(
        $_POST['email'] ?? '',
        FILTER_VALIDATE_EMAIL
    );

    $password = filter_var(
        $_POST['password'] ?? '',
        FILTER_SANITIZE_STRING
    );

    /* Make sure we have all the data we need */
    if (empty($email)
        || empty($password)
    ) {
        $error = true;
        $errorMsg = 'Unable to log you in.  Please check the form below and try again';

        // SHORTCUT.  Generally we would have abstracted out our logic from html, making
        // errors and validations a lot easier to deal with.  For this tutorial we are
        // not creating the usual abstractions, so we're cheating here a little.
        // Please don't do this in the real world.
        goto breaker;
    }

    /* Connect to the database */

    // SHORTCUT.  Generally the database connection should be abstracted out
    // into its own service so you can reuse the code over and over again in your app.
    // Better yet, instead of creating your own database abstraction, I'd highly
    // encourage you to use one that's already built and available through packagist.
    // Might I suggest Doctrine 2 or Zend DB?
    $connection = new mysqli('localhost', 'root', 'root', 'example');

    /* Handle connection errors */
    if ($connection->connect_error) {
        $error = true;

        // Generally you'd never output the actual database error to the user
        // but we'll do so here to make troubleshooting a little easier.
        // Don't do this in production.
        $errorMsg = 'Unable to connect to the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    // Let's fetch the hash if one exists for the user,
    // if the account is locked or not, and how many times they
    // have unsuccessfully attempted to login
    $userHashStatement = $connection->prepare('
        SELECT 
            `password`, 
            `locked`,
             `attempts`
        FROM users 
        WHERE `email` = ?
    ');

    if (!$userHashStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $userHashStatement->bind_param('s', $email);
    $success = $userHashStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Internal error communicating with the database: '.$connection->error;

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $hash = null;
    $locked = false;
    $attempts = 0;

    $userHashStatement->bind_result($hash, $locked, $attempts);
    $userHashStatement->fetch();
    $userHashStatement->free_result();

    //  No hash found
    if (empty($hash)) {
        $error = true;

        // Remember we don't tell the user if the account exists or not
        $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    // Make sure account is not locked
    if ($locked) {
        $error = true;

        // Generally we don't want to give clues if an account exists or not.
        // Unfortunately accounts in a locked state need to reset their passwords
        // to get back into the system.   So let's be nice and let the user know
        // to reset their account.
        $errorMsg = 'This account has been locked.  Please <a href="/forgot-password.php"> reset your password </a> in order to login';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    // Validate the password against the hash.  Redirect on success
    if (password_verify($password, $hash)) {

        // Reset login attempts
        $updateAttemptsStatement = $connection->prepare('
            UPDATE `users` 
            SET `attempts` = ?
            WHERE `email` = ?
        ');

        if (!$updateAttemptsStatement) {
            $error = true;
            $errorMsg = 'Unable to prepare lock statement to the database';

            //  Cheating goto.  Please don't do this in the real world.
            goto breaker;
        }

        $attempts = 0;
        $updateAttemptsStatement->bind_param('is', $attempts, $email);
        $success = $updateAttemptsStatement->execute();

        if (!$success) {
            $error = true;
            $errorMsg = 'Unable to update attempts to the database';

            //  Cheating goto.  Please don't do this in the real world.
            goto breaker;
        }

        $updateAttemptsStatement->free_result();

        //Check to see if the password needs to be rehashed
        if (password_needs_rehash($hash, PASSWORD_DEFAULT)) {
            
            // Generate a new hash using the password supplied.
            $newHash = password_hash($password, PASSWORD_DEFAULT);

            // Save the new hash to the database
            $updateStatement = $connection->prepare(
                'UPDATE users set `password` = ? WHERE `email` = ?'
            );

            if (!$updateStatement) {
                $error = true;
                $errorMsg = 'Unable to prepare statement to the database: '.$connection->error;

                //  Cheating goto.  Please don't do this in the real world.
                goto breaker;
            }

            $updateStatement->bind_param('ss', $newHash, $email);

            $success = $updateStatement->execute();

            if (!$success) {
                $error = true;
                $errorMsg = 'Unable to update user in the database: '.$connection->error;

                //  Cheating goto.  Please don't do this in the real world.
                goto breaker;
            }

            $updateStatement->free_result();
        }

        // Store that user is now logged in.
        $_SESSION['signedIn'] = true;

        // Redirect the user to the restricted page
        header('Location: /');

        // Send header and exit script
        return;
    }

    $error = true;

    // Password invalid.  Remember we don't tell the user if the account exists or not so
    // let's reuse our generic error.
    $errorMsg = 'There was a problem logging you in. Did you <a href="/forgot-password.php"> forget your password? </a>';


    // Lock the account if they have tried over 3 times to login and failed
    $attempts++;

    if ($attempts >= 3) {
        $locked = true;
    }

    // Update the database with the locked and amount of login attempts
    $updateLockAndAttemptsStatement = $connection->prepare('
        UPDATE `users` 
        SET `locked` = ?, `attempts` = ?
        WHERE `email` = ?
    ');

    if (!$updateLockAndAttemptsStatement) {
        $error = true;
        $errorMsg = 'Unable to prepare lock statement to the database';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $updateLockAndAttemptsStatement->bind_param('iis', $locked, $attempts, $email);
    $success = $updateLockAndAttemptsStatement->execute();

    if (!$success) {
        $error = true;
        $errorMsg = 'Unable to update lock and attempts to the database';
    }
}

//  Cheating goto marker.  Please don't do this in the real world.
breaker:
?>

<html>
<head>
    <title>Login</title>
    <style type="text/css">
        .error {
            font-weight: bold;
            color: #ff0000;
        }
    </style>
</head>
<body>
<h1>Login</h1>

<?php if ($error): ?>
    <div class="error"><?= $errorMsg; ?></div>
<?php endif; ?>

<form action="login.php" method="post">
    <p>
        <label>
            Email Address:
            <input type="text" name="email" value="<?= $email; ?>">
        </label>
    </p>

    <p>
        <label>
            Password:
            <input type="password" name="password" value="">
        </label>
    </p>

    <p>
        <input type="submit" name="submit" value="Login">
    </p>

    <p><a href="forgot-password.php" >Forgot Password?</a> </p>
    <p><a href="registration.php" >New User?</a> </p>
</form>
</body>
</html>
```

# Log out
Now that we can login, we need a way to logout.   This is actually REALLY easy.
All we really need to do is clear out the users session like so:

`logout.php`

```php
<?php
// Fetch the users current session
session_start();

// Destroy the users session
session_destroy();
?>
```

All done.  Nothing else to do here.

`logout.php (final)`
```php
<?php
// Fetch the users current session
session_start();

// Destroy the users session
session_destroy();
?>

<html>
<head>
    <title>Logout</title>
</head>
<body>
    <p>You are logged out.</p>
    <p><button onclick='window.location="login.php"'>Login</button></p>
</body>
</html>
```

# Adding restrictions to pages

Now that we can sign up and login, it's time to actually restrict our
index page from users who haven't logged in.

To that we add the following code:

`index.php`

```php
<?php
// Start the users session or begin a new one
session_start();

// Validate that the user is logged in.
// We use empty here, because this might
// be a new session and $_SESSION['signedIn'] might not be set yet.
if (empty($_SESSION['signedIn'])) {
    header('Location: /login.php');

    // Exit the script to make sure our protected page never gets
    // accidentally sent
    return;
}
?>

...
```

What we've done here is looked for our login session var of `$_SESSION['signedIn']`
if it's empty, null, not set, or false we're going to redirect that user to 
the login page.  Otherwise, we'll display the page normally.

`index.php (final)`
```php
<?php
// Start the users session or begin a new one
session_start();

// Validate that the user is logged in.
// We use empty here, because this might
// be a new session and $_SESSION['signedIn'] might not be set yet.
if (empty($_SESSION['signedIn'])) {
    header('Location: /login.php');

    // Exit the script to make sure our protected page never gets
    // accidentally sent
    return;
}
?>

<html>
<head>
    <title>Restricted Page</title>
</head>
<body>
    <h1>Restricted</h1>
    <p>Only logged in users should see this page.</p>
    <p><button onclick='window.location="logout.php"'>Logout</button></p>
</body>
</html>
```
