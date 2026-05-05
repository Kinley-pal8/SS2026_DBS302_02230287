# Practical 6 - Part B: Securing MongoDB

### Authentication, Encryption, and Role-Based Access Control

---

**Module:** DBS302 - NoSQL Database Management  
**Practical:** 6 - Securing Redis and MongoDB  
**Part:** B - MongoDB Security  


---

## 1. Aim

To configure and verify authentication, Role-Based Access Control (RBAC), and TLS encryption on a MongoDB instance running on Kali Linux, and to perform a basic security audit of the configured database.

---

## 2. Objectives

By the end of this practical, the student should be able to:

1. Start a MongoDB instance without authentication to perform initial administrative setup.
2. Create an administrative user and enable authentication in the MongoDB configuration file.
3. Demonstrate that unauthenticated access is denied once authentication is enabled.
4. Create a custom role and application user to implement Role-Based Access Control (RBAC).
5. Generate self-signed TLS certificates and configure MongoDB to require encrypted connections.
6. Perform a security audit to verify that all security controls are correctly enforced.

---

## 3. Theory

### 3.1 Authentication

Authentication is the process of verifying the identity of a client attempting to connect to a database server. Without authentication, any client that can reach the MongoDB port can read, modify, or delete data freely. MongoDB implements authentication by requiring clients to provide a valid username and password before any database operations are permitted. Once enabled via the `security.authorization: "enabled"` directive in `mongod.conf`, MongoDB enforces authentication on every connection. The only exception is the localhost exception, which temporarily allows unauthenticated access when no users have been created yet, enabling the first administrative user to be set up safely.

### 3.2 Role-Based Access Control (RBAC)

Role-Based Access Control is a security model that restricts what actions a user can perform based on the roles they have been assigned. In MongoDB, roles define a set of privileges, where each privilege specifies a resource (a specific database and collection) and the actions that are permitted on that resource (such as find, insert, update, or remove). Users are then assigned one or more roles, and they can only perform actions that their roles explicitly allow. This principle of least privilege ensures that application users cannot access data outside their scope, reducing the risk of accidental or malicious data exposure.

### 3.3 TLS Encryption

Transport Layer Security (TLS) is a cryptographic protocol that protects data while it travels over a network. Without TLS, data transmitted between a MongoDB client and server is sent in plaintext, meaning that anyone monitoring the network can read or intercept it. TLS uses certificates to establish a trusted, encrypted channel between the client and server. In MongoDB, TLS is configured under the `net.tls` section of `mongod.conf`. When `mode: requireTLS` is set, MongoDB rejects all connections that do not use TLS, ensuring that all traffic is encrypted in transit. In this practical, self-signed certificates were generated using OpenSSL for demonstration purposes.

---

## 4. Environment

| Component          | Details                                 |
| ------------------ | --------------------------------------- |
| Operating System   | Kali GNU/Linux Rolling (Kernel 6.18.12) |
| MongoDB Version    | 8.2.7                                   |
| Mongosh Version    | 2.7.0                                   |
| OpenSSL Version    | OpenSSL 3.5.5                           |
| Data Directory     | /var/lib/mongodb                        |
| TLS Certificates   | /etc/mongo/tls/                         |
| Configuration File | /etc/mongod.conf                        |

---

## 5. Procedure and Observations

### 5.1 Step 0: Starting MongoDB Without Authentication (Baseline)

MongoDB was first started using the system service without authentication enabled. A connection was established using `mongosh` and `show dbs` was executed to confirm the server was running. At this stage, no users existed and authentication had not yet been enabled, so the connection was permitted under MongoDB's localhost exception.

**Command used:**

```bash
sudo systemctl start mongod
mongosh --host 127.0.0.1 --port 27017
```

**Observation:** The `mongosh` prompt was accessible and `show dbs` returned an empty result, confirming a fresh MongoDB instance with no data.

![Screenshot 1 - Baseline mongosh connection and show dbs output](/Practical_6b/screenshots/p6b-1.png)

---

### 5.2 Step 1: Creating the Administrative User

Before enabling authentication, the root administrative user `rootAdmin` was created in the `admin` database. This user was assigned three built-in MongoDB roles to allow full administrative control: `userAdminAnyDatabase`, `dbAdminAnyDatabase`, and `readWriteAnyDatabase`.

**Command used:**

```javascript
use admin;

db.createUser({
  user: "rootAdmin",
  pwd: "rootStrongPwd",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
});
```

