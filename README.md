#registry
---
- name: Prepare download directory
  file:
    state: directory
    path: '{{ controller_mirror_registry_download_dir }}'

- name: Download the Mirror registry sums
  get_url:
    url: '{{ mirror_registry_download_url }}/sha256sum.txt.sig'
    dest: '{{ controller_mirror_registry_download_dir }}/sha256sum.txt.sig'

- name: Decrypt the Mirror registry sums
  shell: >
    gpg --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release &&
    gpg --output ./sha256sum.txt --decrypt sha256sum.txt.sig
  args:
    chdir: '{{ controller_mirror_registry_download_dir }}'
    creates: '{{ controller_mirror_registry_download_dir }}/sha256sum.txt'
  changed_when: false

- name: Download the Mirror Registry installer and signature
  get_url:
    url: '{{ mirror_registry_download_url }}/{{ item }}'
    dest: '{{ controller_mirror_registry_download_dir }}/{{ item }}'
    checksum: sha256:{{ lookup("file", controller_mirror_registry_download_dir + "/sha256sum.txt")|regex_search("[0-9a-f]+ +" + item|regex_escape)|split(" ")|list|first }}
  loop:
  - mirror-registry.tar.gz
  - mirror-registry.tar.gz.asc

- name: Validate the signature on the Mirror Registry installer
  shell: >
    gpg --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release &&
    gpg --verify mirror-registry.tar.gz.asc mirror-registry.tar.gz
  args:
    chdir: '{{ controller_mirror_registry_download_dir }}'
  changed_when: false
  
  - name: Ensure directories exist
  become: true
  file:
    path: '{{ item }}'
    state: directory
  loop:
  - /etc/quay/pki
  - /etc/quay/data

- name: Unpack the Mirror Registry installer on the registry node
  unarchive:
    src: '{{ controller_mirror_registry_download_dir }}/mirror-registry.tar.gz'
    dest: '{{ ansible_env["HOME"] }}'
    creates: '{{ ansible_env["HOME"] }}/mirror-registry'

- name: Create registry certificates
  command: openssl req -newkey rsa:3072 -nodes -sha256 -keyout /etc/quay/pki/ssl.key -x509 -days 365 -out /etc/quay/pki/ssl.cert -subj "/C=US/CN=${local_registry%:*}" -addext "subjectAltName=IP:127.0.0.1,DNS:localhost"

- name: Create Mirror Registry instance
  shell: >
    ./mirror-registry install
    --initUser '{{ registry_admin.username }}'
    --initPassword '{{ registry_admin.password }}'
    --quayHostname '{{ private_hostname }}:443'
    --quayRoot /etc/quay/data
    --sslCert /etc/quay/pki/ssl.cert
    --sslKey /etc/quay/pki/ssl.key
    --sslCheckSkip
  args:
    chdir: '{{ ansible_env["HOME"] }}'
    creates: /etc/systemd/system/quay-pod.service
  async: 1800  # wait up to half an hour for installation of the mirror registry
  poll: 5
  register: mirror_registry_install

- name: Ensure Mirror Registry is running
  become: yes
  systemd:
    name: '{{ item }}'
    state: started
    enabled: yes
  loop:
  - quay-pod
  - quay-postgres
  - quay-redis
  - quay-app

- name: Wait for Mirror Registry to report ready
  shell: >
    curl -k https://localhost/api/v1/superuser/registrystatus |
    python3 -c 'import json, sys; print(json.loads(sys.stdin.read())["status"])'
  changed_when: false
  register: quay_status
  until: '"ready" in quay_status.stdout'
  retries: 60
  delay: 5
