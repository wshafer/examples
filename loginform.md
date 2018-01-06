```mysql
CREATE TABLE users (
  id MEDIUMINT NOT NULL AUTO_INCREMENT, 
  email VARCHAR(254) NOT NULL, 
  password VARCHAR(255) NOT NULL,
  PRIMARY KEY (id)
);
```
<br />

`index.php`
```php
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
```php
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
        <input type="submit" name="submit" value="Login">
    </p>
</form>
</body>
</html>
```

<br />

`login.php`
```php
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
```php
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
```php
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
```php
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
            <input type="password" name="validate" value="">
        </label>
    </p>
    <p>
        <input type="submit" name="submit" value="Reset Password">
    </p>
</form>
</body>
</html>
```