**Observation:** MongoDB returned `{ ok: 1 }`, confirming the user was created successfully. This user was created before enabling authentication so that at least one administrator exists to manage the system after auth is turned on.

![Screenshot 2 - rootAdmin user created with { ok: 1 } response](/Practical_6b/screenshots/p6b-2.png)

---

### 5.3 Step 2: Enabling Authentication in mongod.conf

The MongoDB configuration file `/etc/mongod.conf` was edited to enable access control. The `security` section was updated as follows:

```yaml
security:
  authorization: "enabled"
```

MongoDB was then restarted using:

```bash
sudo systemctl restart mongod
sudo systemctl status mongod
```

**Observation:** The service restarted successfully and showed `active (running)` status. From this point, all clients connecting to MongoDB must authenticate with valid credentials.

![Screenshot 3 - mongod.conf showing authorization: enabled](/Practical_6b/screenshots/p6b-3.png)

![Screenshot 4 - systemctl status mongod showing active (running)](/Practical_6b/screenshots/p6b-4.png)

---

### 5.4 Step 3: Testing Authentication Enforcement

#### 5.4.1 Unauthenticated Connection (Should Fail)

An attempt was made to connect to MongoDB without providing any credentials:

```bash
mongosh --host 127.0.0.1 --port 27017
```

The `show dbs` command was executed inside the session.

**Observation:** MongoDB returned an authorization error, confirming that authentication is enforced:

```
MongoServerError: command listDatabases requires authentication
```

This is the expected and correct behaviour. Any client without valid credentials is denied access.

![Screenshot 5 - Authentication error when connecting without credentials](/Practical_6b/screenshots/p6b-5.png)

#### 5.4.2 Authenticated Connection as rootAdmin (Should Succeed)

A second connection was made using the `rootAdmin` credentials:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u rootAdmin -p rootStrongPwd \
  --authenticationDatabase admin
```

The connection status was then checked:

```javascript
db.runCommand({ connectionStatus: 1 });
```

**Observation:** The output confirmed the authenticated user as `rootAdmin` along with their assigned roles, verifying that the admin account functions correctly after authentication was enabled.

![Screenshot 6 - connectionStatus showing rootAdmin authenticated with roles](/Practical_6b/screenshots/p6b-6.png)

---

### 5.5 Step 4: Implementing Role-Based Access Control (RBAC)

#### 5.5.1 Creating a Custom Role

Logged in as `rootAdmin`, a custom role named `myAppRole` was created in the `myapp` database. This role grants privileges only on the `customers` collection within `myapp`:

```javascript
use myapp;

db.runCommand({
  createRole: "myAppRole",
  privileges: [
    {
      resource: { db: "myapp", collection: "customers" },
      actions: ["find", "insert", "update", "remove"]
    }
  ],
  roles: []
});
```

**Observation:** MongoDB returned `{ ok: 1 }`, confirming the custom role was created.

#### 5.5.2 Creating an Application User

An application user `appUser` was created and assigned the `myAppRole` role:

```javascript
db.createUser({
  user: "appUser",
  pwd: "appStrongPwd",
  roles: [{ role: "myAppRole", db: "myapp" }],
});
```

**Observation:** MongoDB returned `{ ok: 1 }`, confirming the user was created with restricted permissions.

![Screenshot 7 - createRole and createUser both returning { ok: 1 }](/Practical_6b/screenshots/p6b-7.png)

#### 5.5.3 Testing RBAC as appUser

A new session was opened as `appUser` and the following operations were tested:

**Allowed operations on myapp.customers:**

```javascript
use myapp;
db.customers.insertOne({ name: "Student One", city: "Phuntsholing" });
db.customers.find();
```

**Observation:** Both the `insertOne` and `find` operations succeeded, returning the inserted document as expected.

**Denied operation on admin database:**

```javascript
use admin;
db.system.users.find();
```

**Observation:** MongoDB returned an authorization error:

```
MongoServerError: not authorized on admin to execute command
```

This confirms that RBAC is correctly enforced. `appUser` is restricted to `myapp.customers` only and cannot access any other database or collection.

![Screenshot 8 - Allowed insertOne and find succeed; admin access denied with auth error](/Practical_6b/screenshots/p6b-8.png)

---

### 5.6 Step 5: Enabling TLS Encryption

#### 5.6.1 Generating Self-Signed Certificates

A dedicated directory `/etc/mongo/tls/` was created to store TLS certificates. The following OpenSSL commands were used to generate a Certificate Authority (CA) and a server certificate with Subject Alternative Names (SAN) to support IP-based connections:

```bash
# CA key and certificate
sudo openssl genrsa -out ca.key 4096
sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 \
  -out ca.pem \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=mongo-lab-ca"

