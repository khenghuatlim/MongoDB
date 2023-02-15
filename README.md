# Creating MongoDB in Ubuntu 20.04

1. Do apt-get to fetch the all latest version of the package list from your distro's software repository which including the 3rd parties.

```
$ sudo apt update
```

2. Install following packages that required

```
$ sudo apt install wget curl gnupg2 software-properties-common apt-transport-https ca-certificates lsb-release
```

3. curl the content (f - handling fail content, s - silent, S - show error, L - location)
   gpg - encryption and signing tool (--dearmor encrypt or decrypt, -o write output to specify location)
   Make sure the URI is valid. Sometimes the provide changed it

```
$ sudo curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/mongodb-6.gpg
```

4. Write a new repository list into /etc/apt/sources.list.d/

```
$ echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
```
