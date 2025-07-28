# Fractyl
> by Trackpoint (akeyboarduser), and hopefully others!
### a decentralized, deniable, reputation-based file sharing system

This document describes a file sharing system that is decentralized (there is no one server that can be taken down),
there is theoretically no way to prove that a hosting node is hosting a certain file, or that it is hosting any file(s) at all.

This is a conceptual design and a unofficial RFC (Request for Comments). All comments and contributions are welcome.

# Concept
> (alright, how should this work?)

The goal of this project is to have a file sharing system where:

1. There is no single point of failure
2. It is plausibly deniable that a hosting node is hosting:  
  (i) a certain file  
  (ii) any files at all
3. Users are pseudo-anonymous, where no true information is required about them outside of what they intentionally share.
4. It is possible to block misbehaving users from uploading files

# Design
> (alright, how will this work?)

By points from the [Concept](#concept) section:

1. This is quite simple. We provide a public specification for how it __should__ work and hopefully a reference implementation, and encourage users to host nodes.
2. When uploading a file, the user's software will:  
    1. Derive a 'superkey' from the encryption key provided by the user. Personally, I (Trackpoint) recommend using argon2id.
    2. Generate filenames from the true filename with ctr as a counter starting at 0, and H as a hash function: `H( H(superkey) || H(filename) || H(ctr))`. ctr is incremented for every filename created. I (Trackpoint) would use SHA3-512 here as H.
    3. Generate N mebibytes of random data (I (Trackpoint) suggest 512 MiB) for every filename
    4. Derive a true encryption key from the superkey. I (Trackpoint) yet again suggest using argon2id here.
    5. Encrypt the file using the true encryption key, with AES-CFB. Pad if necessary, using PKCS#7 padding.
    6. Randomly generate a value using the superkey as a seed. The value must be between 0 and N-128KiB. This value will be known as X<sub>1</sub>. Create a list of used values and add every number X<sub>1</sub> to X<sub>1</sub>+128KiB inclusive.
    7. Insert the first 128KiB of ciphertext into the first file with the first generated filename (ctr=0) at X<sub>1</sub>, overwriting the random data already there.
    8. Repeat 6 and 7 for every 128KiB block of ciphertext, changing to the next file (ctr=ctr<sub>-1</sub>+1) every time you reach 45% saturation (exactly 45% or more of the file is ciphertext, this leads to approx. 230.4 MiB of true ciphertext storage per file. This is quite low, but it's a tradeoff of security vs space usage.)
    9. Upload those files with the generated filenames, not uploading the encryption key or true filename. These files may be uploaded to completely different hosting nodes.
3. Users must create a keypair, and publish the public key to all nodes. Their public key is their identity, and they sign every file upload with their private key. This signature will however be encrypted.
4. All users have a reputation, and starting out the reputation is set intentionally extremely low (-100,000). A user, when uploading a file, must complete PoW challenges given to them by the servers, and the difficulty of the PoW challenges depend on the user's reputation. When uploading a file, a user will include their public key, but it will be encrypted with a brand new generated key. This key will be split into 1,500 pieces and all of those pieces will be sent out to trusted nodes, to be used in a 500-of-1500 Shamir secret sharing scheme, so that if any 500 nodes all together agree that a user needs to be banned, they can reconstruct the key, decrypt the public key (identity), and then all nodes together will tank the user's reputation to -100,000,000 which makes it practically infeasible to upload any more files. Then all other files can be quickly searched for the same public key and they can be removed by hosting nodes. This discourages Sybil attacks since it is computationally hard to upload from brand new identities. If there aren't 1500 trusted nodes to send them out to, then the pieces can be sent out to nodes in this preferred order: active hosting node, active uploading node, active regular node, hosting node, uploading node, regular node.

Reputations are tracked by each node separately, and majority decision is used, e.g. 10 nodes say that a user's reputation is -10,000, 15 nodes say that it is -15,000, and so it is -15,000. A Sybil attack is also possible here, so when a new node is created it must also create itself an identity and prove itself to other nodes via PoW.  
Reputations can be increased or decreased by user ratings.

# Types of nodes:

* Regular node: This is just a node that doesn't host any files, only downloads. It doesn't do anything particularly special.
* Active regular node: Regular node, but it's up for at least 7 days.
* Uploading node: This is a regular node, but it also uploads.
* Active uploading node: Uploading node, but it's up for at least 7 days.
* Hosting node: This is a node that hosts files. It can also download and upload.
* Active hosting node: Hosting node, but it's up for at least 30 days.
* Trusted node: This is the highest rank of node. To be a trusted node, any other node has to be online for at least 7 days straight and must have a identity with exactly 0 reputation (meaning perfect reputation). Trusted nodes do not have to do a PoW challenge at all when uploading, since the difficulty would be 0 due to their perfect reputation.

# help needed!
This project is a work in progress and, I (Trackpoint), believe that it would be quite a great feat to finish this project. I could never finish it by myself, which is why I'm releasing this document publicly. Don't be afraid to send me comments or questions at my [email](mailto:akeyboarduser@gmail.com) (akeyboarduser@gmail.com), or submit merge (pull) requests.