# Server key and CSR
sudo openssl genrsa -out mongo.key 4096
sudo openssl req -new -key mongo.key -out mongo.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"

# Sign server certificate with SAN extension
sudo bash -c 'printf "subjectAltName=IP:127.0.0.1,DNS:localhost" > /etc/mongo/tls/san.ext'
sudo openssl x509 -req -in mongo.csr -CA ca.pem -CAkey ca.key -CAcreateserial \
  -out mongo.crt -days 365 -sha256 -extfile /etc/mongo/tls/san.ext

# Combine key and cert into single PEM
sudo bash -c 'cat mongo.key mongo.crt > mongo.pem'
```

**Note:** During initial certificate generation, a hostname mismatch error occurred (`IP: 127.0.0.1 is not in the cert's list`). This was resolved by regenerating the server certificate with a SAN extension explicitly listing `IP:127.0.0.1` and `DNS:localhost`.

**Observation:** All certificate files were created successfully in `/etc/mongo/tls/`: `ca.key`, `ca.pem`, `mongo.key`, `mongo.csr`, `mongo.crt`, `mongo.pem`, and `san.ext`.

![Screenshot 9 - ls -la /etc/mongo/tls/ showing all certificate files](/Practical_6b/screenshots/p6b-9.png)

#### 5.6.2 Configuring mongod.conf for TLS

The `/etc/mongod.conf` file was updated to require TLS for all connections:

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/tls/mongo.pem
    CAFile: /etc/mongo/tls/ca.pem
    allowConnectionsWithoutCertificates: true

security:
  authorization: "enabled"
```

MongoDB was restarted to apply the changes.

![Screenshot 10 - mongod.conf showing full net.tls and security sections](/Practical_6b/screenshots/p6b-10.png)

#### 5.6.3 Testing TLS Enforcement

**Connection without TLS (should fail):**

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

**Observation:** The connection was refused with:

```
MongoServerSelectionError: connection <monitor> to 127.0.0.1:27017 closed
```

This confirms that MongoDB no longer accepts non-TLS connections.

![Screenshot 11 - Connection refused without --tls flag](/Practical_6b/screenshots/p6b-11.png)

**Connection with TLS (should succeed):**

```bash
mongosh --host 127.0.0.1 --port 27017 --tls \
  --tlsCAFile /etc/mongo/tls/ca.pem \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

```javascript
use myapp;
db.customers.insertOne({ name: "TLS Test", city: "Thimphu" });
db.customers.find();
```

**Observation:** The connection succeeded. Both the insert and find operations completed successfully, returning:

```javascript
[
  { _id: ObjectId("..."), name: "Student One", city: "Phuntsholing" },
  { _id: ObjectId("..."), name: "TLS Test", city: "Thimphu" },
];
```

This confirms that TLS encryption is working correctly alongside authentication.

![Screenshot 12 - Successful TLS connection with insertOne and find output](/Practical_6b/screenshots/p6b-12.png)

---

### 5.7 Step 6: Security Audit

#### 5.7.1 Verifying appUser Roles

Logged in as `rootAdmin` with TLS, the `appUser` document was retrieved from `admin.system.users`:

```javascript
use admin;
db.system.users.find({ user: "appUser" }).pretty();
```

**Observation:** The output confirmed that `appUser` has only one role assigned:

```javascript
roles: [{ role: "myAppRole", db: "myapp" }];
```

This confirms that the user has no elevated privileges and is restricted to the custom role on `myapp` only.

![Screenshot 13 - appUser document showing only myAppRole on myapp](/Practical_6b/screenshots/p6b-13.png)

---

## 6. Observation Table

