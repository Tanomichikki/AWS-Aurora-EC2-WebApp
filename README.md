# 🚀 Connecting a Web App on EC2 to Amazon Aurora MySQL

In a real-world scenarios company, employees interact with applications daily—HR portals, attendance tracking, payroll systems, task dashboards, etc. These applications need a secure place to run.

In this project, I build a real-world employee management web application.
Employees working in a company should be able to:
- Submit their information
- View/update basic details
- Fetch employee records
- Perform operations through a web interface

For this, your web application needs a server environment to run — and that’s exactly why an EC2 instance is required.
As the simulated company grows from 20 → 200 → 2,000 employees…
Aurora automatically scales the storage and performance

### 🧭 Project Overview

This project includes:
- Creating an EC2 instance for hosting the web application
- Launching an Amazon Aurora MySQL cluster
- Connecting EC2 ↔ Aurora inside the same VPC
- Handling SSH, permission errors, and folder ownership issues
- Deploying a PHP + MySQLi demo application

**🏗️ AWS Architecture Diagram**
<p align="center">
  <img src="https://github.com/Tanomichikki/Aurora-EC2-WebApp-AWS/blob/main/architecture-diagram-AWS-Aurora-connecting-Webapp.png" width="50%" />
</p>


### 1️. Launch EC2 Instance

- **EC2 instance name:**  employee-aurora-ec2-instance
- **AMI Type:**  Amazon Linux 2023 kernel-6.1 AMI
- **Instance Type:** t2.micro, t3.micro (if included in free tier)

**(a). Create Key Pair (RSA, .pem)**
Why?

A key pair enables secure SSH access using public–private cryptography, eliminating passwords. Only someone with the private key can log in,
keeping your EC2 instance protected. This is AWS’s standard and safest authentication method.

**(b). Create Security Group**
Allow:

SSH(22) is restricted to your IP for security, so only you can admin the server. HTTP(80) must be open to everyone so users can access the web app. This follows least privilege while keeping the app public.

**(c). Save Your EC2 Public DNS**
You need the public DNS/IP to SSH into the instance and to open the website from a browser. It’s the address clients use to reach your server. Keeping it handy avoids mistakes during connection.


### 2️. Aurora MySQL Cluster
**Why Aurora?**
Aurora provides a managed, high-performance, fault-tolerant relational database. It auto-scales storage, performs faster than MySQL, and handles backups/replication automatically. This reduces admin work and increases reliability.
- Highly scalable
- Auto-failover + high availability
- Faster than traditional MySQL
- Production-ready engine

**Same Region Requirement**
EC2 and Aurora must be in same AWS region.

Why must EC2 and Aurora be in the same region?

- Same-region placement ensures low-latency communication over the VPC’s private network. Cross-region traffic is slower, costlier, and adds complexity. Keeping them together ensures smooth DB connectivity.

**Aurora Setup Steps**

- **Engine:** Aurora MySQL 3.08.2
- **Template: Dev/Test :** The Dev/Test template is cost-optimized and ideal for learning projects. Choosing the right MySQL-compatible version ensures PHP and MySQL tools work without compatibility issues. It keeps the setup simple and reliable.
- **DB Cluster Idetifier:** employee
- **Master username:** admin
- **Credential management:** Self managed
- **Master Password:** companydatabase
- **Instance filter:** Burstable classes
- **Availability:** Don't create an aurora replica
- **Connectivity:** Enable Connect to an EC2 compute resource : Chose your newly created instance (This ensures your EC2 instance can reach Aurora over private networking inside the VPC. It prevents exposing your database publicly. It’s the secure and recommended architecture.)


### 3️. SSH Into EC2 — Fixing Access Denied

Try SSH:
```bash
ssh -i EmployeeAuroraApp.pem ec2-user@YOUR_EC2_ADDRESS
```

❌ **Error: Permission denied (publickey)**

**Why this happens?**
- .pem file permissions are insecure.
- SSH refuses to use private keys with weak file permissions.
- **chmod 400** locks the file so only you can read it. This satisfies SSH security requirements and allows login.

Fix it:
```bash
chmod 400 EmployeeAuroraApp.pem
```
**Why run chmod 400 EmployeeAuroraApp.pem?**
- It protects the private key by restricting access to only your user. SSH demands these strict permissions for safety. Without it, authentication will fail.
SSH again:
```
ssh -i EmployeeAuroraApp.pem ec2-user@YOUR_EC2_ADDRESS
```
### 4️. Install Web Server & PHP
Updating ensures you have the latest security patches and package versions. It prevents installation errors and compatibility issues. This creates a stable environment before installing software.
Update packages:
```bash
sudo dnf update -y
```

