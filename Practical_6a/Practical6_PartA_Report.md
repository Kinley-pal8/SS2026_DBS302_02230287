# Practical 6: Securing Redis

## Authentication, Encryption, and Role-Based Access Control

---

**Module:** DBS302 - NoSQL Database Management  
**Practical:** 6 - Securing Redis and MongoDB  
**Part:** Part A - Securing Redis with ACL, RBAC, and TLS Encryption

---

## 1. Aim

To configure and verify authentication, role-based access control (RBAC), and TLS encryption on a Redis instance, and to demonstrate that security policies are correctly enforced through practical testing.

---

## 2. Objectives

By the end of this practical, the following objectives were achieved:

1. Start a Redis server instance and verify its availability.
2. Configure Access Control List (ACL) users with distinct permissions in `redis.conf`.
3. Test and verify that each user is restricted to their designated commands and key patterns.
4. Generate self-signed TLS certificates using OpenSSL.
5. Configure Redis to accept only TLS-encrypted connections.
6. Verify that TLS connections succeed and non-TLS connections are refused.
7. Demonstrate a Python application connecting to Redis over TLS with ACL-based authentication.

---

## 3. Theory

### 3.1 Authentication in Redis

Authentication is the process of verifying the identity of a client before granting access to the database. In older versions of Redis, authentication was handled by a single global password set using the `requirepass` directive. However, from Redis 6.0 onwards, a more robust system known as Access Control Lists (ACL) was introduced. ACL allows multiple named users to be created, each with their own password, making it possible to distinguish between different clients and apply different security policies to each one.

In this practical, three users were created: `admin`, `app_user`, and `monitoring`, each with a separate password. The built-in `default` user was disabled to prevent unauthenticated access.

### 3.2 Role-Based Access Control (RBAC) Using ACL

Role-Based Access Control is a security model in which permissions are grouped and assigned to users based on their role or function. Redis implements a form of RBAC through its ACL system. Each user can be granted access to specific commands (e.g., `+get`, `+set`) and specific key patterns (e.g., `~session:*`), which restricts what data they can read or modify and what operations they can perform.

This is important because the principle of least privilege dictates that each user or application should have only the minimum permissions necessary to perform their task. For example, a monitoring user should be able to read data and gather statistics but should never be able to write or delete data.

### 3.3 TLS Encryption

Transport Layer Security (TLS) is a cryptographic protocol that encrypts data travelling between a client and a server. Without TLS, all data including passwords, keys, and values are transmitted in plaintext, which means any attacker with access to the network can read or intercept them.

Redis supports TLS from version 6.0 onwards. To enable TLS, a certificate authority (CA), a server certificate, and a private key must be generated. The server uses these to establish an encrypted connection with clients. In this practical, self-signed certificates were generated using OpenSSL for demonstration purposes, which is acceptable in a lab environment.

---

## 4. Environment and Tools

| Component            | Details                                               |
| -------------------- | ----------------------------------------------------- |
| Operating System     | Kali Linux                                            |
| Redis Version        | 8.0.5                                                 |
| Shell                | Bash (zsh)                                            |
| Tools Used           | `redis-cli`, `openssl`, `nano`, `systemctl`, Python 3 |
| Python Library       | `redis` (system package)                              |
| TLS Certificate Type | Self-signed (OpenSSL)                                 |
| Configuration File   | `/etc/redis/redis.conf`                               |
| TLS Directory        | `/etc/redis/tls/`                                     |

---

## 5. Procedure

### 5.1 Step 0: Starting Redis and Verifying Operation

Redis was started using the systemd service manager. Before making any configuration changes, the server was confirmed to be running and responding to commands.

**Commands executed:**

```bash
redis-server --version
sudo systemctl start redis-server
redis-cli ping
```

**Observed output:**

```
PONG
```

The `PONG` response confirmed that Redis was running and accepting connections on the default port 6379.

![image.png](/Practical_6a/screenshots/p6a-1.png)
![image.png](/Practical_6a/screenshots/p6a-2.png)

---

### 5.2 Step 1: Troubleshooting Redis Startup Failure

During the initial restart of Redis after editing `redis.conf`, the service failed to start. The following diagnostic commands were used to identify the cause:

```bash
sudo journalctl -xeu redis-server.service --no-pager | tail -30
sudo tail -30 /var/log/redis/redis-server.log
```

The log revealed two problems:

1. A placeholder module directive in `redis.conf` referencing a non-existent file `/path/to/redisearch.so` was causing Redis to abort on startup.
2. The PID directory `/var/run/redis/` did not exist.

**Fix applied:**

```bash
sudo sed -i 's|^loadmodule /path/to/redisearch.so|#loadmodule /path/to/redisearch.so|' /etc/redis/redis.conf
sudo mkdir -p /var/run/redis
sudo chown redis:redis /var/run/redis
sudo systemctl restart redis-server
```

After applying these fixes, Redis started successfully and showed `active (running)` status.

---

### 5.3 Step 2: Configuring ACL Users

The `redis.conf` file was edited to define three users with different levels of access. The default user was disabled to prevent unauthenticated connections.

**Configuration added to `/etc/redis/redis.conf`:**

