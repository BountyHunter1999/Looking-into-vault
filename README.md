# How to run?

## File System Backend

- `docker compose -f docker-compose.file.yml up  -d vault-filesystem` run vault with filesystem backend
- `docker exec -it my_vault sh` connect to vault container
- `vault status` get vault status, initally vault is at sealed status (Sealed true)

### About the files:

- `Dockerfile.file` is the vault docker system with Filesystem storage backend which will not scale beyond a single server
- `Dockerfile.consul` designed for distributed systems, dockerized Consul storage backend for the vault.
  ![Consul](./consul.png)
- `Dockerfile` dockerzied vault with consul storage backend configs

### Initialize vault cluster

- `vault operator init` generate 5 master key shares and root token which we'll use to authenticate with the http client

  ```
  Unseal Key 1: ZjTsyr1IY/hvxm+BxfYV4e+jEaBzynOhA8+Smu0ZN1EO
  Unseal Key 2: Nz5ItVq97V1Cvtmb4qQTlFo2652UkZQLFkcNnUUNkQk+
  Unseal Key 3: +V2w2Bh3ku0SvVfsA6qg0kXWfP3ozXwUD2/Zn0DVAzy+
  Unseal Key 4: OyA5n2sd3irChOwoGcJnm1oGZCBdaju2Rr1kJupJKQoU
  Unseal Key 5: 9XUX43DC6z5ytrTORUT9oJXH5CrEBodMtSujc8tpwM/1

  Initial Root Token: s.FHH94aTuh2oOWoN4wTmjbqYZ
  ```

### Unseal Vault Server

- `vault operator unseal` Execute this command (vault operator unseal command) 3 times with 3 different key shares

### Login

- `vault login` login to vault with root token

### Adding secrets and getting them

- `vault secrets enable kv` enable kv secret engine
- `vault kv put kv/mikeyy ops=lambda` create new secret with a key `ops` and value `lambda` within the `kv/mikeyy` path
- `vault kv get kv/mikeyy` get secret from the `kv/mikeyy` path

#### With HTTP

- add secret with HTTP API (this is running on port 8200)

  > replace the $VAULT_TOKEN with actual vault token

  ```bash
  curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{ "data": { "ops": "koko" } }' \
  http://0.0.0.0:8200/v1/kv/bassa
  ```

- get secret with HTTP API

  > replace the $VAULT_TOKEN with actual vault token

      ```bash
      curl -H "X-Vault-Token: s.FHH94aTuh2oOWoN4wTmjbqYZ" \
      -H "Content-Type: application/json" \
      -X GET \
      http://0.0.0.0:8200/v1/kv/bassa
      ```

  - response
    ```json
    {
      "request_id": "c7e41510-6f71-4dae-c4b0-ec92c0d66337",
      "lease_id": "",
      "renewable": false,
      "lease_duration": 2764800,
      "data": {
        "data": {
          "ops": "koko"
        }
      },
      "wrap_info": null,
      "warnings": null,
      "auth": null
    }
    ```

## Consul Backend

- `docker compose up -d`
- `docker exec -it vault_consul bash`
- `vault status` see vault status
- `vault operator init` initialize vault cluster

  ```
  Unseal Key 1: P+LJRgUcelR8y5r8DqJf0LvdXkKe4AdvwQf8qBaaxC4b
  Unseal Key 2: UxB4ttyxUo+FjeoBrzFuAFno4dRyuZGQnE67AcSh64HA
  Unseal Key 3: OUSNJoewPA5ItBFxGYKxrTymp5qPl+AvaDFgexwwTsBm
  Unseal Key 4: e6lzHth25ysGDtIi73LEnzaKsHp/nNm4L9s1j4S1txAW
  Unseal Key 5: 00+13Zb01HY9DtcVf8B59EyOmgGmIslMhxk41NyhMNgY
  Initial Root Token: s.0WDDlDEg7SZlDxVlB413E7jl
  ```

- `vault operator unseal` run 3 times and provide the unseal key

## Integrating with postgres

> **Unseal vault (`vault operator unseal`) and login (`vault login`)**

> [Docs](https://developer.hashicorp.com/vault/docs/secrets/databases/postgresql)

- `vault secrets enable database`: Enable the database secrets engine if it is not already enabled
- configure vault with proper plugin and connection information

  ```bash
  vault write database/config/db \
    plugin_name="postgresql-database-plugin" \
    allowed_roles="my-role" \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/db?sslmode=disable" \
    username="admin" \
    password="adminpw" \
    password_authentication="scram-sha-256"
  ```

- Configure a role that maps a name in Vault to an SQL statement to execute to create the database credential

  ```bash
  vault write database/roles/my-role \
    db_name="db" \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
  ```

> the `db` in enable database and configure role statement referes to the database created from postgres container

### Get the credentials

```bash
vault read database/creds/my-role
```
output:
```
Key                Value
---                -----
lease_id           database/creds/my-role/GOvn2QYs7y6FXzuwJAITLkCG
lease_duration     1h
lease_renewable    true
password           lUSV0N-1mpGdLLDMKoGf
username           v-root-my-role-jqHwX6NMmz7LPqyL4ykg-1706427475
```