Install Apache, PHP & MySQL drivers:
Apache hosts the web app, PHP runs server-side logic, and php-mysqli enables MySQL/Aurora connections. MariaDB client tools help test DB connectivity. Together they form a full web stack.
```bash
sudo dnf install -y httpd php php-mysqli mariadb105
```
Why?
- httpd → Apache web server
- php → Run PHP app
- php-mysqli → Connect PHP to Aurora MySQL
- mariadb105 → MySQL client tools

Start Apache:

Starting the service launches the web server so the site becomes accessible. Without this, the browser can’t load your PHP files. It activates your backend.
```bash
sudo systemctl start httpd
```
### 5️. Handle Permission Denied in /var/www

Go to the web root:
```bash
cd /var/www
```
```bash
mkdir inc
```

❌ **Error: mkdir: Permission denied**

Why?

The /var/www directory is owned by root, so regular users cannot create folders inside it. This protects system web directories. Changing ownership fixes this.


Fix ownership by change ownership of /var/www to ec2-user:
Giving ec2-user ownership allows you to edit and upload files without root access. It prevents permission errors during development. This is standard practice on EC2 web servers.
```bash
cd ../../
```
```bash
sudo chown ec2-user:ec2-user /var/www
```
💡 This changes the owner of the folder to the ec2-user....us!
👨‍🚒 chown means "change the owner permissions".
👨‍👩‍👧‍👦 ec2-user:ec2-user means we change the individual and group user from root to ec2-user.
Now try again — it works.
Now, let's see the folder again
```bash
ls -ld var/www
```


### 6️. Add Database Configuration

- Go to the config folder:
```bash
cd /var/www
```
- Create a new sub-folder named inc.
```bash
mkdir inc
```
- Now go to your inc folder
```bash
cd inc
```
- Create a new file in the inc directory named dbinfo.inc
```bash
>dbinfo.inc
```
- Now we have a blank file. We can edit our new file by calling nano
```bash
nano dbinfo.inc
```

Add:
```php
<?php
define('DB_SERVER', 'YOUR_WRITER_ENDPOINT');
define('DB_USERNAME', 'admin');
define('DB_PASSWORD', 'companydatabase');
define('DB_DATABASE', 'employee');
?>
```
**Why Writer Endpoint?**
- Reader endpoints are read-only and will reject write operations. The writer endpoint is the only one that accepts modifications. Using the wrong one breaks your insert logic.
- The writer endpoint is required for all INSERT and UPDATE operations. Keeping DB credentials in a config file makes management easier. It centralizes all connection details.

### 7️. Create the Web App

Navigate to:
```bash
cd /var/www/html
```

Create the file:
```bash
nano EmployeePage.php
```
Permission issues prevent file creation or edits. Adjusting ownership restores write access for ec2-user. It resolves common deployment errors.
Fix the issue:
```bash
sudo chown ec2-user:ec2-user /var/www
```

