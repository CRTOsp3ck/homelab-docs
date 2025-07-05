# Public Key Cryptography: A Complete Guide

## Table of Contents
1. [Introduction](#introduction)
2. [The Fundamentals](#the-fundamentals)
3. [How Key Pairs Are Generated](#how-key-pairs-are-generated)
4. [The Authentication Process](#the-authentication-process)
5. [Mathematical Foundation](#mathematical-foundation)
6. [Real-World Examples](#real-world-examples)
7. [Security Principles](#security-principles)
8. [Common Algorithms](#common-algorithms)
9. [Practical Implementation](#practical-implementation)
10. [Troubleshooting](#troubleshooting)

## Introduction

Public key cryptography, also known as asymmetric cryptography, is a cryptographic system that uses pairs of mathematically related keys: a **public key** and a **private key**. This system enables secure communication and authentication without sharing secret information.

### Why Public Key Cryptography Matters

- **Secure remote access** (SSH, HTTPS)
- **Digital signatures** and authentication
- **Encrypted communication** without pre-shared secrets
- **CI/CD automation** (Jenkins, GitHub Actions)
- **Certificate-based security** (SSL/TLS)

## The Fundamentals

### Key Concepts

```
┌─────────────────┐    ┌─────────────────┐
│   Private Key   │    │   Public Key    │
│                 │    │                 │
│ • Keep Secret   │◄──►│ • Share Freely  │
│ • Sign/Decrypt  │    │ • Verify/Encrypt│
│ • Proves ID     │    │ • Validates ID  │
└─────────────────┘    └─────────────────┘
```

### The Core Principle

**Asymmetric Property**: What one key does, only the other key can undo.

- **Private key signs** → Public key verifies
- **Public key encrypts** → Private key decrypts

## How Key Pairs Are Generated

### Generation Process

```bash
# Example: Generating an Ed25519 key pair
ssh-keygen -t ed25519 -C "user@example.com" -f ~/.ssh/my_key

# This creates:
# ~/.ssh/my_key      (private key - keep secret!)
# ~/.ssh/my_key.pub  (public key - share this)
```

### What Happens During Generation

1. **Entropy Collection**
   ```
   System gathers random data from:
   - Mouse movements
   - Keyboard timings  
   - System events
   - Hardware random number generators
   ```

2. **Mathematical Computation**
   ```
   Algorithm processes entropy to create:
   - Large prime numbers (for RSA)
   - Points on elliptic curves (for Ed25519)
   - Mathematically linked key pairs
   ```

3. **Key Pair Output**
   ```
   Private Key: Large secret number/point
   Public Key:  Mathematically derived counterpart
   ```

## The Authentication Process

### SSH Authentication Flow

```
┌─────────────┐                           ┌─────────────┐
│   Client    │                           │   Server    │
│ (Jenkins)   │                           │ (GitHub)    │
└─────────────┘                           └─────────────┘
       │                                          │
       │ 1. Connection Request                    │
       │─────────────────────────────────────────►│
       │                                          │
       │ 2. "Prove you own the private key"       │
       │◄─────────────────────────────────────────│
       │                                          │
       │ 3. Challenge Data                        │
       │◄─────────────────────────────────────────│
       │                                          │
       │ 4. Sign Challenge with Private Key       │
       │                                          │
       │ 5. Send Signed Response                  │
       │─────────────────────────────────────────►│
       │                                          │
       │ 6. Verify Signature with Public Key     │
       │                                          │
       │ 7. Access Granted/Denied                 │
       │◄─────────────────────────────────────────│
```

### Detailed Step-by-Step Process

#### Step 1: Connection Initiation
```bash
# Client initiates SSH connection
ssh git@github.com

# Server responds with supported authentication methods
Server: "Available auth methods: publickey, password"
Client: "I'll use publickey authentication"
```

#### Step 2: Key Identification
```bash
# Client sends public key fingerprint
Client: "I have a key with fingerprint: SHA256:abc123..."
Server: "I found that key in authorized_keys for user 'git'"
```

#### Step 3: Challenge Creation
```bash
# Server creates a random challenge
Server generates: random_data = "f7a3b2c1d4e5..."
Server: "Sign this data to prove you have the private key"
```

#### Step 4: Digital Signature
```bash
# Client uses private key to sign the challenge
signature = sign(random_data, private_key)
Client sends: signature back to server
```

#### Step 5: Verification
```bash
# Server verifies signature using stored public key
if verify(signature, random_data, public_key):
    Server: "Authentication successful"
    # Connection established
else:
    Server: "Authentication failed"
    # Connection rejected
```

## Mathematical Foundation

### RSA Algorithm Example

**Key Generation:**
```
1. Choose two large prime numbers: p = 61, q = 53
2. Calculate n = p × q = 61 × 53 = 3233
3. Calculate φ(n) = (p-1)(q-1) = 60 × 52 = 3120
4. Choose e = 17 (public exponent)
5. Calculate d = 2753 (private exponent)

Public Key:  (n=3233, e=17)
Private Key: (n=3233, d=2753)
```

**Signing Process:**
```
Message: m = 123
Signature: s = m^d mod n = 123^2753 mod 3233 = 855

Verification: m' = s^e mod n = 855^17 mod 3233 = 123
If m' = m, signature is valid!
```

### Ed25519 Algorithm (Simplified)

**Based on Elliptic Curve Cryptography:**
```
Curve: y² = x³ + ax + b (mod p)
Private Key: Random 256-bit number
Public Key: Private_key × Generator_point_on_curve

Signing: Uses curve mathematics to create signature
Verification: Uses curve mathematics to verify signature
```

## Real-World Examples

### Example 1: Jenkins to GitHub Authentication

**Setup Phase:**
```bash
# 1. Jenkins generates key pair
ssh-keygen -t ed25519 -f /home/jenkins/.ssh/github_key

# 2. Copy public key to GitHub
cat /home/jenkins/.ssh/github_key.pub
# Add this to GitHub → Repository → Settings → Deploy Keys
```

**Authentication Phase:**
```bash
# 3. Jenkins connects to GitHub
git clone git@github.com:user/repo.git

# Behind the scenes:
# - Jenkins uses private key to sign challenge
# - GitHub verifies with stored public key
# - Access granted, clone proceeds
```

### Example 2: CI/CD Pipeline Deployment

**SSH Key Distribution:**
```bash
# Production server setup
mkdir -p ~/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5..." >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Jenkins deployment
scp -i /path/to/private_key app.tar.gz user@prod-server:/opt/app/
ssh -i /path/to/private_key user@prod-server 'systemctl restart app'
```

### Example 3: Docker Registry Authentication

**Harbor Registry Access:**
```bash
# Generate key for registry access
ssh-keygen -t rsa -b 4096 -f ~/.ssh/harbor_key

# Add public key to Harbor user profile
# Use private key in CI/CD for automated pushes/pulls
docker login -u username --password-stdin < harbor_key
```

## Security Principles

### 1. Key Protection

**Private Key Security:**
```bash
# Correct permissions
chmod 600 ~/.ssh/private_key

# Wrong permissions (too open)
chmod 644 ~/.ssh/private_key  # ❌ Others can read!

# Key storage locations
~/.ssh/            # User keys
/etc/ssh/          # System keys  
/var/lib/jenkins/  # Service keys
```

**Public Key Distribution:**
```bash
# Safe to share
cat ~/.ssh/key.pub | email-to-admin@company.com

# Add to multiple servers
ssh-copy-id -i ~/.ssh/key.pub user@server1
ssh-copy-id -i ~/.ssh/key.pub user@server2
```

### 2. Key Rotation

**Regular Key Rotation:**
```bash
# Generate new key pair
ssh-keygen -t ed25519 -f ~/.ssh/new_key

# Update all services with new public key
# Remove old public key from services
# Delete old private key securely
shred -u ~/.ssh/old_key
```

### 3. Principle of Least Privilege

**Dedicated Keys for Different Purposes:**
```
~/.ssh/
├── github_deploy_key      # GitHub repository access
├── github_deploy_key.pub
├── production_deploy_key  # Production server access
├── production_deploy_key.pub
├── harbor_registry_key    # Docker registry access
└── harbor_registry_key.pub
```

## Common Algorithms

### Comparison of Key Types

| Algorithm | Key Size | Security Level | Speed | Use Case |
|-----------|----------|----------------|-------|----------|
| RSA | 2048-4096 bits | High | Slower | Legacy systems, certificates |
| Ed25519 | 256 bits | Very High | Very Fast | Modern SSH, Git |
| ECDSA | 256-521 bits | High | Fast | Certificates, signing |
| DSA | 1024-3072 bits | Medium | Medium | Legacy (deprecated) |

### Algorithm Selection Guide

**For SSH Keys:**
```bash
# Best choice (2024)
ssh-keygen -t ed25519

# Good alternative
ssh-keygen -t ecdsa -b 521

# Legacy/compatibility
ssh-keygen -t rsa -b 4096
```

**For Different Use Cases:**
- **SSH Authentication**: Ed25519
- **Git Signing**: Ed25519 or RSA 4096
- **Web Certificates**: ECDSA P-256 or RSA 2048
- **Code Signing**: RSA 4096 or ECDSA P-384

## Practical Implementation

### Setting Up SSH Keys for Different Services

#### GitHub Setup
```bash
# Generate GitHub-specific key
ssh-keygen -t ed25519 -C "jenkins@company.com" -f ~/.ssh/github_key

# Configure SSH client
cat >> ~/.ssh/config << EOF
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes
EOF

# Test connection
ssh -T git@github.com
```

#### Server Access Setup
```bash
# Generate server access key
ssh-keygen -t ed25519 -C "deploy-key" -f ~/.ssh/server_key

# Copy to server
ssh-copy-id -i ~/.ssh/server_key.pub user@production-server

# Test access
ssh -i ~/.ssh/server_key user@production-server
```

#### Jenkins Configuration
```bash
# Store private key in Jenkins credentials
# 1. Jenkins UI → Manage Jenkins → Credentials
# 2. Add Credential → SSH Username with private key
# 3. ID: "github-ssh-key"
# 4. Username: "git"
# 5. Private Key: paste contents of ~/.ssh/github_key

# Use in Jenkinsfile
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github-ssh-key',
                    url: 'git@github.com:user/repo.git'
            }
        }
    }
}
```

### Key Management Scripts

**Automated Key Rotation:**
```bash
#!/bin/bash
# rotate-keys.sh

OLD_KEY="~/.ssh/deploy_key"
NEW_KEY="~/.ssh/deploy_key_new"
SERVERS=("server1.example.com" "server2.example.com")

# Generate new key
ssh-keygen -t ed25519 -f $NEW_KEY -N ""

# Deploy new key to all servers
for server in "${SERVERS[@]}"; do
    ssh-copy-id -i ${NEW_KEY}.pub user@$server
done

# Test new key on all servers
for server in "${SERVERS[@]}"; do
    ssh -i $NEW_KEY user@$server "echo 'New key works on $server'"
done

# Remove old key from servers
for server in "${SERVERS[@]}"; do
    ssh -i $NEW_KEY user@$server \
        "sed -i '/$(cat ${OLD_KEY}.pub | cut -d' ' -f2)/d' ~/.ssh/authorized_keys"
done

# Securely delete old key
shred -u $OLD_KEY ${OLD_KEY}.pub

# Rename new key to standard name
mv $NEW_KEY $OLD_KEY
mv ${NEW_KEY}.pub ${OLD_KEY}.pub
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Permission Denied (publickey)

**Symptoms:**
```
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

**Debugging:**
```bash
# Verbose SSH connection
ssh -vT git@github.com

# Check SSH agent
ssh-add -l

# Test specific key
ssh -i ~/.ssh/specific_key -T git@github.com
```

**Solutions:**
```bash
# Fix key permissions
chmod 600 ~/.ssh/private_key
chmod 644 ~/.ssh/private_key.pub

# Add key to SSH agent
ssh-add ~/.ssh/private_key

# Verify public key is added to service
cat ~/.ssh/private_key.pub
```

#### 2. Host Key Verification Failed

**Symptoms:**
```
Host key verification failed.
fatal: Could not read from remote repository.
```

**Solution:**
```bash
# Add host to known_hosts
ssh-keyscan github.com >> ~/.ssh/known_hosts

# Or accept interactively
ssh -T git@github.com
# Type "yes" when prompted
```

#### 3. Multiple Keys Confusion

**Symptoms:**
```
# Wrong key being used
ssh -T git@github.com
# Uses id_rsa instead of github_key
```

**Solution:**
```bash
# Configure SSH client properly
cat >> ~/.ssh/config << EOF
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_key
    IdentitiesOnly yes
EOF
```

#### 4. Jenkins Key Access Issues

**Common Problems:**
- Jenkins user can't access private key
- Wrong file permissions
- Key not in Jenkins credentials store

**Solutions:**
```bash
# Fix ownership
sudo chown jenkins:jenkins /home/jenkins/.ssh/deploy_key
sudo chmod 600 /home/jenkins/.ssh/deploy_key

# Test as jenkins user
sudo -u jenkins ssh -i /home/jenkins/.ssh/deploy_key user@server

# Verify Jenkins credentials
# UI → Manage Jenkins → Credentials → Test Connection
```

### Debugging Commands

**SSH Debugging:**
```bash
# Maximum verbosity
ssh -vvv user@server

# Test specific identity file
ssh -i /path/to/key -T git@github.com

# Show SSH agent keys
ssh-add -L

# Show key fingerprints
ssh-keygen -lf ~/.ssh/key.pub
```

**Key Validation:**
```bash
# Verify key pair match
ssh-keygen -y -f ~/.ssh/private_key > /tmp/extracted_public_key
diff ~/.ssh/private_key.pub /tmp/extracted_public_key

# Check key format
ssh-keygen -l -f ~/.ssh/key.pub

# Test key encryption
openssl rsa -in ~/.ssh/key -check  # For RSA keys
```

## Conclusion

Public key cryptography is the foundation of modern secure communication and authentication. Understanding how key pairs work, how authentication flows operate, and how to properly implement and troubleshoot these systems is essential for:

- **DevOps Engineers**: CI/CD pipeline security
- **System Administrators**: Server access management  
- **Developers**: Git repository access and code signing
- **Security Professionals**: Infrastructure security design

### Key Takeaways

1. **Mathematical Foundation**: Key pairs are mathematically linked - only the private key can create signatures that the public key can verify
2. **Authentication Flow**: Challenge-response mechanism proves possession of private key without transmitting it
3. **Security Best Practices**: Protect private keys, rotate regularly, use principle of least privilege
4. **Proper Implementation**: Use appropriate algorithms, configure SSH clients correctly, manage credentials securely

### Next Steps

- Practice setting up SSH keys for different services
- Implement automated key rotation procedures
- Set up monitoring for key usage and expiration
- Review and audit existing key management practices

---

*This document provides a comprehensive foundation for understanding and implementing public key cryptography in real-world scenarios. For specific implementation questions, consult the official documentation for your tools and services.*