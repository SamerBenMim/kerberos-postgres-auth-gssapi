# kerberos-postgres-auth-gssapi
This project showcases the setup and configuration of a Kerberos authentication system between a client, a Postgres server, and a Key Distribution Center (KDC). The client can securely authenticate to the Postgres server using Kerberos tickets obtained from the KDC, enabling SSO (Single-Sign-On) access to the database. The project includes three virtual machines: the KDC, the Postgres server, and the client, and demonstrates how to configure each of them to establish a secure connection using Kerberos.

<br>
<p align="center">
<img src="/_other/kerberos-logo.png" alt="Kerberos logo" width="420" height="270" />
</p>



## Getting Started

In order to run this project, we need to follow some few steps : 

### Prerequisites

* Make sure that you have a virtualization software.In this demo i used **Oracle VM VirtualBox** ( [Download Here](https://www.virtualbox.org/wiki/Downloads)).
* Make sure you have 3 linux machines with the **Ubuntu 20.04 LTS distribution**  ( [Download Here](https://ubuntu.com/download/desktop)).
* Make sure you have **postgresql** on the server and client machine.
```bash
  sudo apt update
  sudo apt-get install Postgresql postgresql-contrib
``` 



### Kerberos Configuration

#### 1. Architecture
<p align="center">
<img src="/_other/architecture.png" alt="Kerberos logo" width="390" height="250" />
  &nbsp;&nbsp;
<img src="/_other/architecture2.png" alt="Kerberos logo" width="390" height="250" />
</p>


#### 2. Environnement

In order to proceed with the configurations we need to have a : 
- Domain name : "kdc.insat.tn"
- Realm : "INSAT.TN" (must be all uppercase)
- Three machines : 

 | Machine Name     |   Machine IP   | Sub-domain name    |
 |    :---:         |     :---:      |    :---:           |
 | KDC              | 192.168.56.108 | kdc.insat.tn       |
 | Pg server        | 192.168.56.107 | postgres.insat.tn  |
 | Client           | 192.168.56.106 | client.insat.tn    |
 > machines IP's are just an example, use `hostname -I` to get each machine ip. <br>
 > All the configurations must be done in **root** mode, use `su -` to connect as root.
 
<p align="right">(<a href="#top">back to top</a>)</p>

#### 3. DNS (Domain name system)
Used to match domain name to their IP's.
```bash
sudo nano /etc/hosts
```
and add _(for each machine)_ : 
```bash
192.168.56.108    kdc.insat.tn         kdc
192.168.56.107    postgres.insat.tn    server
192.168.56.106    client.insat.tn      client
```
then set the **hostname**  _(for each machine)_ :
 | Machine Name     |            set new hostname                   | 
 |    :---:         |              :---:                            |
 | KDC              | `hostnamectl set-hostname kdc.insat.tn `    |
 | server           | `hostnamectl set-hostname postgres.insat.tn`  |
  | client           | `hostnamectl set-hostname client.insat.tn`  |

 

<p align="right">(<a href="#top">back to top</a>)</p>

#### 4. Time Synchronization
When the client obtains a ticket from Kerberos, it includes in its message the current time of day. One of the three parts of the response from Kerberos is a timestamp issued by the Kerberos server.

4.1. on the **_KDC_** install **ntp**:
```bash
apt install ntp
```
then edit the `/etc/ntp.conf` and add the lines below under the `# local users may interrogate the ntp server more closely` section: 
```bash
restrict 127.0.0.1
restrict ::1
restrict 192.168.56.108 mask 255.255.255.0
nomodify notrap
server 127.127.1.0 stratum 10
listen on *
```
4.2. on the **_server_** install **ntp** and **ntpdate**:
```bash
apt-get install ntp
apt-get install ntpdate
```
then edit the `/etc/ntp.conf` and add the lines below under the `# Use Ubuntu's ntp server as a fallback` section: 
```bash
pool ntp.ubuntu.com
server 192.168.56.108
server obelix
```

4.3. Synchronize time by running the below command on the server machine:
```bash
ntpdate -dv 192.168.56.108
```
<p align="right">(<a href="#top">back to top</a>)</p>


#### 5. Configure KDC (Key Distribution Center)

5.1. We need to install **the packages** _krb5-kdc_, _krb5-admin-server_ and _krb5-config_ by running : 
```bash
apt-get install krb5-kdc krb5-admin-server krb5-config
```
During installation you will be prompted to enter the _realm_, _kerberos server_ and _administartive server_ and it would be in order:

 | Prompt                  |    value       | 
 |    :---:                |     :---:      |
 | Realm                   | INSAT.TN       |
 | Kerberos servers        | kdc.insat.tn   |
 | Administrative Service  | kdc.insat.tn   |
 
>Its capital sensitive.<br>
>View kdc settings with `cat /etc/krb5kdc/kdc.conf`.<br>

5.2 Now we need to add **kerberos database** where principals will be stored
```bash
sudo krb5_newrealm
```
> You will be prompted to choose a password.

5.3 we will create an _admin principal_ , a _host principal_ and generate its keytab:
- **principal:** a unique identity to which Kerberos can assign tickets.
- **keytab:** stores long-term keys for one or more principals and allow server applications to accept authentications from clients, but can also be used to obtain initial credentials for client applications.
run the following commands:
```bash
kadmin.local                              # login as local admin
addprinc root/admin                       # add admin principal
```
> We can check if the user root/admin was successfully created by running the command : `kadmin.local: list_principals`. We should see the 'root/admin@INSAT.TN' principal listed along with other default principals.
> 
> type `q` to exit.

5.4 Grant the **admin principal** all privileges by editing `/etc/krb5kdc/kadm5.acl`:
```bash
*/admin@INSAT.TN *                              # just uncomment the line
```
5.4 restart the kerberos service by running: 
```bash
systemctl restart krb5-admin-server
systemctl status krb5-admin-server        # to check service status
```
5.5 we create a principal for the client
```bash
   $ sudo kadmin.local
    kadmin.local:  addprinc samer
```
We create also a principal for the service server
```bash
   kadmin.local:  add_principal postgres/pg.insat.tn
   kadmin.local:  list_principals
```

<p align="right">(<a href="#top">back to top</a>)</p>

#### 6. Configure Service Server Machine
### Configuration of Kerberos*

6.1. We need to install **the packages** _krb5-user_, _libpam-krb5_ and _libpam-ccreds_ by running: 
```bash
sudo apt-get update
sudo apt install krb5-user libpam-krb5 libpam-ccreds
```
During installation you will be prompted to enter the _realm_, _kerberos server_ and _administartive server_ and it would be in order:
 | Prompt                  |    value         | 
 |    :---:                |     :---:        |
 | Realm                   | INSAT.TN       |
 | Kerberos servers        | kdc.insat.tn   |
 | Administrative Service  | kdc.insat.tn   |
>Its capital sensitive.<br>
>View krb settings with `cat /etc/krb5.conf`.

PS : *We need to enter the same information used for KDC Server.*

6.2 Preparation of the keytab file:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;a. In the KDC machine run the following command to generate the keytab file in the current folder :
```bash
  $ ktutil 
   ktutil:  add_entry -password -p postgres/pg.insat.tn@INSAT.TN -k 1 -e aes256-cts-hmac-sha1-96
   Password for postgres/postgres.insat.tn@INSAT.TN: 
   ktutil:  wkt postgres.keytab
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;b. Send the keytab file from the KDC machine to the Service server machine :

In the Postgres server machine make the following directories :

`mkdir -p /home/postgres`

In the KDC machine send the keytab file to the Postgres server :

`scp postgres.keytab postgres@<POSTGRES_SERVER_IP_ADDRESS>:/home/postgres/pgsql`


! We need to have **openssh-server** package installed on the service server : 

`sudo apt-get install openssh-server`.
   


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;c. Verify that the service principal was succesfully extracted from the KDC database :

   1. List the current keylist
        
        `ktutil:  list`
        
   2. Read a krb5 keytab into the current keylist
   
   `ktutil:  read_kt pgsql/data/postgres.keytab`

   3. List the current keylist again 
       
       `ktutil:  list`

*### Configuration of the service (PostgreSQL)*

##### Installation of PostgreSQL 

1. Update the package lists 
 
   `sudo apt-get update`
 
2. Install necessary packages for Postgres

   `sudo apt-get install postgresql postgresql-contrib`

3. Ensure that the service is started

   `sudo systemctl start postgresql`



##### Create a Postgres Role for the Client 

We will need to :

* create a new role for the client
    
    `create user samer with encrypted password 'my_password';`

* create a new database

    `create database samer;`

* grant all privileges on this database to the new role

    `grant all privileges on database samer to samer;`


To ensure the role was successfully created run the following command :

```
postgres=# SELECT usename FROM pg_user WHERE usename LIKE 'samer';
```


The client samer has now a role in Postgres and can access its database 'samer'.


##### Update Postgres Configuration files (*postgresql.conf* and *pg_hba.conf* )

* Updating *postgresql.conf*

To edit the file run the following command :

`sudo vi /etc/postgresql/<pg_version>/main/postgresql.conf`

By default, Postgres Server only allows connections from localhost. Since the client will connect to the Postgres server remotely, we will need to modify *postgresql.conf* so that Postgres Server allows connection from the network :

```
listen_addresses = '*'
```

 We will also need to specify the keytab file location :
 ```
 krb_server_keyfile = '/home/postgres/postgres.keytab'
 ```

* Updating *pg_hba.conf*

HBA stands for host-based authentication. *pg_hba.conf* is the file used to control clients authentication in PostgreSQL. It is basically a set of records. Each record specifies a **connection type**, a **client IP address range**, a **database name**, a **user name**, and the **authentication method** to be used for connections matching these parameters.

The first field of each record specifies the **type of the connection attempt** made. It can take the following values :

* `local` : Connection attempts using *Unix-domain sockets* will be matched. 
* `host` : Connection attempts using *TCP/IP* will be matched (SSL or non-SSL as well as GSSAPI encrypted or non-GSSAPI encrypted connection attempts).
* `hostgssenc` : Connection attempts using *TCP/IP* will be matched, but only when the connection is made with GSSAPI encryption.

This field can take other values that we won't use in this setup. For futher information you can visit the official [documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).


```
# IPv4 local connections:
hostgssenc   samer     samer           <IP_ADDRESS_RANGE>         gss include_realm=0 krb_realm=INSAT.TN
```

And comment other connections over TCP/IP.

`krb_realm=INSAT.TN` : Only users of INSAT.Tn realm will be accepted.

`include_realm=0` : If [include_realm](https://www.postgresql.org/docs/current/gssapi-auth.html) is set to 0, the realm name from the authenticated user principal is stripped off before being passed through the user name mapping. In a multi-realm environments this may not be secure unless krb_realm is also used. 


For changes to take effect we need to restart the service :   `sudo systemctl restart postgresql`.

### Client Machine Configuration

Following are the packages that need to be installed on the Client machine : <br>

```
$ sudo apt-get update
$ sudo apt-get install krb5-user libpam-krb5 libpam-ccreds
```
 
During the installation, we will be asked for configuration of :
 * the realm : 'INSAT.TN' (must be *all uppercase*)
 * the Kerberos server : 'kdc.insat.tn'
 * the administrative server : 'kdc.insat.tn'


PS : *We need to enter the same information used for KDC Server.*


## User Authentication

Once the setup is complete, it's time for the client to authenticate using kerberos.

First, try to connect to PostgreSQL remotely :

`$ psql -d samer -h postgres.insat.tn -U samer` # No pg_hba.con entry found for host xxx


*-d* specifies the database, *-U* specifies the postgres role and *-h* specifies the ip address of the machine hosting postgres.

In the client machine check the cached credentials :

`$ klist` # No credentails found

Then initial the user authentication :

`$ kinit samer`

And check the ticket granting ticket (TGT) :

`$ klist`


Now try to connect once again and check the service ticket  !

<p align="right">(<a href="#top">back to top</a>)</p>

## Acknowledgments
A list of resources which are helpful and would like to give credit to:
* [GSSAPI kerberos](https://idrawone.github.io/2020/03/11/PostgreSQL-GSSAPI-Authentication-with-Kerberos-part-1/
)
* [Kerberos docs](https://web.mit.edu/kerberos/krb5-latest/doc/)
* [Postgresql with KRB](https://www.postgresql.org/docs/current/gssapi-auth.html)

* [TECHWALL : Useful video tutorial](https://www.youtube.com/watch?v=vx2vIA2Ym14)