Paste full PHP code:
```php

<?php include "../inc/dbinfo.inc"; ?>
<html>
<body>
<h1>Employee Directory</h1>
<?php

  /* Connect to MySQL and select the database. */
  $connection = mysqli_connect(DB_SERVER, DB_USERNAME, DB_PASSWORD);

  if (mysqli_connect_errno()) echo "Failed to connect to MySQL: " . mysqli_connect_error();

  $database = mysqli_select_db($connection, DB_DATABASE);

  /* Ensure that the EMPLOYEES table exists. */
  VerifyEmployeesTable($connection, DB_DATABASE);

  /* If input fields are populated, add a row to the EMPLOYEES table. */
  $employee_name = htmlentities($_POST['NAME']);
  $employee_address = htmlentities($_POST['ADDRESS']);

  if (strlen($employee_name) || strlen($employee_address)) {
    AddEmployee($connection, $employee_name, $employee_address);
  }
?>

<!-- Input form -->
<form action="<?PHP echo $_SERVER['SCRIPT_NAME'] ?>" method="POST">
  <table border="0">
    <tr>
      <td>NAME</td>
      <td>ADDRESS</td>
    </tr>
    <tr>
      <td>
        <input type="text" name="NAME" maxlength="45" size="30" />
      </td>
      <td>
        <input type="text" name="ADDRESS" maxlength="90" size="60" />
      </td>
      <td>
        <input type="submit" value="Add Data" />
      </td>
    </tr>
  </table>
</form>

<!-- Display table data. -->
<table border="1" cellpadding="2" cellspacing="2">
  <tr>
    <td>ID</td>
    <td>NAME</td>
    <td>ADDRESS</td>
  </tr>

<?php

$result = mysqli_query($connection, "SELECT * FROM EMPLOYEES");

while($query_data = mysqli_fetch_row($result)) {
  echo "<tr>";
  echo "<td>",$query_data[0], "</td>",
       "<td>",$query_data[1], "</td>",
       "<td>",$query_data[2], "</td>";
  echo "</tr>";
}
?>

</table>

<!-- Clean up. -->
<?php

  mysqli_free_result($result);
  mysqli_close($connection);

?>

</body>
</html>


<?php

/* Add an employee to the table. */
function AddEmployee($connection, $name, $address) {
   $n = mysqli_real_escape_string($connection, $name);
   $a = mysqli_real_escape_string($connection, $address);

   $query = "INSERT INTO EMPLOYEES (NAME, ADDRESS) VALUES ('$n', '$a');";

   if(!mysqli_query($connection, $query)) echo("<p>Error adding employee data.</p>");
}

/* Check whether the table exists and, if not, create it. */
function VerifyEmployeesTable($connection, $dbName) {
  if(!TableExists("EMPLOYEES", $connection, $dbName))
  {
     $query = "CREATE TABLE EMPLOYEES (
         ID int(11) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
         NAME VARCHAR(45),
         ADDRESS VARCHAR(90)
       )";

     if(!mysqli_query($connection, $query)) echo("<p>Error creating table.</p>");
  }
}

/* Check for the existence of a table. */
function TableExists($tableName, $connection, $dbName) {
  $t = mysqli_real_escape_string($connection, $tableName);
  $d = mysqli_real_escape_string($connection, $dbName);

  $checktable = mysqli_query($connection,
      "SELECT TABLE_NAME FROM information_schema.TABLES WHERE TABLE_NAME = '$t' AND TABLE_SCHEMA = '$d'");

  if(mysqli_num_rows($checktable) > 0) return true;

  return false;
}
?>                        

```

### 8️. Fix HTML Folder Permission Denied

If you cannot save the file:
```bash
ls -ld
```
```bash
sudo chown ec2-user:ec2-user 
```

Try saving again.

### 9️. Test the Application

Open:
```cpp
http://YOUR_EC2_ADDRESS/EmployeePage.php
```

You should see:

✔️ Input form

✔️ Employee table

✔️ Aurora DB connected successfully


### 10. Connect to your Database using MySQL CLI
After setting up your web application and connecting EC2 → Aurora, it's important to confirm whether the data submitted through your webapp is actually stored in the database.
After setting up your web application and connecting EC2 → Aurora, it's important to confirm whether the data submitted through your webapp is actually stored in the database.
- **MySQL CLI** is a powerful, secure, and efficient tool for interacting with MySQL databases. It provides direct access to all MySQL features, supports automation, and is a preferred method for managing databases on remote servers, such as EC2 instances.

**Download MySQL Rpository in your EC2 instance**
```bash
sudo yum install https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm -y
```

**Install MySQL**
```bash
sudo yum install mysql-community-client -y
```

**Connect to your Aurora MySQL Database:**
```bash
mysql -h YOUR_WRITER_ENDPOINT -P 3306 -u admin -p
```
Make sure you replace:
- **YOUR_ENDPOINT** with the Endpoint from your Aurora Writer instance
- **YOUR_PORT** with the Port from your Aurora Writer instance; 3306
- **YOUR_AURORA_USERNAME** with your Aurora username; admin
Enter your Aurora Password: **companydatabase**

**Show All Databases**
```bash
SHOW DATABASES;
```
**Select your Database**
```bash
USE sample;
```
**List All Tables**
```bash
SHOW TABLES;
```
**Check Table Structure**
```bash
DESCRIBE EMPLOYEES;
```
**View Stored Employees Data**
```bash
SELECT * FROM EMPLOYEES;
```

### 🎯 Final Achievements
- EC2 web server successfully running Apache + PHP
- Aurora MySQL writer endpoint connected
- Private networking between EC2 & DB
- Solved SSH errors
- Solved filesystem permission issues
- Working dynamic DB-driven PHP application
- Validated Everything through MySQL CLI
