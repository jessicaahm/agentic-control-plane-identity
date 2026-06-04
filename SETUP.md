# Setup Vault

> Please note that this is a standalone vault instance for POC purposes only. This is not for production. For production ready, refer to our [reference architecture](https://developer.hashicorp.com/vault/tutorials/day-one-raft/raft-reference-architecture)

Refer to [config/vault.sh](./config/vault.sh)

Please ensure that you set up the following:
```sh
# Update to your VM IP Addresses
export VM_IP_ADDR="<UPDATE>"
export VM_PRIVATE_IP_ADDR="<UPDATE>"
export VAULT_PATH="/vault" #Update if you wanted to change the path where vault store the configuration and raft storage
```

Once initialize, vault will create a file which store the unseal key and root key in the following path `"$VAULT_PATH"/vault_init_output.txt`