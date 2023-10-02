# github-tf-backend

## Steps to setup github repo as remote state

1. Create ssh key pair for github repository authentication from github actions (Don't put any passphrase while creating the keys)

    ```bash
        ssh-keygen -t rsa -C "<repo-url>"
    ```

2. Copy the ssh public key and upload it to 'Deploy Keys' section of the repository that will be used to store terraform state file. Also allow write access for the deploy key.
3. Copy the private key and and store it in `SSH_PRIVATE_KEY` secrets variable in the terraform configuration infrastructure repository.
4. **Setup state encryption**
   1. We are using sops as encryption abstraction by setting the `TF_BACKEND_HTTP_ENCRYPTION_PROVIDER` to `sops`.
   2. Create PGP keys:

        ```bash
            gpg --full-generate-key
        ```

   3. Copy the public fingerprint of the key using `gpg --list-secret-keys --keyid-format=long` command. Store the fingerprint in `TF_BACKEND_HTTP_SOPS_PGP_FP` secrets variable in github. Store the key passphrase in `PASSPHRASE`
   4. Extract the private key using `gpg --armor --export-secret-key <key_id> > private.key` and store it in `GPG_PRIVATE_KEY` secret variable.
5. Create a `terraform-backend-git.hcl` file and store the ssh url of the repository, branch, and state file name.

## Lock/Unlock state

### Lock

```bash
# Checkout current ref requested by user and cleanup any leftovers
git reset --hard
git checkout ${ref}
git branch -D locks/${file}
# Pull latest remote state
git pull origin ${ref}
# Start a new locking branch
git checkout -b locks/${file}
# Save lock metadata
echo ${lock} > ${file}.lock
git add ${file}.lock
git commit -m "Lock ${file}"
git push origin locks/${file}
# If push failed saying that fast forward is not possible - something else had it already locked
```

### Check existing Lock

```bash
# Checkout current ref requested by user and cleanup any leftovers
git reset --hard
git checkout ${ref}
git branch -D locks/${file}
# Fetch locks
git fetch origin refs/heads/locks/*:refs/remotes/origin/locks/*
# Checkout the lock branch, if it fails - it wasn't locked
git checkout locks/${file}
# Check if it was locked by me
cat ${file}.lock
```

### Unlock

```bash
# First - use routine from above to check that it is currently locked and the lock author is me.
# Then - it's a matter of deleting the lock branch remotely
git push origin --delete locks/${file}
```

### Get state

```bash
# Checkout current ref requested by user and cleanup any leftovers
git reset --hard
git checkout ${ref}
# Pull latest
git pull origin ${ref}
# Read state
cat ${file}
```

### Update state

```bash
# First - use routine from above to check that it is currently locked and the lock author is me.
# Then - checkout current ref requested by user and cleanup any leftovers
git reset --hard
git checkout ${ref}
# Pull latest
git pull origin ${ref}
# Save state
echo ${state} > ${file}
git add ${file}
git commit -m "Update ${file}"
git push origin ${ref}
```

## Delete state

```bash
# First - use routine from above to check that it is currently locked and the lock author is me.
# Then - checkout current ref requested by user and cleanup any leftovers
git reset --hard
git checkout ${ref}
# Pull latest
git pull origin ${ref}
# Delete state
git rm -f ${file}
git commit -m "Delete ${file}"
git push origin ${ref}
```
