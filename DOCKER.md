# Ansible in Docker

This setup allows you to run the Ansible playbook in a containerized environment without installing Ansible locally.

## Using Docker Compose (Recommended)

### Build the image:
```bash
docker compose -f docker-compose.ansible.yml build
```

### Interactive shell (run commands yourself):
```bash
docker compose -f docker-compose.ansible.yml run --rm ansible-shell
```

The container will automatically mount your SSH keys from `$HOME/.ssh` and set your username in the environment.

Once inside, you can run Ansible commands manually:
```bash
# Inside the container
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml --check
ansible-playbook -i inventory/hosts.yml deploy-media-stack.yml -v
```

### Run the playbook directly:
```bash
docker compose -f docker-compose.ansible.yml run --rm ansible
```

### Run with additional options:
```bash
# Dry-run (check mode)
docker compose -f docker-compose.ansible.yml run --rm ansible --check

# Verbose mode
docker compose -f docker-compose.ansible.yml run --rm ansible -v

# With custom variables
docker compose -f docker-compose.ansible.yml run --rm ansible -e "appdata_dir=/custom/path"

# Different playbook
docker compose -f docker-compose.ansible.yml run --rm ansible -i inventory/hosts.yml other-playbook.yml
```

## Using Docker directly

### Build the image:
```bash
docker build -t media-stack-ansible .
```

### Run the playbook:
```bash
docker run --rm \
  -v $(pwd)/ansible:/ansible \
  -v ~/.ssh:/root/.ssh:ro \
  --network host \
  media-stack-ansible
```

### With custom options:
```bash
docker run --rm \
  -v $(pwd)/ansible:/ansible \
  -v ~/.ssh:/root/.ssh:ro \
  --network host \
  media-stack-ansible -i inventory/hosts.yml deploy-media-stack.yml --check
```

## Notes

- The `ansible/` directory is mounted, so any changes you make locally are immediately available
- Your SSH keys from `~/.ssh` are mounted read-only for authentication
- Uses host network mode to allow Ansible to connect to target hosts
- No need to copy files into the image - everything is mounted at runtime