```conf
user default off

user admin on >adminStrongPwd ~* +@all

user app_user on >appStrongPwd ~session:* +get +set +del +expire +ttl +@connection

user monitoring on >monitorPwd ~* +@read +info +dbsize +lastsave +@connection
```

**Explanation of each user:**

- `default off` - Disables anonymous access entirely.
- `admin` - Full access to all keys and all commands. Intended for database administrators only.
- `app_user` - Can only access keys matching the pattern `session:*` and can only run basic data commands. Represents a web application's limited database user.
- `monitoring` - Can read all keys and run informational commands but cannot modify any data. Represents a monitoring or observability tool.

Redis was restarted after saving the configuration:

```bash
sudo systemctl restart redis-server
```

![image.png](/Practical_6a/screenshots/p6a-3.png)

---

### 5.4 Step 3: Testing ACL Users

#### 5.4.1 Admin User Test

```bash
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379
```

Commands run inside `redis-cli`:

```
ACL WHOAMI
set mykey "hello"
get mykey
ACL LIST
```

**Observed output:**

```
"admin"
OK
"hello"
1) "user admin on ..."
2) "user app_user on ..."
3) "user default off ..."
4) "user monitoring on ..."
```

The admin user successfully authenticated, performed read and write operations, and was able to view the full ACL list.

> _(Insert Screenshot 3 here – Admin user ACL WHOAMI and set/get)_
> ![image.png](/Practical_6a/screenshots/p6a-4.png)

> _(Insert Screenshot 6 here – ACL LIST output)_
> ![image.png](/Practical_6a/screenshots/p6a-7.png)

---

#### 5.4.2 Application User Test (RBAC Verification)

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

Commands run:

```
set session:user123 "data"
get session:user123
set otherkey "oops"
```

**Observed output:**

```
OK
"data"
(error) NOPERM this user has no permissions to access one of the keys used as arguments
```

The `app_user` was able to set and retrieve a key matching the `session:*` pattern, but was correctly denied when attempting to write to a key outside that pattern. This confirms that key-level RBAC is working as intended.

> _(Insert Screenshot 4 here – app_user RBAC key restriction)_
> ![image.png](/Practical_6a/screenshots/p6a-5.png)

---

#### 5.4.3 Monitoring User Test

```bash
redis-cli -u redis://monitoring:monitorPwd@127.0.0.1:6379
```

Commands run:

```
ACL WHOAMI
info server
set somekey "test"
```

**Observed output:**

```
"monitoring"
# Server
redis_version:8.0.5
...
(error) NOPERM this user has no permissions to run the 'set' command
```

The monitoring user could read server information but was correctly blocked from writing data.

---

### 5.5 Step 4: Generating TLS Certificates

Self-signed certificates were generated using OpenSSL. A Certificate Authority (CA) was created first, and then a server certificate was signed by that CA.

```bash
sudo mkdir -p /etc/redis/tls
cd /etc/redis/tls

sudo openssl genrsa -out ca.key 4096

sudo openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 \
  -out ca.crt \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=redis-lab-ca"

sudo openssl genrsa -out redis.key 4096

sudo openssl req -new -key redis.key -out redis.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"

sudo openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out redis.crt -days 365 -sha256
```

File ownership was set so the Redis process could read the certificates:

```bash
sudo chown redis:redis /etc/redis/tls/redis.key
sudo chmod 640 /etc/redis/tls/redis.key
sudo chown redis:redis /etc/redis/tls/redis.crt
sudo chown redis:redis /etc/redis/tls/ca.crt
```

> _(Insert Screenshot 7 here – TLS certificate files generated)_
> ![image.png](/Practical_6a/screenshots/p6a-8.png)

---

### 5.6 Step 5: Configuring Redis for TLS

The following TLS configuration was added to `/etc/redis/redis.conf`:

```conf
port 0

tls-port 6379

tls-ca-cert-file /etc/redis/tls/ca.crt
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key

tls-auth-clients no
```

Setting `port 0` disables the plain TCP port, forcing all connections to use TLS. The `tls-auth-clients no` option was set for the lab environment so that clients do not need to present their own certificate (one-way TLS).

Redis was restarted to apply the changes:

```bash
sudo systemctl restart redis-server
```

> _(Insert Screenshot 8 here – TLS configuration in redis.conf)_
> ![image.png](/Practical_6a/screenshots/p6a-9.png)

---

### 5.7 Step 6: Testing TLS Connection

#### 5.7.1 Successful TLS Connection

```bash
redis-cli --tls \
  --cacert /etc/redis/tls/ca.crt \
  -u rediss://app_user:appStrongPwd@127.0.0.1:6379
```

Commands run:

```
set session:user456 "secure"
get session:user456
```

**Observed output:**

```
OK
"secure"
```

The `rediss://` scheme (with double `s`) indicates a TLS connection. Data was successfully written and retrieved over an encrypted channel.

> _(Insert Screenshot 9 here – Successful TLS connection)_
> ![image.png](/Practical_6a/screenshots/p6a-10.png)

