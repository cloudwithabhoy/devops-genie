# Ansible — Basic Interview Questions

---

## 1. What is Ansible Vault?

> **Also asked as:** "How do you secure secrets in Ansible?"

**Ansible Vault** is a feature that allows you to keep sensitive data such as passwords or keys in encrypted files rather than as plaintext in playbooks or roles. These vault files can then be distributed or placed in source control.

**Key Features:**
- **Encryption:** Uses AES (Advanced Encryption Standard).
- **Granularity:** You can encrypt entire files or just specific variables (using `ansible-vault encrypt_string`).
- **Seamless Integration:** Playbooks can decrypt vault files on-the-fly during execution using a password file or prompt.

**Common Commands:**
```bash
# Encrypt an existing file
ansible-vault encrypt secrets.yml

# Edit an encrypted file
ansible-vault edit secrets.yml

# Run a playbook with a vault password prompt
ansible-playbook site.yml --ask-vault-pass
```

---

## 2. What are Ansible Roles?

> **Also asked as:** "How do you organize Ansible code for reuse?"

**Ansible Roles** are a way to group content (tasks, handlers, variables, templates, and files) into reusable units. Instead of having one massive playbook, you break logic into roles (e.g., `common`, `webserver`, `database`).

**Standard Role Structure:**
```text
roles/
  common/               # Role Name
    tasks/              # main.yml: The meat of the role
    handlers/           # main.yml: Service restarts, etc.
    vars/               # main.yml: High priority variables
    defaults/           # main.yml: Low priority variables (easily overridden)
    templates/          # *.j2: Jinja2 templates for configs
    files/              # Static files to be copied
    meta/               # main.yml: Role dependencies and author info
```

**Why use Roles?**
- **Readability:** Playbooks become simple "mappings" of hosts to roles.
- **Reusability:** Roles can be shared across different projects/playbooks.
- **Maintainability:** Changing a task in a role updates it everywhere that role is used.
