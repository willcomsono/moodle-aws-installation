# Moodle Installation on Amazon EC2 - Complete Guide

## Overview
This guide provides a complete step-by-step process to install Moodle 4.4.11 on Amazon EC2 running Amazon Linux 2023. This documentation covers everything from EC2 instance creation to a fully functional Moodle Learning Management System.

## Prerequisites
- AWS Account with EC2 access
- Basic knowledge of AWS console
- SSH client (Terminal, PuTTY, etc.)

---

## Phase 1: AWS EC2 Setup

### Step 1: Launch EC2 Instance
1. **Login to AWS Console** → Navigate to EC2
2. **Click "Launch Instance"**
3. **Choose AMI:** Amazon Linux 2023 (64-bit x86)
4. **Instance Type:** t2.medium (recommended for production)
5. **Key Pair:** Create new or select existing
6. **Network Settings:**
   - VPC: Use default VPC or create new
   - Subnet: Default or specific subnet
   - Auto-assign public IP: Enable
7. **Storage:** 20GB minimum (gp3 recommended)
8. **Security Group:** Create new with these rules:
   - SSH (22) - Your IP
   - HTTP (80) - 0.0.0.0/0
   - HTTPS (443) - 0.0.0.0/0
9. **Launch Instance**

### Step 2: Connect to Instance
```bash
# Replace with your key file and public IP
ssh -i "your-key.pem" ec2-user@YOUR_PUBLIC_IP
```

---

## Phase 2: System Preparation

### Step 1: Update System
```bash
sudo dnf update -y
```

### Step 2: Install Essential Packages
```bash
# Install Apache, PHP, Git, and other essentials
sudo dnf install -y httpd php php-cli php-fpm php-mysqlnd php-xml php-soap php-intl php-zip php-gd php-mbstring php-opcache php-pdo git memcached

# Start and enable services
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl start memcached
sudo systemctl enable memcached
```

### Step 3: Install MySQL 8.0
```bash
# Download MySQL repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

# Install MySQL repository
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y

# Import GPG key
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

# Install MySQL
sudo dnf install mysql-community-server -y

# Start and enable MySQL
sudo systemctl start mysqld
sudo systemctl enable mysqld
```

### Step 4: Secure MySQL Installation
```bash
# Get temporary password
sudo grep 'temporary password' /var/log/mysqld.log

# Run security script
sudo mysql_secure_installation
# Use strong password
```

### Step 5: Create Moodle Database
```bash
# Login to MySQL
sudo mysql -u root -p

# Create database and user
CREATE DATABASE moodledb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'Moodle2025!@#';
GRANT ALL PRIVILEGES ON moodledb.* TO 'moodleuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## 📥 Phase 3: Moodle Installation

### Step 1: Download and Setup Moodle
```bash
# Navigate to web root
cd /var/www

# Clone Moodle (latest stable)
sudo git clone -b MOODLE_404_STABLE https://github.com/moodle/moodle.git

# Set permissions
sudo chown -R apache:apache /var/www/moodle
sudo chmod -R 755 /var/www/moodle

# Create moodledata directory
sudo mkdir /var/www/moodledata
sudo chown -R apache:apache /var/www/moodledata
sudo chmod -R 755 /var/www/moodledata
```

### Step 2: Configure Apache Virtual Host
```bash
# Create Moodle virtual host
sudo nano /etc/httpd/conf.d/moodle.conf
```

Add the following content:
```apache
<VirtualHost *:80>
    DocumentRoot /var/www/moodle
    <Directory /var/www/moodle>
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog /var/log/httpd/moodle_error.log
    CustomLog /var/log/httpd/moodle_access.log combined
</VirtualHost>
```

### Step 3: Restart Apache
```bash
sudo systemctl restart httpd
```

---

## Phase 4: PHP Configuration

### Step 1: Configure PHP Settings
```bash
# Edit PHP configuration
sudo nano /etc/php.ini
```

Update these settings:
```ini
memory_limit = 256M
upload_max_filesize = 200M
post_max_size = 200M
max_execution_time = 300
max_input_vars = 5000
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
```

### Step 2: Install PHP Development Tools
```bash
# Install development packages
sudo dnf install -y php-devel php-pear gcc libsodium-devel

