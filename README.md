# Creating MongoDB (version 6.0.4) in Ubuntu 20.04

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

5. Park the the downloaded file into "temp", make sure the uri is valid

```
$ wget http://archive.ubuntu.com/ubuntu/pool/main/o/openssl/libssl1.1_1.1.1f-1ubuntu2.17_amd64.deb
```

6. Install the package libssl, this is part of SSL/TLS toolkit might used for SSL setup in MongoDB for communication channel

```
$ sudo dpkg -i ./libssl1.1_1.1.1f-1ubuntu2.17_amd64.deb
```

7. Re-fetch the latest package list

```
$ sudo apt update
```

8. Install MongoDB version 6. At this moment of writing is version "6.0.4"

```
$ sudo apt install mongodb-org
```

9. To enable a start a service at boot level

```
$sudo systemctl enable --now mongod
```

10. After installed, check the version

```
$ mongod --version
```

At this level is done on installation MongoDB in your Ubuntu, you may start using it or you wanted to have replication on MongoDB, feel free to read following steps

---

<br><br>

# Create replication on MongoDB with security enabled

Following setup, will have
1 primary node, 1 secondary node and 1 arbiter. That's the minimum setup

1. Assumed you have setup MongoDB on 3 instances/boxes.

   FYI - Arbiters are mongod members that are part of a replica set but do not hold data. It functions to do select order access to a shared resources among asyn requests.
   I assumed your IP would be

```
    172.31.10.30 - will be primary
    172.31.16.31 - will be secondary
    172.31.16.32 - will be for arbiter
```

2. Ensure the port (default 27017), that these 3 members able to communicate/visible each other. You may "mongosh" to login each other members.

```
eg. $ mongosh  "mongodb://172.31.10.30:27017/"
```

3. Generate keyfile for mongodb. This keyfile is generally internal authentication on the members vice versa.
   Make sure the access right from user "mongodb" and user group "mongodb"

```
$ openssl rand -base64 756 >  <path to keyfile>
eg. $ openssl rand -base64 756 >  /var/lib/mongodb/keyfile.txt

chmod 400 <path to keyfile>
eg. chmod 400 /var/lib/mongodb/keyfile.txt

change the owner
chown mongodb:mongodb /var/lib/mongodb/keyfile.txt
```

4. SFTP/FTP copy to the rest of members with this keyfile and ensure the access right (owner on step 3).

5. At member (172.31.10.30), login mongosh create user eg. username "system" with password "manager" with "root" as role

```
    $ mongosh
    > user admin
    > db.createUser(
    {
        user: "system",
        pwd: 'manager',
        roles: [ { role: "root", db: "admin" }, "readWriteAnyDatabase" ]
    }
```

6. Enable security feature on /etc/mongod.conf configuration in all members (arbiter member - not required). In this case is primary and secondary members.
   Next is enable the replica set names eg. "rs0"

```
eg.
  security:
    authorization: enabled
    keyFile: "/var/lib/mongodb/keyfile.txt"

    ...
    replication:
      replSetName: rs0


```

8. Now, restart all the replica set members starting from secondary (172.31.16.31) to primary (172.31.16.30)

9. Add the replica members into primary. Current sample using replica set "rs0"

```
    After you login with correct username/password
    > cfg = {
        "id":'rs0',
        members: [
            {'_id': 0, 'host': '172.31.10.30:27017' },
            {'_id': 1, 'host': '172.31.10.31:27017' },
            {'_id': 2, 'host': '172.31.10.32:27017',
            "arbiterOnly":true
            }
        ]
    }
    > rs.initiate(cfg)


    Note: arbiter is on 172.31.10.32
```

9. Testing - url string
   with authSource=admin and replica set "rs0"

```
    mongosh "mongodb://system:manager@172.31.10.30:27017,172.31.10.32:27017/?authSource=admin&replicaSet=rs0"
```

# Wanted to switch primary member

Due to some cases/reasons you wanted to switch primary to another member. Eg. You've increased the hardware specfication and decided increased spec instance deserved to be primary

1. To understand where is your primary (after login)

```
    > rs.isMaster().primary

    # it will print IP:Port
    eg. 172.31.10.30:27017

```

and wanted to switch 172.31.10.31 to become primary.

2. Get the configuration and understand location array of primary and secondary. Then set priority appropriately (0 to 1). Then rs.reconfig(cfg) will re-configure the settings

```
> cfg = rs.conf()
> cfg

printed following json
...
  members: [
    {
      _id: 0,
      host: '172.31.10.30:27017',
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 1,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    },
    {
      _id: 1,
      host: '172.31.10.31:27017',
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 0.5,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    },
    {
      _id: 2,
      host: '172.31.10.32:27017',
      arbiterOnly: true,
      buildIndexes: true,
      hidden: false,
      priority: 0,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    }
  ],
...

With that configure the members[1] to have lower priority. Then mongoDB able to reset the priority as primary

>cfg.members[0].priority = 0.5
>cfg.members[1].priority = 1
>rs.reconfig(cfg)
```
