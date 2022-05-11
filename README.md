# Ansible-image-and-container-creation-using-latest-git-repo


## Description
On this repository, we are going to discuss on how to create a docker image and container associated from latest git repo. Iam using a build server to create the image and a test server to create the container.

### Ansible Modules used
1. yum
2. pip
3. service
4. git
5. docker_image
6. docker_container
7. docker_login

### How to Use

~~~
git clone https://github.com/vyjith/Ansible-image-and-container-creation-using-latest-git-repo.git
cd Ansible-image-and-container-creation-using-latest-git-repo
ansible-playbook -i hosts main.yml
~~~

### Behind the code

### Ansible Hosts file

~~~
[build]

server1.example.com ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="yourkeyfile.pem"

[test]

server2.example.com  ansible_user="ec2-user" ansible_port=22 ansible_ssh_private_key_file="yourkeyfile.pem"

~~~

~~~
---

- name: "Building Docker Image and container from git"
  hosts: build
  become: true
  vars:
    packages:
      - git
      - pip
      - docker
    repo_url: "https://github.com/vyjith/devops-flask.git"
    repo_dir: "/home/ec2-user/flaskapp/"
    docker_user: "please_provide_docker_username_here"
    docker_password: "Please_provide_your_docker_password_here"
    image_name: "vyjith/flaskrep"

  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: started
        enabled: true

    - name: "Clonning the repo using {{ repo_url }}"
      git:
        repo: "{{repo_url}}"
        dest: "{{ repo_dir }}"
      register: git_status


    - name: "Logging into the docker-hub official"
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"



    - name: "Creating docker Image and push to hub"
      when: git_status.changed == true
      docker_image:
        source: build
        build:
          path: "{{ repo_dir }}"
          pull: yes
        name: "{{ image_name }}"
        tag: "{{ item }}"
        push: true
        force_tag: yes
        force_source: yes
      with_items:
        - "{{ git_status.after }}"
        - latest

    - name: "Logout from hub"
      when: git_status.changed == true
      docker_login:
        username: "{{ docker_user }}"
        password: "{{ docker_password }}"
        state: absent

    - name: "Removing docker image from build server"
      when: git_status.changed == true
      docker_image:

        name: "{{ image_name }}"
        tag: "{{ item }}"
        state: absent
      with_items:
        - "{{ git_status.after }}"
        - latest


- name: "creating container in build server"
  hosts: test
  become: true
  vars:
    packages:
      - pip
      - docker
    image_name: "vyjith/flaskrep"
  tasks:
    - name: " installing packages"
      yum:
        name: "{{ packages }}"
        state: present

    - name: "Installing docker client for python"
      pip:
        name: docker-py

    - name: "adding ec2-user to docker group"
      user:
        name: "ec2-user"
        groups:
          - docker
        append: true


    - name: "Restarting/enabling Docker"
      service:
        name: docker
        state: started
        enabled: true


    - name: "Pulling the docker Image from hub "
      docker_image:
        name: "{{image_name}}:latest"
        source: pull
        force_source: true
      register: image_status

    - name: " Creating the Container from pulled image"
      when: image_status.changed == true
      docker_container:
        name: flaskapp
        image: "{{ image_name }}:latest"
        recreate: yes
        pull: yes
        published_ports:
          - "80:5000"

~~~

Lets run the ansible playbook now,


~~~
ansible-playbook -i hosts main.yml

~~~

After running the playbook, we could see that container is created on the client server


~~~
#docker container ls

CONTAINER ID   IMAGE                     COMMAND            CREATED         STATUS         PORTS                  NAMES
c68a5ece9750   vyjith/flaskrep:latest   "python3 app.py"   3 minutes ago   Up 3 minutes   0.0.0.0:80->5000/tcp   flaskapp
~~~


If we run the same whithout any changes on docker image repository, it will skip the creation and build


![image]()


## Conclusion

In this tutorial, we discussed about creating docker image and container associated from latest git repo. If developer do any change on the dockerfile which placed on git. This will recreate the image and container without skipping.
