# arista-roas

At the end of the first interview I got after I finished my CCNA, one of the interviewers recommended that I get hip to [containerlab](https://containerlab.dev/). So this is my first, 'Hello World' clab project—a dirt-simple ROAS config with two VLANs.

## Architecture & Workflow

For now, I'm dropping into a shell to configure things manually, and doing a `clab save ./configs` when I'm done.

```text
.
├── README.md                  # Root index
├── .gitignore                 # Ignores local runtime directories (clab-*/)
  └── <lab-name>/
      ├── <lab-name>.clab.yml  # Containerlab topology definition
      └── configs/             # Persistent startup-configs (saved via `clab save`)
          ├── r1.cfg
          └── sw1.cfg

```

### Quickstart Workflow

1. **Deploy a Lab:**

```bash
sudo clab deploy -t <lab-folder>/<lab-name>.clab.yml

```

2. **Access & Configure:**
   Access device CLIs using `docker exec` or SSH to build and test configurations:

```bash
docker exec -it clab-<lab-name>-r1 Cli

```

3. **Persist Configuration to Git:**
   Once topology changes are complete, save the running configurations across all nodes back to the `configs/` directory in one command:

```bash
sudo clab save -t <lab-folder>/<lab-name>.clab.yml

```

4. **Tear Down:**

```bash
sudo clab destroy -t <lab-folder>/<lab-name>.clab.yml

```
