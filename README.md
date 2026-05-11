# ansible-platform-central-auth-config

Ansible content to configure **SAML or LDAP authentication on Ansible Automation Platform (AAP) 2.6+** through the **platform gateway** API, using the supported **`ansible.platform`** collection. This replaces the older automation controller pattern; gateway authenticators live under `/api/gateway/v1/authenticators/`.

**This branch** adds the SAML gateway playbook and `vars/saml_config.yml.example`. The LDAP playbook remains available for directory-based auth.

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

## SAML (platform gateway)

Non-secret SAML settings are composed in **`vars/saml_config.yml`** (copy from **`vars/saml_config.yml.example`**; that path is **gitignored** once created). The playbook **`playbooks/configure_aap_gateway_saml.yml`** ensures:

1. A SAML **authenticator** (`ansible_base.authentication.authenticator_plugins.saml`) with the configuration shape expected by the gateway API.
2. An **authenticator map** of type **`allow`** whose triggers default to **`always`**, matching a typical “allow all IdP users” mapping.

### SAML variables (`vars/saml_config.yml`)

Tune these (or override with `-e`) per environment:

- **Gateway:** `aap_hostname`, `aap_validate_certs`
- **Authenticator:** `aap_saml_authenticator_name`, `aap_saml_authenticator_order`, `aap_saml_enabled`, `aap_saml_create_objects`, `aap_saml_remove_users`
- **URLs:** `aap_saml_authenticator_slug` (used in `aap_saml_callback_url` for IdP redirect configuration), `aap_saml_callback_url` (derived from `aap_hostname` and slug by default)
- **IdP:** `saml_idp_entity_id`, `saml_idp_url`, `saml_idp_x509_cert`
- **Assertion attributes:** `saml_idp_attr_email`, `saml_idp_attr_first_name`, `saml_idp_attr_last_name`, `saml_idp_attr_username`, `saml_idp_attr_user_permanent_id` (must match your IdP claim or friendly names)
- **SP metadata:** `saml_sp_entity_id`, `saml_sp_extra`, `saml_sp_public_cert`, `saml_org_info`, `saml_support_contact`, `saml_technical_contact`
- **Built payload:** `aap_saml_configuration` assembles the above; **`SP_PRIVATE_KEY`** is injected from the vault at runtime (see below)
- **Allow map:** `aap_saml_map_allow_name`, `aap_saml_map_allow_order`, `aap_saml_map_allow_revoke`, `aap_saml_map_allow_triggers`

For an existing authenticator, copy **`slug`** from `GET /api/gateway/v1/authenticators/<id>/` so callback URLs stay consistent with what the IdP expects.

### Secrets for SAML

1. Copy the example files:

   ```bash
   cp vars/vault.yml.example vars/vault.yml
   cp vars/saml_config.yml.example vars/saml_config.yml
   ```

2. Edit **`vars/vault.yml`** and set:

   - **`vault_aap_username`** and **`vault_aap_password`**, **or** **`vault_aap_token`** (OAuth token for the gateway API)
   - **`vault_saml_sp_private_key`** — PEM for the service provider private key that pairs with **`saml_sp_public_cert`** in `vars/saml_config.yml`. The gateway stores this encrypted; use your original key material, not an export from the API.

3. Encrypt before committing:

   ```bash
   ansible-vault encrypt vars/vault.yml
   ```

4. Run:

   ```bash
   ansible-playbook playbooks/configure_aap_gateway_saml.yml --ask-vault-pass
   # or: --vault-password-file ~/.vault_pass
   ```

The authenticator task uses `no_log` to reduce accidental exposure of configuration payloads. Optional: **`--check`** if your installed `ansible.platform` version supports check mode for these modules.

Extra variables (`-e`) override values from `vars/vault.yml` and `vars/saml_config.yml` when you need a one-off.

## LDAP (platform gateway)

Non-secret LDAP settings live in **`vars/ldap_config.yml`** (copy from **`vars/ldap_config.yml.example`**). The playbook **`playbooks/configure_aap_gateway_ldap.yml`** ensures an LDAP authenticator and a superuser **authenticator map** driven by **`ldap_superuser_group_cn`** and the other variables documented in that example file.

Put **`vault_ldap_bind_password`** in **`vars/vault.yml`** when using LDAP. Run:

```bash
ansible-playbook playbooks/configure_aap_gateway_ldap.yml --ask-vault-pass
```

Extend the playbooks with additional `ansible.platform.authenticator_map` tasks if you need organization, team, or role maps beyond the examples.

## LDAPS and custom CAs

If you use LDAPS and the directory presents a certificate that the gateway does not trust, add the CA to the gateway trust store as described in the product documentation (for example the LDAPS certificate procedures in the same [access management guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/access_management_and_authentication/gw-configure-authentication)). That step is separate from this playbook.