| #   | Security Check                                 | Expected Result                 | Actual Result                                           | Pass / Fail |
| --- | ---------------------------------------------- | ------------------------------- | ------------------------------------------------------- | ----------- |
| 1   | Anonymous access without credentials           | Authentication error            | `command listDatabases requires authentication`         | ✅ Pass     |
| 2   | `rootAdmin` login with correct password        | Successful login, roles shown   | Login succeeded, roles confirmed via `connectionStatus` | ✅ Pass     |
| 3   | `appUser` insert into `myapp.customers`        | Insert succeeds                 | `acknowledged: true`, document inserted                 | ✅ Pass     |
| 4   | `appUser` find on `myapp.customers`            | Documents returned              | Both records returned correctly                         | ✅ Pass     |
| 5   | `appUser` access `admin` database              | Authorization error             | `not authorized on admin to execute command`            | ✅ Pass     |
| 6   | Connection without `--tls` flag                | Connection refused              | `MongoServerSelectionError: connection closed`          | ✅ Pass     |
| 7   | Connection with `--tls` and `--tlsCAFile`      | Successful encrypted connection | Connected, data operations succeeded                    | ✅ Pass     |
| 8   | `appUser` role verification via `system.users` | Only `myAppRole` on `myapp`     | `roles: [ { role: 'myAppRole', db: 'myapp' } ]`         | ✅ Pass     |

---

## 7. Security Audit Summary

### 7.1 What is Secure

After completing all configuration steps, the following security controls are in place and verified:

- **Authentication is enforced.** No client can connect to MongoDB without providing valid credentials. Anonymous connections are denied with an authorization error.
- **Role-Based Access Control is working.** The `appUser` account can only perform CRUD operations on `myapp.customers` and is denied access to any other database or collection, including the `admin` database.
- **TLS encryption is enforced.** All connections without the `--tls` flag are rejected. Data in transit is encrypted between the client and server using the self-signed certificates generated during the practical.
- **Principle of least privilege is applied.** `appUser` has a custom role with only the minimum permissions required. They cannot drop collections, create indexes, or access any data outside their assigned scope.

### 7.2 What Still Needs Improvement

While the above controls provide a solid security baseline, the following weaknesses remain:

- **Weak passwords.** The passwords used in this lab (`rootStrongPwd`, `appStrongPwd`) are simple and predictable. In a production environment, passwords should be long, randomly generated, and stored in a secrets manager such as HashiCorp Vault.
- **Self-signed certificates.** The TLS certificates used were self-signed and not issued by a trusted Certificate Authority. In production, certificates from a trusted CA should be used to prevent man-in-the-middle attacks.
- **`bindIp` should be more restrictive.** Although currently set to `127.0.0.1` (localhost only), in a multi-server environment, MongoDB should be bound only to specific trusted internal IP addresses and protected behind a firewall.
- **No audit logging.** MongoDB's `auditLog` feature is not enabled. In a real deployment, audit logging should be turned on to track who performed which operations and when, which is important for compliance and incident response.
- **Certificate expiry.** The self-signed certificates are valid for only 365 days. A certificate rotation policy should be established so that expired certificates do not cause unexpected outages.
- **No network firewall rules.** In production, firewall rules (such as `ufw` or `iptables`) should be configured to block all external access to port 27017 and allow only trusted sources.

---

## 8. Conclusion

This practical demonstrated how to secure a MongoDB instance using three fundamental security controls: authentication, Role-Based Access Control, and TLS encryption. By enabling `security.authorization: "enabled"` in `mongod.conf`, all clients were required to provide valid credentials before any database operations could be performed. The RBAC implementation using a custom role `myAppRole` ensured that the application user `appUser` could only interact with the `myapp.customers` collection and was denied access to any other part of the system. Finally, TLS was configured using self-signed certificates generated with OpenSSL, ensuring that all network traffic between the client and server is encrypted in transit.

All eight security audit checks passed successfully, confirming that the security controls were correctly implemented and are functioning as intended. This practical highlights that securing a NoSQL database such as MongoDB requires a layered approach combining identity verification, access restriction, and encryption working together. In a production environment, these controls would be further strengthened with stronger passwords, trusted CA certificates, audit logging, and strict network firewall rules.

---

## 9. References

- MongoDB Documentation - Enable Access Control: https://www.mongodb.com/docs/manual/tutorial/enable-authentication/
- MongoDB Documentation - Role-Based Access Control: https://www.mongodb.com/docs/manual/core/authorization/
- MongoDB Documentation - TLS/SSL Configuration: https://www.mongodb.com/docs/manual/tutorial/configure-ssl/
- OpenSSL Documentation - Certificate Generation: https://www.openssl.org/docs/
- DBS302 Practical 6 Lab Guide - Securing Redis and MongoDB
