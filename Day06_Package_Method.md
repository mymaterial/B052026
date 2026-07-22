# Installing PostgreSQL on RHEL-based Systems

https://github.com/user-attachments/assets/414303c6-6256-4884-97e6-7aa3c6484806

## Step 1: Add PostgreSQL Repository
Install the PostgreSQL Global Development Group (PGDG) repository.
```sh
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

## Step 2: Disable Default PostgreSQL Module
Ensure the default PostgreSQL module is disabled to avoid conflicts.
```sh
dnf -qy module disable postgresql
```

## Step 3: Install PostgreSQL 17 Server
This command installs the PostgreSQL 16 server package and automatically creates a `postgres` user at the OS level.
```sh
dnf install -y postgresql17-server
```

## Step 4: Initialize the Database
Run the following command to initialize the PostgreSQL database.
```sh
/usr/pgsql-16/bin/postgresql-17-setup initdb
```

## Step 5: Enable and Start PostgreSQL Service
Enable PostgreSQL to start on boot and then start the service.
```sh
systemctl enable postgresql-17
systemctl start postgresql-17
```

# Custom Data Directory Configuration
If you need to set up a custom data directory:
## Step 1: Stop the existing postgres, if any
```
systemctl stop postgresql-17
```
## Step 2: Decide the location for the data directory
```
mkdir -p /u01/pgsql/17
chown -R postgres:postgres /u01/pgsql/17
```

## Step 3: Edit the `PGDATA` variable in the PostgreSQL service file.
```sh
vi /lib/systemd/system/postgresql-17.service
```

## Step 4: Reload the systemd daemon to apply changes.
```sh
systemctl daemon-reload
```

## Step 5: Initialize the Database
Run the following command to initialize the PostgreSQL database.
```sh
/usr/pgsql-16/bin/postgresql-17-setup initdb
```

## Step 6: Enable and Start PostgreSQL Service
Enable PostgreSQL to start on boot and then start the service.
```sh
systemctl enable postgresql-17
systemctl start postgresql-17
systemctl status postgresql-17
```
