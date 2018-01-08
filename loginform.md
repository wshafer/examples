# Introduction


### Storing and managing passwords
Before we begin this tutorial we need to have a serious talk about how
to handle and store passwords.

Passwords need to be treated with care.  Remember these passwords will
grant users access to your system, sometime that access contains
even more incredibly sensitive data.  For this reason you should follow
these two very simple rules.

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


### Hashing
Ok so now that we know what not do with passwords let's talk about hashing.
Hashing passwords is by far the most misunderstood thing in this whole process. I
can't tell you how many times I see questions about this on forums, facebook pages,
slack, etc.  Not only are people confused, the answers I see other developers 
give are absolutely horrifying.

First thing to know, is that hashing **IS NOT** encryption.  Encryption is meant
to someday be unencrypted. So while storing an encrypted password is a little safer 
then storing the password outright, it's not much safer.  Since all it would take is 
to crack one password and your encryption would compromised.  Once encryption is 
compromised it's no better then if you have the passwords stored in the clear since 
all the passwords would be encrypted the same way.  On the other hand, 
**hashing** is a generated string that represents the password.  A hash can only go 
one-way, so you can get a hash from a password, but you'll never get the password
from the hash.  If one password is hacked (brute forced), the hacker would only have
one password, not all of them.

Hashing has it's own caveats that you should be aware of however.  While you can't get
the exact password out of a hash easily, you can get a combination of letters that can 
also generate the same hash.  These are called "collision vulnerabilities" and some
hashing algorithms are more vulnerable to this then others.

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
CREATE TABLE users (
  id MEDIUMINT NOT NULL AUTO_INCREMENT, 
  email VARCHAR(254) NOT NULL UNIQUE, 
  password VARCHAR(255) NOT NULL,
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

Before we can add any kind of authentication or a user can login we need a way for users
to register with the site.  So let's begin building out the registration screen.

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
as expected now.  Go ahead, give it a try.

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
now. However, I would encourage you to go ahead and use default beacuse if you change 
this you will have to maintain this setting on your own, instead of letting PHP do it 
for you.

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
        $errorMsg = 'Unable to connect to the database';

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
        $errorMsg = 'Unable to add new user to the database.';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $count = null;
    $userCheckStatement->bind_result($count);
    $userCheckStatement->fetch();

    if ($count > 0) {
        $error = true;
        $errorMsg = 'There is already an account with that email address.  Did you <a href="/forgot-password.php"> forget your password? </a>';

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
        $errorMsg = 'Unable to connect to the database';

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
        $errorMsg = 'Unable to add new user to the database.';

        //  Cheating goto.  Please don't do this in the real world.
        goto breaker;
    }

    $count = null;
    $userCheckStatement->bind_result($count);
    $userCheckStatement->fetch();

    if ($count > 0) {
        $error = true;
        $errorMsg = 'There is already an account with that email address.  Did you <a href="/forgot-password.php"> forget your password? </a>';

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