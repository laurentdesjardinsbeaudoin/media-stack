# Media Stack Ansible Deployment

This Ansible playbook deploys the media stack to `/opt/media-stack` on your target server.

## Prerequisites

1. Install Ansible on your control machine:
   ```bash
   pip install ansible
   ```

2. Install the required Ansible collection:
   ```bash
   ansible-galaxy collection install community.docker
   ```

## Configuration

1. **Edit the inventory file** (`inventory/hosts.yml`):
   - Update `ansible_host` with your server's IP address
   - Update `ansible_user` with your SSH user
   - Adjust `ansible_python_interpreter` if needed

2. **Customize mount points** (`group_vars/media-stack.yml`):
   - Modify the base directories and individual mount points as needed
   - All volume paths are defined as variables for easy customization

## Usage

### Deploy the media stack:
```bash
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml
```

### Deploy with SSH password prompt:
```bash
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml --ask-pass --ask-become-pass
```

### Deploy with SSH key:
```bash
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml
```

### Check what would change (dry-run):
```bash
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml --check
```

## Structure

```
ansible/
├── inventory/
│   └── hosts.yml              # Inventory file with media-stack group
├── group_vars/
│   └── media-stack.yml        # Variables for mount points and paths
├── templates/
│   └── docker-compose.yml.j2  # Jinja2 template for docker-compose
├── deploy-media-stack.yml     # Main playbook
└── README.md                  # This file
```

## What the playbook does

1. Ensures Docker and docker-compose are installed
2. Creates the `/opt/media-stack` directory
3. Creates all required mount point directories
4. Deploys the docker-compose.yml file from template
5. Stops existing containers if the compose file changed
6. Starts all media stack containers
7. Pulls the latest images

## Customization

- **Ports**: Ports are static in the template and can be modified in `templates/docker-compose.yml.j2`
- **Mount points**: All mount points are variables in `group_vars/media-stack.yml`
- **Images**: Image versions can be changed in the template file
