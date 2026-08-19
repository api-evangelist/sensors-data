---
name: sensors-data-provision-account-access
description: Provision, scope, suspend and remove a person's access to a Sensors Data deployment through the Portal identity API — the joiner/mover/leaver flow.
api: Sensors Portal OpenAPI (神策业务门户)
generated: '2026-08-13'
method: generated
source: openapi/sensors-data-portal-identity-v2-openapi.yml
operations:
  - ListAccounts
  - GetAccountByName
  - AddAccount
  - UpdateAccount
  - AddAccountRole
  - DeleteAccountRole
  - InviteAccountToProject
  - DisableAccount
  - EnableAccount
  - DeleteAccount
  - ListRoles
  - GetRole
  - GetAccountByApiKey
---

# Provision and de-provision access

Base: `{base}/api/v3/portal/v2`. This is the shared identity plane behind every Sensors
product, so a change here moves a person's access across Analytics, Focus and Horizon at
once. Handle with corresponding care.

The calling api-key must belong to an administrator account — the key inherits the
permissions of the account it was minted for, and identity writes require an admin.

## Joiner

1. `GetAccountByName` — `GET /identity/account/get-by-name` — confirm the person does not
   already exist. There is no idempotency key; `AddAccount` twice creates two accounts.
2. `ListRoles` — `GET /identity/role/list` — read the real role ids. Never hard-code one.
3. `AddAccount` — `POST /identity/account/add`.
4. `AddAccountRole` — `POST /identity/account/role/add` — grant least privilege. An account
   with no role has no access; that is the correct starting state.

## Mover

- `InviteAccountToProject` — `POST /identity/account/project/invite` — extend an existing
  account into another project rather than creating a second identity for the same human.
- `AddAccountRole` / `DeleteAccountRole` — adjust scope. Do the removal in the same change
  as the addition; role accretion is how a reader becomes an administrator by accident.
- `UpdateAccount` — `POST /identity/account/update` — profile fields.

## Leaver

- `DisableAccount` — `POST /identity/account/disable` — the reversible step. Use this
  first. `EnableAccount` reverses it.
- `DeleteAccount` — `POST /identity/account/delete` — irreversible, and it orphans anything
  the account owned. The contract exposes `delete_account_id` and `archive_account_id`
  fields precisely so ownership can be reassigned; populate them.

## Audit an unknown key

`GetAccountByApiKey` — `POST /identity/account/get-by-api-key` — resolves an api-key to its
owning account. This is the operation to reach for when a key of unknown provenance turns
up in a config file or a log: it tells you whose permissions that key is carrying.

Related: `account-id` is a documented impersonation header usable only with an
administrator key. When set, the call executes as that account and the audit log attributes
it to them. Use it deliberately and never as a convenience.

## Rules

- Test `code == "SUCCESS"`; the HTTP status is `default` for both success and failure.
- Log `request_id` on every identity write — this is the surface you will be asked to
  reconstruct during an access review.
- `page_index` / `page_size` (max 100) on `ListAccounts`; `expand_field=last_access_time`
  hydrates last-access, which is what dormant-account cleanup needs.
