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

## Architecture
<p align="center">
<img src="/_other/architecture.png" alt="Kerberos logo" width="420" height="270" />
<img src="/_other/architecture2.png" alt="Kerberos logo" width="420" height="270" />
</p>
