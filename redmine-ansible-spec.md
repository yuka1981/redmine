# Redmine on Rocky Linux – Ansible 規格書

## 0. 目標與前提

### 目標

用 Ansible 自動完成一台 Redmine 節點：

- OS：Rocky Linux 10（或相容 RHEL）
- Ruby：**3.4.7**，安裝在 `/opt/ruby/3.4.7`，**不產生 ruby doc**
- Redmine：**6.1.0**，安裝在 `/opt/redmine/6.1.0`
- DB：PostgreSQL 14，本機或 DB server（預先決定），建立：
  - 使用者：`redmine`
  - 密碼：`QCT@Admin`
  - 資料庫：`redmine`
- Web front：nginx + Puma（Unix socket）
- Service account：`redmine`（專職跑 Redmine / Puma）
- SELinux：Enforcing，正確標記 socket 目錄並設定必要 booleans
- systemd：`redmine-puma.service` 管理 Puma

---

## 1. Ansible 專案結構建議

大致結構：

```text
playbooks/
  site.yml
  group_vars/
    all.yml
roles/
  common/          # 基礎工具、套件、firewalld 等
  ruby/            # 編譯 /opt/ruby/3.4.7
  postgres/        # 安裝並設定 PostgreSQL 14
  redmine/         # 下載、解壓、bundler、database.yml
  puma/            # puma.rb & systemd service
  nginx/           # nginx 安裝 & redmine vhost
  selinux/         # SELinux context & booleans
```

### `site.yml` 範例

```yaml
- hosts: redmine_servers
  become: yes

  vars:
    ruby_version: '3.4.7'
    ruby_prefix_base: '/opt/ruby'
    ruby_prefix: '{{ ruby_prefix_base }}/{{ ruby_version }}'

    redmine_version: '6.1.0'
    redmine_root_base: '/opt/redmine'
    redmine_root: '{{ redmine_root_base }}/{{ redmine_version }}'
    redmine_user: 'redmine'
    redmine_group: 'redmine'

    redmine_db_name: 'redmine'
    redmine_db_user: 'redmine'
    redmine_db_password: 'QCT@Admin'
    redmine_db_host: '127.0.0.1'
    redmine_db_port: 5432

  roles:
    - role: common
    - role: ruby
    - role: postgres
    - role: redmine
    - role: puma
    - role: nginx
    - role: selinux
```

---

## 2. `common` role：基本環境

### 需求

- 安裝編譯 Ruby、Redmine 需要的工具與 library
- 建立 `/opt` 相關目錄

### Tasks 大綱

```yaml
# roles/common/tasks/main.yml
- name: Install base packages
  ansible.builtin.dnf:
    name:
      - gcc
      - gcc-c++
      - make
      - autoconf
      - bison
      - openssl-devel
      - libffi-devel
      - readline-devel
      - zlib-devel
      - gdbm-devel
      - ncurses-devel
      - libyaml-devel
      - tar
      - bzip2
      - curl
      - git
      - ImageMagick
      - ImageMagick-devel
    state: present

- name: Ensure /opt directories exist
  ansible.builtin.file:
    path: '{{ item }}'
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - '/opt/ruby'
    - '/opt/redmine'
```

---

## 3. `ruby` role：從 source 編譯 Ruby 3.4.7（不產 doc）

### 需求

- 下載 Ruby 3.4.7 source
- 編譯到 `/opt/ruby/3.4.7`
- `./configure` 關掉 doc 安裝
- 安裝 bundler

### Tasks 大綱

```yaml
# roles/ruby/tasks/main.yml
- name: Download Ruby source
  ansible.builtin.get_url:
    url: 'https://cache.ruby-lang.org/pub/ruby/3.4/ruby-{{ ruby_version }}.tar.gz'
    dest: '/usr/local/src/ruby-{{ ruby_version }}.tar.gz'
    mode: '0644'

- name: Extract Ruby source
  ansible.builtin.unarchive:
    src: '/usr/local/src/ruby-{{ ruby_version }}.tar.gz'
    dest: '/usr/local/src'
    remote_src: yes
    creates: '/usr/local/src/ruby-{{ ruby_version }}'

- name: Configure Ruby
  ansible.builtin.command:
    cmd: './configure --prefix={{ ruby_prefix }} --disable-install-doc --enable-shared'
    chdir: '/usr/local/src/ruby-{{ ruby_version }}'
  args:
    creates: '{{ ruby_prefix }}/bin/ruby'

- name: Compile Ruby
  ansible.builtin.command:
    cmd: 'make -j {{ ansible_facts.processor_vcpus | default(2) }}'
    chdir: '/usr/local/src/ruby-{{ ruby_version }}'
  args:
    creates: '/usr/local/src/ruby-{{ ruby_version }}/ruby'

- name: Install Ruby
  ansible.builtin.command:
    cmd: 'make install'
    chdir: '/usr/local/src/ruby-{{ ruby_version }}'
  args:
    creates: '{{ ruby_prefix }}/bin/ruby'

- name: Install bundler gem
  ansible.builtin.command:
    cmd: '{{ ruby_prefix }}/bin/gem install bundler'
  args:
    creates: '{{ ruby_prefix }}/bin/bundle'
```

