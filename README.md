# ansible-platform-central-auth-config

Ansible content to configure **LDAP or SAML authentication on Ansible Automation Platform (AAP) 2.6+** through the **platform gateway** API, using the supported **`ansible.platform`** collection. This replaces the older automation controller pattern (`/api/v2/settings/ldap/`); gateway authenticators live under `/api/gateway/v1/authenticators/`.

Official background: [Configuring authentication in Ansible Automation Platform 2.6](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/access_management_and_authentication/gw-configure-authentication).

## Requirements

- **Ansible Core** 2.16 or newer (the `ansible.platform` collection [declares this](https://github.com/ansible/ansible.platform/blob/devel/meta/runtime.yml)).
- Network access from the control node to the platform gateway HTTPS endpoint.
- A gateway account (or OAuth token) with permission to manage authenticators and authenticator maps.

## Install the collection

From the repository root:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

This installs **`ansible.platform`** (version `>=2.5.0`), which provides `ansible.platform.authenticator` and `ansible.platform.authenticator_map`.

## Configure variables

Non-secret settings are composed in **`vars/ldap_config.yml`**. Tune these (or override with `-e`) per environment:

- **Gateway:** `aap_hostname`, `aap_validate_certs`
- **LDAP connection:** `ldap_uri_scheme`, `ldap_server_host`, `ldap_server_port`, `ldap_start_tls`, `ldap_opt_*`
- **Directory layout:** `ldap_base_dn`, `ldap_bind_user`, and derived search bases (`ldap_user_search_base`, `ldap_group_search_base` are built from `ldap_accounts_ou`)
- **Authenticator:** `aap_ldap_authenticator_name`, `aap_ldap_authenticator_order`, `aap_ldap_enabled`, `aap_ldap_create_objects`, `aap_ldap_remove_users`
- **Groups / users:** `ldap_group_type`, `ldap_group_type_params`, `ldap_group_search_filter`, `ldap_user_dn_template`, `ldap_user_attr_map`
- **Superuser map:** `ldap_superuser_group_cn` (full DN is `ldap_superuser_group_dn`), plus `aap_ldap_map_admin_*`

The `aap_ldap_configuration` dict is assembled from those variables and the bind password supplied at runtime (see below).

## Secrets and Ansible Vault

1. Copy the example vault vars file (this path is **gitignored** once you create it):

   ```bash
   cp vars/vault.yml.example vars/vault.yml
   ```

2. Edit **`vars/vault.yml`** and set:

   - **`vault_aap_username`** and **`vault_aap_password`**, **or** **`vault_aap_token`** (OAuth token for the gateway API)
   - **`vault_ldap_bind_password`** (LDAP bind account password)

3. Encrypt the file before committing anything else:

   ```bash
   ansible-vault encrypt vars/vault.yml
   ```

4. Run the playbook with a vault password (or password file):

   ```bash
   ansible-playbook playbooks/configure_aap_gateway_ldap.yml --ask-vault-pass
   # or: --vault-password-file ~/.vault_pass
   ```

Extra variables (`-e`) still override values from `vars/vault.yml` and `vars/ldap_config.yml` if you need a one-off. The authenticator task uses `no_log` to reduce accidental exposure of configuration payloads.

Optional: **`--check`** for check mode if your installed `ansible.platform` version supports it for these modules.

## What the playbook does

1. Ensures an LDAP **authenticator** exists with the settings from `vars/ldap_config.yml`.
2. Ensures an **authenticator map** grants `is_superuser` when your configured LDAP trigger matches (for example membership in an admin group).

Extend the repository with additional `ansible.platform.authenticator_map` tasks if you need organization, team, or role maps.

## SAML (platform gateway)

Non-secret SAML settings are composed in **`vars/saml_config.yml`** (copy from **`vars/saml_config.yml.example`**). The playbook **`playbooks/configure_aap_gateway_saml.yml`** ensures:

1. A SAML **authenticator** (`ansible_base.authentication.authenticator_plugins.saml`) with the same configuration shape as the gateway API.
2. An **authenticator map** of type **`allow`** with trigger **`always`**, matching a typical “allow all IdP users” mapping.

Put the **service provider private key** PEM in **`vars/vault.yml`** as **`vault_saml_sp_private_key`** (it must match **`saml_sp_public_cert`** in your SAML vars file). The gateway stores this encrypted; export it from your original key material, not from the API.

Run:

```bash
cp vars/saml_config.yml.example vars/saml_config.yml
ansible-playbook playbooks/configure_aap_gateway_saml.yml --ask-vault-pass
```

## LDAPS and custom CAs

If you use LDAPS and the directory presents a certificate that the gateway does not trust, add the CA to the gateway trust store as described in the product documentation (for example the LDAPS certificate procedures in the same [access management guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/access_management_and_authentication/gw-configure-authentication)). That step is separate from this playbook.
