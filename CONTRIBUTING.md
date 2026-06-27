# Contributing

Thanks for taking a look! Issues and pull requests are welcome.

## Ground rules

- **Keep roles idempotent.** Re-running `site.yml` must not change a converged node.
- **Lint and syntax-check** before opening a PR:
  ```bash
  ansible-lint playbooks/ roles/
  ansible-playbook --syntax-check site.yml
  ```
- **Never commit secrets.** `credentials.json` and a real `inventory-home.yml` with
  passwords are git-ignored — keep it that way. Update the `*.example` files instead.
- **Test on one node first.** For upgrades, converge a single worker before the fleet.

## Good first contributions

- Support for another CNI (Calico/Cilium) behind the `kubernetes_cni` variable.
- Additional Molecule scenarios (e.g. converging the `containerd` role).
- An etcd snapshot/restore playbook (see "Disaster recovery scope" in the README).

## Dev environment

A VS Code dev container is provided under `.devcontainer/` with Ansible and the test
tooling preinstalled. Open the repository in VS Code and choose
**"Reopen in Container"** (Dev Containers extension) to get a ready-to-run shell.
