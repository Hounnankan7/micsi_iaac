# Déploiement Automatisé de VMs Proxmox avec Ansible, Netbox et Semaphore

## 📋 Description

Ce projet fournit un playbook Ansible permettant de déployer automatiquement des machines virtuelles sur Proxmox en utilisant Netbox comme source de vérité (SSOT - Single Source of Truth) et Semaphore comme orchestrateur d'exécution.

## 🏗️ Architecture

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│             │      │              │      │             │
│  Semaphore  │─────▶│   Ansible    │─────▶│   Proxmox   │
│             │      │   Playbook   │      │             │
└─────────────┘      └──────┬───────┘      └─────────────┘
                            │
                            │ Récupère les infos
                            ▼
                     ┌──────────────┐
                     │              │
                     │   Netbox     │
                     │              │
                     └──────────────┘
```

## ✨ Fonctionnalités

- 🔄 Récupération automatique des informations depuis Netbox
- 🖥️ Déploiement de VMs sur Proxmox
- 🎯 Orchestration via Semaphore UI
- 📊 Gestion centralisée de l'inventaire
- 🔐 Gestion sécurisée des credentials

## 📦 Prérequis

### Logiciels requis

- **Ansible** >= 2.9
- **Python** >= 3.8
- **Semaphore** >= 2.8
- **Proxmox VE** >= 6.0
- **Netbox** >= 3.0

### Collections Ansible requises

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install community.proxmox
ansible-galaxy collection install netbox.netbox
```

### Modules Python requis

```bash
pip install proxmoxer
pip install pynetbox
pip install requests
```

## 🚀 Installation

### 1. Cloner le repository

```bash
git clone <url-du-repo>
cd <nom-du-projet>
```

### 2. Configuration de Netbox

#### Créer un token API dans Netbox

1. Se connecter à Netbox
2. Aller dans **Admin** → **API Tokens**
3. Créer un nouveau token avec les permissions appropriées
4. Sauvegarder le token de manière sécurisée

#### Structure des données dans Netbox

Assurez-vous que vos VMs sont documentées dans Netbox avec :
- Nom de la VM
- Adresse IP
- CPU, RAM, Stockage
- Tags pour l'identification
- Site/Cluster de destination

### 3. Configuration de Proxmox

#### Créer un utilisateur API

```bash
pveum user add ansible@pve
pveum aclmod / -user ansible@pve -role PVEVMAdmin
pveum user token add ansible@pve ansible-token --privsep 0
```

### 4. Configuration de Semaphore

#### Ajouter le projet dans Semaphore

1. **Key Store** : Ajouter les credentials
   - Clé SSH pour Ansible
   - Token Netbox
   - Credentials Proxmox

2. **Repository** : Configurer le dépôt Git

3. **Environment** : Créer les variables d'environnement
   ```
   NETBOX_URL=https://netbox.example.com
   NETBOX_TOKEN=<votre-token>
   PROXMOX_HOST=proxmox.example.com
   PROXMOX_USER=ansible@pve
   PROXMOX_TOKEN_ID=ansible-token
   PROXMOX_TOKEN_SECRET=<votre-secret>
   ```

4. **Task Templates** : Créer un template pour le playbook

## 📁 Structure du projet

```
.
├── README.md
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── all.yml
├── playbooks/
│   ├── deploy_vms.yml
│   ├── gather_netbox_data.yml
│   └── create_proxmox_vm.yml
├── roles/
│   ├── netbox_inventory/
│   │   ├── tasks/
│   │   ├── defaults/
│   │   └── templates/
│   └── proxmox_vm/
│       ├── tasks/
│       ├── defaults/
│       └── templates/
├── vars/
│   └── vault.yml
└── requirements.yml
```

## ⚙️ Configuration

### ansible.cfg

```ini
[defaults]
inventory = ./inventory/hosts.yml
roles_path = ./roles
host_key_checking = False
retry_files_enabled = False

[inventory]
enable_plugins = netbox
```

### Variables principales

Créer le fichier `vars/vault.yml` (à chiffrer avec ansible-vault) :

```yaml
---
netbox_url: "https://netbox.example.com"
netbox_token: "votre-token-netbox"

proxmox_api_host: "proxmox.example.com"
proxmox_api_user: "ansible@pve"
proxmox_api_token_id: "ansible-token"
proxmox_api_token_secret: "votre-secret"
```

Chiffrer le fichier :

```bash
ansible-vault encrypt vars/vault.yml
```

## 🎮 Utilisation

### En ligne de commande

#### Déployer une VM

```bash
ansible-playbook playbooks/deploy_vms.yml \
  --extra-vars "vm_name=test-vm-01" \
  --ask-vault-pass
```

#### Déployer plusieurs VMs

```bash
ansible-playbook playbooks/deploy_vms.yml \
  --extra-vars "netbox_tag=production" \
  --ask-vault-pass
```

