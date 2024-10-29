# DevOps TP-3 Report


## #Goals and Introduction#
The goal of this TP is to install and deploy an application automatically using Ansible. By utilizing Ansible's automation capabilities, we streamline the setup and configuration of infrastructure. In this TP, the focus is on using Ansible to:
- Install Docker on a remote server.
- Create and configure various Docker containers for database, network, application, and front-end components.
- Ensure continuous deployment and verify the successful setup of our application environment.

## #Inventory and Setup#
###  Inventory File
We created an inventory file at `ansible/inventories/setup.yml` to define the servers that Ansible will manage. This file groups hosts together and specifies SSH access information.
```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /path/to/your/private/key
  children:
    prod:
      hosts:
        my-server: 192.168.x.x  # Replace with actual IP or hostname
```

###  Hosts Configuration
In this setup, `ansible_user` is set to "admin", and the `ansible_ssh_private_key_file` points to the private SSH key needed for connecting to the remote server.

###  Command for Testing
To confirm connectivity to the server, we used the following command:
```bash
ansible all -i inventories/setup.yml -m ping
```
Upon successful configuration, this command should return "pong" as output, verifying that the server is accessible.

## #Facts and Variables#
### Facts
Ansible facts are automatically gathered system information variables that help with making decisions in playbooks. For example, facts can tell us the operating system, IP address, and available memory of a host.

### Usage
In our setup, we used facts to determine the OS distribution with the command:
```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution"
```
This was essential to ensure compatibility, particularly for the Docker installation process.

## #Playbooks#
### Initial Playbook

Our first playbook, `playbook.yml`, was a simple connectivity test:
```yaml
- hosts: all
  gather_facts: false
  become: true
  tasks:
    - name: Test connection
      ping:
```
This playbook verifies that Ansible can connect to all hosts defined in the inventory.

### Advanced Playbook
We created an advanced playbook to handle the Docker installation. This included the following tasks:
1. **Install Required Packages**: Install prerequisites like `apt-transport-https`, `curl`, `gnupg`, and more.
2. **Add Docker's GPG Key**: Add Docker's official GPG key.
3. **Set Up Docker Repository**: Set up the Docker stable repository for downloading Docker packages.
4. **Install Docker**: Install the Docker engine.
5. **Ensure Docker is Running**: Verify that Docker is active.

Example task structure:
```yaml
- name: Install Docker
  hosts: all
  become: true
  tasks:
    - name: Install prerequisites for Docker
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - apt-transport-https
        - ca-certificates
        - curl
        - gnupg
        - lsb-release
        - python3-venv

    - name: Add Docker's official GPG key
      apt_key:
        url: https://download.docker.com/linux/debian/gpg
        state: present

    - name: Set up Docker stable repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
        state: present
        update_cache: yes

    - name: Install Docker
      apt:
        name: docker-ce
        state: present

    - name: Ensure Docker is running
      service:
        name: docker
        state: started
```

## #Roles and Task Organization#
###  Role Structure
Using the command `ansible-galaxy init roles/docker`, we created a role for Docker installation. This structure allows us to separate concerns and modularize our Ansible code. The primary directories used within the role are:
- `tasks`: Contains the main tasks for Docker installation.
- `handlers`: Contains handlers to restart Docker if needed.

### Role Execution
In the main playbook which is playbook.yml, we referenced the Docker role as follows:
```yaml
- hosts: all
  gather_facts: true
  become: true
  roles:
    - install_docker
    - create_network
    - launch_database
    - launch_app
    - launch_proxy

```

## #Application Deployment#
###  Docker Containers
For the application, we set up several Docker containers:
1. **Database Container**:
    ```yaml
    - name: Run database container
      docker_container:
        name: my-db
        image: aminekr/tp-devops-database
        env:
          POSTGRES_USER: user
          POSTGRES_PASSWORD: pwd
          POSTGRES_DB: db
        networks:
          - my_network
        volumes:
          - db-volume:/var/lib/postgresql/data
        state: started
    ```

2. **Network Configuration**:
   We configured a Docker network to allow containers to communicate:
   ```yaml
   - name: Create Docker network
     docker_network:
       name: my_network
       state:present
   ```

3. **App and Proxy Containers**:
   Similar configuration was used to set up application and proxy containers, ensuring that each container joined `my-network` and could communicate internally.
   ```yaml
   - name: Run proxy container
     docker_container:
       name: httpd
       image: aminekr/tp-devops-http-server
       ports:
         - "80:80"
    networks:
      - name: my_network

    state: started
      ```
   ```yaml
    ---
    - name: Run application container
      docker_container:
         name: my-api
         image: aminekr/tp-devops-simple-api
         env:
            DATABASE_HOST: my-db
            DATABASE_PORT: "5342"
            POSTGRES_DB: db
            POSTGRES_USER: user
            POSTGRES_PASSWORD: pwd

         networks:
          - name: my_network

         state: started
   ```


## #Conclusion#
Through this TP-3, we gained a deep understanding of:
- How to use Ansible for infrastructure automation.
- Managing roles and modularizing tasks for reusability.
- Deploying applications using Docker and connecting services via networks.