---

## 4. `postgres` role：PostgreSQL 14 + redmine DB/user

### 需求

- 安裝 PostgreSQL 14
- 啟動並開機自動
- 建立 `redmine` 資料庫與使用者 `redmine` / `QCT@Admin`

### Tasks 大綱

```yaml
# roles/postgres/tasks/main.yml
- name: Install PostgreSQL 14 server & client
  ansible.builtin.dnf:
    name:
      - postgresql14-server
      - postgresql14
      - postgresql14-contrib
    state: present

- name: Initialize PostgreSQL 14 database
  ansible.builtin.command:
    cmd: '/usr/pgsql-14/bin/postgresql-14-setup initdb'
  args:
    creates: '/var/lib/pgsql/14/data/PG_VERSION'

- name: Ensure PostgreSQL 14 is enabled and started
  ansible.builtin.service:
    name: postgresql-14
    state: started
    enabled: yes

- name: Install psycopg2 (required by Ansible postgres modules)
  ansible.builtin.dnf:
    name: python3-psycopg2
    state: present

- name: Ensure redmine database user exists
  community.postgresql.postgresql_user:
    name: '{{ redmine_db_user }}'
    password: '{{ redmine_db_password }}'
    db: '{{ redmine_db_name }}'
    role_attr_flags: 'LOGIN'
  become_user: postgres

- name: Ensure redmine database exists
  community.postgresql.postgresql_db:
    name: '{{ redmine_db_name }}'
    owner: '{{ redmine_db_user }}'
  become_user: postgres
```

---

## 5. `redmine` role：下載 Redmine 6.1.0 + bundler + database.yml

### 需求

- 下載 Redmine 6.1.0 tarball
- 解壓到 `/opt/redmine/6.1.0`
- 建立 service account `redmine`
- 使用 bundler 安裝 gem → `vendor/bundle`
- 產生 `config/database.yml`（PostgreSQL 使用你指定的帳密）

### 建立 service account

```yaml
# roles/redmine/tasks/main.yml
- name: Create redmine group
  ansible.builtin.group:
    name: '{{ redmine_group }}'
    system: yes

- name: Create redmine user
  ansible.builtin.user:
    name: '{{ redmine_user }}'
    group: '{{ redmine_group }}'
    home: '{{ redmine_root_base }}'
    create_home: yes
    shell: '/usr/sbin/nologin'
    system: yes
    comment: 'Redmine service account'
```

### 下載與解壓 Redmine

```yaml
- name: Download Redmine {{ redmine_version }}
  ansible.builtin.get_url:
    url: 'https://www.redmine.org/releases/redmine-{{ redmine_version }}.tar.gz'
    dest: '/usr/local/src/redmine-{{ redmine_version }}.tar.gz'
    mode: '0644'

- name: Extract Redmine
  ansible.builtin.unarchive:
    src: '/usr/local/src/redmine-{{ redmine_version }}.tar.gz'
    dest: '{{ redmine_root }}'
    remote_src: yes
    extra_opts: ['--strip-components=1']
    creates: '{{ redmine_root }}/Gemfile'

- name: Ensure Redmine directories exist
  ansible.builtin.file:
    path: '{{ item }}'
    state: directory
    owner: '{{ redmine_user }}'
    group: '{{ redmine_group }}'
    mode: '0750'
  loop:
    - '{{ redmine_root }}'
    - '{{ redmine_root }}/tmp'
    - '{{ redmine_root }}/tmp/pids'
    - '{{ redmine_root }}/tmp/sockets'
    - '{{ redmine_root }}/log'
    - '{{ redmine_root }}/files'
```

### `config/database.yml`

`roles/redmine/templates/database.yml.j2`：

```yaml
production:
  adapter: postgresql
  database: { { redmine_db_name } }
  host: { { redmine_db_host } }
  port: { { redmine_db_port } }
  username: { { redmine_db_user } }
  password: '{{ redmine_db_password }}'
  encoding: utf8
  pool: 5
```

Task：

```yaml
- name: Deploy database.yml
  ansible.builtin.template:
    src: database.yml.j2
    dest: '{{ redmine_root }}/config/database.yml'
    owner: '{{ redmine_user }}'
    group: '{{ redmine_group }}'
    mode: '0640'
```

### bundler 安裝 Redmine gems

```yaml
- name: Configure bundle path for Redmine
  ansible.builtin.command:
    cmd: "{{ ruby_prefix }}/bin/bundle config set --local path 'vendor/bundle'"
    chdir: '{{ redmine_root }}'
  become_user: '{{ redmine_user }}'

- name: Bundle install for Redmine
  ansible.builtin.command:
    cmd: '{{ ruby_prefix }}/bin/bundle install --without development test'
    chdir: '{{ redmine_root }}'
  become_user: '{{ redmine_user }}'
```

---

## 6. `puma` role：Puma 設定檔 + systemd service

### 需求

- `config/puma.rb`：放在 `{{ redmine_root }}/config/puma.rb`
- 使用 Unix socket：`{{ redmine_root }}/tmp/sockets/puma.sock`
- systemd service：`redmine-puma.service`，用 `{{ ruby_prefix }}/bin/bundle exec puma -C config/puma.rb`