# Install libsodium from source
cd /tmp
wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.19-stable.tar.gz
tar -xzf libsodium-1.0.19-stable.tar.gz
cd libsodium-stable
./configure
make && make check
sudo make install
sudo ldconfig
```

### Step 3: Install PHP 8.2 with Sodium Support
```bash
# Remove PHP 8.1
sudo dnf remove -y php8.1 php8.1-*

# Install PHP 8.2 with sodium
sudo dnf install -y php8.2 php8.2-sodium php8.2-cli php8.2-fpm php8.2-mysqlnd php8.2-xml php8.2-soap php8.2-intl php8.2-zip php8.2-gd php8.2-mbstring php8.2-opcache php8.2-pdo

# Copy PHP configuration
sudo cp /etc/php.ini.rpmsave /etc/php.ini

# Update max_input_vars
sudo sed -i 's/;max_input_vars = 1000/max_input_vars = 5000/' /etc/php.ini

# Start and enable services
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl restart httpd
```

### Step 4: Verify PHP Configuration
```bash
# Check PHP version
php -v

# Check sodium extension
php -m | grep sodium

# Check max_input_vars
php -i | grep max_input_vars
```

---

## Phase 5: Moodle Web Installation

### Step 1: Access Moodle Installation
1. **Open browser** → Navigate to `http://YOUR_PUBLIC_IP`
2. **Verify server checks** - All should show "OK"
3. **Click "Continue"**

### Step 2: Database Configuration
- **Database Host:** localhost
- **Database Name:** moodledb
- **Database User:** moodleuser
- **Database Password:** Moodle2025!@#
- **Database Port:** 3306
- **Tables Prefix:** mdl_

### Step 3: Admin Account Setup
- **Username:** admin (or your choice)
- **Password:** Strong password meeting requirements
- **First Name:** Admin
- **Last Name:** User
- **Email:** Your email address

### Step 4: Site Configuration
- **Full site name:** Your LMS name
- **Short name:** LMS
- **Timezone:** Your local timezone
- **Self registration:** Disable (recommended)

### Step 5: Complete Installation
- **Click "Save changes"**
- **Skip registration** (optional)
- **Access Moodle Dashboard**

---

## ✅ Phase 6: Post-Installation

### Step 1: Verify Installation
- ✅ Access Moodle dashboard
- ✅ All server checks pass
- ✅ Admin account works
- ✅ Database connection successful

### Step 2: Security Recommendations
```bash
# Set up SSL/HTTPS (recommended)
# Configure firewall rules
# Set up automated backups
# Configure email settings
```

### Step 3: Performance Optimization
```bash
# Enable OPcache
# Configure MySQL settings
# Set up caching
# Monitor system resources
```

---

## Troubleshooting

### Common Issues and Solutions

#### Issue: "PHP extension sodium must be installed"
**Solution:** Follow Phase 4, Step 3 to install PHP 8.2 with sodium support

#### Issue: "max_input_vars must be at least 5000"
**Solution:** Update php.ini with `max_input_vars = 5000`

#### Issue: "Connection timed out"
**Solution:** Check security group rules and ensure HTTP (80) is open

#### Issue: MySQL connection failed
**Solution:** Verify database credentials and MySQL service status

---

## System Requirements Met

- ✅ **PHP 8.2.29** with sodium extension
- ✅ **MySQL 8.0.43** database
- ✅ **Apache HTTP Server** configured
- ✅ **All required PHP extensions** installed
- ✅ **Proper file permissions** set
- ✅ **Security settings** configured

---

## Final Result

**A fully functional Moodle 4.4.11 Learning Management System running on AWS EC2 with:**
- Production-ready configuration
- Secure database setup
- Optimized PHP settings
- Complete admin access
- Ready for course creation and user management

---

## Support

For issues or questions:
1. Check server logs: `/var/log/httpd/` and `/var/log/mysqld.log`
2. Verify all services are running: `sudo systemctl status httpd php-fpm mysqld`
3. Check PHP configuration: `php -i`
4. Review Moodle documentation: https://docs.moodle.org/

---

**🎉 Congratulations! Your Moodle LMS is ready for production use!**