### Via Semaphore

1. Se connecter à l'interface Semaphore
2. Sélectionner le projet
3. Choisir le template de tâche
4. Renseigner les paramètres :
   - Nom de la VM ou tag Netbox
   - Node Proxmox cible
   - Autres paramètres spécifiques
5. Lancer l'exécution
6. Suivre les logs en temps réel

## 📝 Exemple de Playbook

```yaml
---
- name: Déployer des VMs depuis Netbox vers Proxmox
  hosts: localhost
  gather_facts: false
  vars_files:
    - ../vars/vault.yml
  
  tasks:
    - name: Récupérer les informations depuis Netbox
      netbox.netbox.nb_lookup:
        api_endpoint: "{{ netbox_url }}"
        token: "{{ netbox_token }}"
        plugin: virtualization.virtual_machines
        api_filter: "tag={{ netbox_tag }}"
      register: netbox_vms

    - name: Créer les VMs sur Proxmox
      community.general.proxmox_kvm:
        api_host: "{{ proxmox_api_host }}"
        api_user: "{{ proxmox_api_user }}"
        api_token_id: "{{ proxmox_api_token_id }}"
        api_token_secret: "{{ proxmox_api_token_secret }}"
        name: "{{ item.value.name }}"
        node: "{{ item.value.cluster.name }}"
        memory: "{{ item.value.memory }}"
        cores: "{{ item.value.vcpus }}"
        net:
          net0: "virtio,bridge=vmbr0"
        state: present
      loop: "{{ netbox_vms.value | dict2items }}"
```

## 🔒 Sécurité

### Bonnes pratiques

1. **Ne jamais commiter de secrets** dans le repository
2. **Utiliser ansible-vault** pour chiffrer les fichiers sensibles
3. **Limiter les permissions** des tokens API
4. **Activer l'audit logging** dans Semaphore
5. **Utiliser des connexions HTTPS** pour toutes les API

### Gestion des secrets dans Semaphore

- Utiliser le Key Store de Semaphore pour stocker les credentials
- Ne jamais afficher les secrets dans les logs
- Rotation régulière des tokens API

## 🐛 Dépannage

### Erreurs courantes

#### Connexion à Netbox échoue

```bash
# Vérifier la connectivité
curl -H "Authorization: Token <votre-token>" https://netbox.example.com/api/

# Vérifier les permissions du token
```

#### Connexion à Proxmox échoue

```bash
# Tester l'API Proxmox
curl -k https://proxmox.example.com:8006/api2/json/access/ticket \
  -d "username=ansible@pve" \
  -d "token_name=ansible-token" \
  -d "token_value=<votre-secret>"
```

#### Problèmes d'authentification Semaphore

- Vérifier que les variables d'environnement sont correctement définies
- Vérifier que le vault password est correct
- Vérifier les logs Semaphore : `/var/log/semaphore/`

## 📊 Monitoring et Logs

### Logs Ansible

```bash
# Activer le verbose mode
ansible-playbook playbooks/deploy_vms.yml -vvv
```

### Logs Semaphore

```bash
# Consulter les logs
journalctl -u semaphore -f
```

### Logs Proxmox

```bash
# Sur le serveur Proxmox
tail -f /var/log/pve/tasks/active
```

## 🤝 Contribution

Les contributions sont les bienvenues ! Pour contribuer :

1. Forker le projet
2. Créer une branche (`git checkout -b feature/amelioration`)
3. Commiter les changements (`git commit -m 'Ajout fonctionnalité'`)
4. Pousser la branche (`git push origin feature/amelioration`)
5. Ouvrir une Pull Request

## 📄 Licence

Ce projet est sous licence MIT. Voir le fichier `LICENSE` pour plus de détails.

## 👥 Auteurs

- Votre Nom - *Travail initial*

## 🙏 Remerciements

- Communauté Ansible
- Équipe Netbox
- Équipe Semaphore
- Équipe Proxmox

## 📚 Ressources supplémentaires

- [Documentation Ansible](https://docs.ansible.com/)
- [Documentation Netbox](https://netbox.readthedocs.io/)
- [Documentation Semaphore](https://docs.ansible-semaphore.com/)
- [Documentation Proxmox](https://pve.proxmox.com/pve-docs/)
- [Collection Ansible Netbox](https://docs.ansible.com/ansible/latest/collections/netbox/netbox/)
- [Module Proxmox KVM](https://docs.ansible.com/ansible/latest/collections/community/general/proxmox_kvm_module.html)

## 📞 Support

Pour toute question ou problème :
- Ouvrir une issue sur GitHub
- Contacter l'équipe à : support@example.com

---

**Note** : Ce README est un document vivant et sera mis à jour régulièrement avec de nouvelles fonctionnalités et améliorations.