#### 5.7.2 Failed Non-TLS Connection

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379
```

**Observed output:**

```
I/O error
Error: Server closed the connection
```

Since the plain TCP port was disabled (`port 0`), the non-TLS connection was refused, confirming that encryption is enforced.

![image.png](/Practical_6a/screenshots/p6a-11.png)

---

### 5.8 Step 7: Python Application Demo

A Python script was written to demonstrate a real application connecting to Redis using both TLS and ACL authentication.

**Script (`redis_secure_demo.py`):**

```python
import redis

def create_redis_client():
    client = redis.Redis(
        host="127.0.0.1",
        port=6379,
        username="app_user",
        password="appStrongPwd",
        ssl=True,
        ssl_ca_certs="/etc/redis/tls/ca.crt",
        ssl_cert_reqs="none",
        decode_responses=True,
    )
    return client

if __name__ == "__main__":
    r = create_redis_client()
    print("Connected successfully as app_user (TLS)")
    r.set("session:python_demo", "hello from python")
    print("Value:", r.get("session:python_demo"))
```

**Command to run:**

```bash
python3 ~/redis_secure_demo.py
```

**Observed output:**

```
Connected successfully as app_user (TLS)
Value: hello from python
```

The script successfully connected to Redis over TLS as `app_user`, wrote a session key, and retrieved it. This demonstrates that application-level integration with secured Redis works correctly.

![image.png](/Practical_6a/screenshots/p6a-12.png)

---

## 6. Observations

The following table summarises all security tests performed and their results:

| Test                       | Command / Action             | Expected Result    | Actual Result                         | Pass/Fail |
| -------------------------- | ---------------------------- | ------------------ | ------------------------------------- | --------- |
| Anonymous access           | `redis-cli ping` (no auth)   | Denied             | Connection refused (default user off) | ✅ Pass   |
| Admin login                | `ACL WHOAMI` as admin        | `"admin"`          | `"admin"`                             | ✅ Pass   |
| Admin full write           | `set mykey "hello"`          | OK                 | OK                                    | ✅ Pass   |
| app_user session key write | `set session:user123 "data"` | OK                 | OK                                    | ✅ Pass   |
| app_user non-session key   | `set otherkey "oops"`        | NOPERM error       | NOPERM error                          | ✅ Pass   |
| monitoring read            | `info server`                | Success            | Server info returned                  | ✅ Pass   |
| monitoring write           | `set somekey "test"`         | NOPERM error       | NOPERM error                          | ✅ Pass   |
| ACL LIST                   | View all users as admin      | All 4 users listed | All 4 users listed                    | ✅ Pass   |
| TLS connection             | Connect with `--tls` flag    | Success            | OK, data exchanged                    | ✅ Pass   |
| Non-TLS connection         | Connect without `--tls`      | Refused            | I/O error, connection closed          | ✅ Pass   |
| Python TLS demo            | Run `redis_secure_demo.py`   | Value printed      | `hello from python` returned          | ✅ Pass   |

---

## 7. Security Audit Summary

### 7.1 What Is Secure

- **Authentication is enforced.** The default user is disabled, and all clients must provide valid credentials to connect.
- **Key-level RBAC is working.** The `app_user` is restricted to `session:*` keys only and cannot access or modify any other keys.
- **Command-level RBAC is working.** The `monitoring` user cannot execute write commands such as `SET`, `DEL`, or `FLUSHALL`.
- **TLS encryption is enforced.** Plain TCP connections on port 6379 are disabled. All clients must use TLS, ensuring data in transit is encrypted.
- **Principle of least privilege is applied.** Each user has only the permissions necessary for their role. No application user has admin-level access.

### 7.2 What Could Be Improved

- **Password strength.** The passwords used in this lab (`adminStrongPwd`, `appStrongPwd`) are acceptable for demonstration but should be replaced with randomly generated, cryptographically strong passwords in a real environment.
- **Self-signed certificates.** The TLS certificates used are self-signed, which means clients must trust the CA manually. In production, certificates should be issued by a trusted Certificate Authority.
- **`tls-auth-clients no`.** Currently clients are not required to present their own certificate. Enabling mutual TLS (`tls-auth-clients yes`) with client certificates would add an additional layer of security.
- **Network binding.** Redis is bound to `127.0.0.1`, which is correct for a single-server setup. In distributed environments, the `bind` address and firewall rules should be carefully configured to allow only trusted hosts.
- **No audit logging.** Redis does not natively log every command executed by each user. An external tool or Redis module would be needed for full audit trail capability.

---

## 8. Conclusion

This practical successfully demonstrated the three core pillars of database security - authentication, role-based access control, and encryption - applied to a Redis instance. By configuring ACL users, it was shown that different clients can be given precisely defined permissions at both the command and key level, enforcing the principle of least privilege. The TLS configuration ensured that all data transmitted between the client and server is encrypted, protecting against network-level interception.

The troubleshooting experience during this practical also highlighted the importance of reading server logs carefully, as the actual cause of startup failures (a broken module path and permission issues on certificate files) was only visible in the Redis log file and not in the systemd output alone.

Overall, these security measures significantly reduce the attack surface of a Redis deployment and reflect best practices for securing NoSQL databases in real-world applications.

---