### `puma.rb` template

`roles/puma/templates/puma.rb.j2`：

```ruby
app_dir    = "{{ redmine_root }}"
directory app_dir

max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5).to_i
min_threads_count = ENV.fetch("RAILS_MIN_THREADS", max_threads_count).to_i
threads min_threads_count, max_threads_count

environment ENV.fetch("RAILS_ENV") { "production" }

workers ENV.fetch("WEB_CONCURRENCY", 2)

bind "unix://{{ redmine_root }}/tmp/sockets/puma.sock"

pidfile    "{{ redmine_root }}/tmp/pids/puma.pid"
state_path "{{ redmine_root }}/tmp/pids/puma.state"

stdout_redirect "{{ redmine_root }}/log/puma.stdout.log",
                "{{ redmine_root }}/log/puma.stderr.log",
                true

plugin :tmp_restart
```

Task：

```yaml
# roles/puma/tasks/main.yml
- name: Deploy puma config
  ansible.builtin.template:
    src: puma.rb.j2
    dest: '{{ redmine_root }}/config/puma.rb'
    owner: '{{ redmine_user }}'
    group: '{{ redmine_group }}'
    mode: '0640'
```

### systemd service template

`roles/puma/templates/redmine-puma.service.j2`：

```ini
[Unit]
Description=Redmine Puma Application Server
After=network.target

[Service]
Type=simple
User={{ redmine_user }}
Group={{ redmine_group }}
WorkingDirectory={{ redmine_root }}

Environment=RAILS_ENV=production
Environment=PATH={{ ruby_prefix }}/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
UMask=0007

ExecStart={{ ruby_prefix }}/bin/bundle exec puma -C config/puma.rb
ExecReload={{ ruby_prefix }}/bin/bundle exec pumactl -S tmp/pids/puma.state phased-restart

Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=redmine-puma

[Install]
WantedBy=multi-user.target
```

Tasks：

```yaml
- name: Install redmine-puma systemd service
  ansible.builtin.template:
    src: redmine-puma.service.j2
    dest: /etc/systemd/system/redmine-puma.service
    mode: '0644'

- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: yes

- name: Enable and start redmine-puma
  ansible.builtin.service:
    name: redmine-puma
    state: started
    enabled: yes
```

---

## 7. `nginx` role：安裝 + Redmine vhost

### 需求

- 安裝 nginx
- upstream 使用 Unix socket：`{{ redmine_root }}/tmp/sockets/puma.sock`
- 先提供 HTTP，之後再加 HTTPS / SSL Labs A+

### vhost template

`roles/nginx/templates/redmine.conf.j2`：

```nginx
upstream redmine_puma {
    server unix:{{ redmine_root }}/tmp/sockets/puma.sock fail_timeout=0;
}

server {
    listen 80;
    server_name {{ inventory_hostname }};

    root {{ redmine_root }}/public;
    access_log /var/log/nginx/redmine_access.log;
    error_log  /var/log/nginx/redmine_error.log;

    client_max_body_size 20m;

    location ~ ^/(assets|plugin_assets|themes|favicon.ico) {
        expires max;
        add_header Cache-Control public;
        try_files $uri @redmine;
    }

    location / {
        try_files $uri @redmine;
    }

    location @redmine {
        proxy_pass http://redmine_puma;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Tasks：

```yaml
# roles/nginx/tasks/main.yml
- name: Install nginx
  ansible.builtin.dnf:
    name: nginx
    state: present

- name: Deploy redmine nginx conf
  ansible.builtin.template:
    src: redmine.conf.j2
    dest: /etc/nginx/conf.d/redmine.conf
    mode: '0644'

- name: Test nginx configuration
  ansible.builtin.command: nginx -t
  register: nginx_test
  changed_when: false

- name: Fail if nginx config test failed
  ansible.builtin.fail:
    msg: 'nginx -t failed: {{ nginx_test.stderr }}'
  when: nginx_test.rc != 0

- name: Enable and restart nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
    enabled: yes
```

---

## 8. `selinux` role：讓 Nginx 能碰 Puma socket

### 需求

- SELinux Enforcing
- `{{ redmine_root }}/tmp/sockets` 標成 `httpd_var_run_t`
- 必要時開 `httpd_can_network_connect`

### Tasks 大綱

```yaml
# roles/selinux/tasks/main.yml

- name: Ensure SELinux is enforcing
  ansible.posix.selinux:
    policy: targeted
    state: enforcing

- name: Label Redmine Puma sockets directory as httpd_var_run_t
  community.general.sefcontext:
    target: '{{ redmine_root }}/tmp/sockets(/.*)?'
    setype: httpd_var_run_t
    state: present

- name: Apply SELinux context to sockets directory
  ansible.builtin.command:
    cmd: 'restorecon -Rv {{ redmine_root }}/tmp/sockets'
  args:
    warn: false

- name: Allow httpd to make outbound network connections (optional)
  ansible.posix.seboolean:
    name: httpd_can_network_connect
    state: true
    persistent: true
```
