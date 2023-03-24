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
<img src="/_other/architecture.png" alt="Kerberos logo" width="420" height="270" />
  &nbsp;&nbsp;&nbsp;
<img src="/_other/architecture2.png" alt="Kerberos logo" width="420" height="270" />
</p>


#### 2. Environnement

In order to proceed with the configurations we need to have a : 
- Domain name : "kdc.insat.tn"
- Realm : "INSAT.TN"
- Three machines : 

 | Machine Name     |   Machine IP   | Sub-domain name    |
 |    :---:         |     :---:      |    :---:           |
 | KDC              | 192.168.56.108 | kdc.insat.tn       |
 | Pg server        | 192.168.56.107 | postgres.insat.tn  |
 | Client           | 192.168.56.106 | client.insat.tn    |
 > machines IP's are just an example, use `hostname -I` to get each machine ip. <br>
 > All the configurations must be done in **root** mode, use `su -` to connect as root.
 
<p align="right">(<a href="#top">back to top</a>)</p>

#### 2. DNS (Domain name system)
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

#### 3. Time Synchronization
When the client obtains a ticket from Kerberos, it includes in its message the current time of day. One of the three parts of the response from Kerberos is a timestamp issued by the Kerberos server.

3.1. on the **_KDC_** install **ntp**:
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
3.2. on the **_server_** install **ntp** and **ntpdate**:
```bash
apt install ntp
apt install ntpdate
```
then edit the `/etc/ntp.conf` and add the lines below under the `# Use Ubuntu's ntp server as a fallback` section: 
```bash
pool ntp.ubuntu.com
server 192.168.56.108
server obelix
```

3.3. Synchronize time by running the below command on the server machine:
```bash
ntpdate -dv 192.168.56.108
```
<p align="right">(<a href="#top">back to top</a>)</p>
