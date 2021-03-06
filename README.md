## HDWallet: Secure, hierarchical Bitcoin wallet generation

**The implementation is based on the proposal [BIP 0032](https://en.bitcoin.it/wiki/BIP_0032) and is currently in audit mode. Please do not use in production yet. Testing welcome.**

A common problem for Bitcoin enabled webservices is secure storage of user funds. Usually one generates for each new user an own address to track what user owns which coins. 


The classic generation of a new bitcoin address requires basically 3 steps:

1. Generate a large 256-bit number from a (P)RNG (= private key)
2. Calculate the ECC public key for that number (= public key)
3. Do some hashing and encoding on the pubkey (= bitcoin address)


If the webserver gets hacked and the private keys are stored on it an attacker can steal all user funds.

### Address generation without a private key

Based on the mathematical properties of ECC we can apply equivalent operations on a private key and its public key. The resulting keys will be a new corresponding keypair. In pseudocode:

```python
N = 42
privkey1, pubkey1 = generate_keypair()

privkey2 = privkey1 × N
pubkey2  = pubkey1 × N

assert(is_keypair(privkey2, pubkey2) == True)
```

We apply the operation on the public key on the webserver to generate new bitcoin addresses no private key is needed.
To spend the funds later, we derive the private for the address in a secure, offline environment.

For creating a hierachical wallet structure we use the child derivation function described in [BIP 0032](https://en.bitcoin.it/wiki/BIP_0032).

### Installation
```bash
pip install hdwallet
```

### Code example

```python
from hdwallet import HDWallet

# 1. generate a master wallet with a (random) seed 
master = HDWallet.from_master_seed('HDWallet seed')
# 2. store the Private Extended Key somewhere very (!) safe
prv_master_key = master.to_extended_key(include_prv=True)
# 3. store the Public Extended Key on the webserver
pub_master_key = master.to_extended_key()


# 4. On the webserver we can generate child wallets
webserver_wallet = HDWallet.from_extended_key(pub_master_key)
child2342 = webserver_wallet.child(23).child(42)
print '- Public Extended Key (M):', pub_master_key
print 'Child: M/23/42'
print 'Address:', child2342.address()
print 'Privkey:', child2342.prvkey() # ... but the private keys remain *unknown*
print ''


# 5. In case we need the private key for a child wallet, start with the private master key
cold_wallet = HDWallet.from_extended_key(prv_master_key)
child2342 = cold_wallet.child(23).child(42)
print '- Private Extended Key (m):', prv_master_key
print 'Child: m/23/42'
print 'Address:', child2342.address()
print 'Privkey:', child2342.prvkey().encode('hex')
```

The code above produces the following output
```
- Public Extended Key (M): xpub661MyMwAqRbcG7J3wr5XW6FEvDHbqUJrKGEXVUJMTuJWJqwV1rJWZPENd1jDkEdaGVs1RGMp5oWpJ89xTu2bkRSRSoxVpMzLG1ZXiYbBBgZ
Child: M/23/42
Address: 1PT7U5s9PTuXs1RK3ygoxFuAL59UjJZALV
Privkey: None

- Private Extended Key (m): xprv9s21ZrQH143K3dDaqpYX8xJWNBT7S1azx3Jvh5tjuZmXS3cLUJzG1autmmmeSFzRZF2GQm7Dr8f36saxwZGoHjhHb4u33FgkTi3cvjgFeFK
Child: m/23/42
Address: 1PT7U5s9PTuXs1RK3ygoxFuAL59UjJZALV
Privkey: 04c0ff8c288ce5f21f7f3e5e50db24e13c75d72e6815a448deb35568c6c8ed1f
```

