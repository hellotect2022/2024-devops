# ansible 구축

태그: ANSIBLE, DEVOPS, LINUX
AI 요약: 이 문서는 Ansible을 사용하여 여러 사이트에 동시 배포할 수 있는 방법을 설명합니다. Ansible 설치, SSH 공개키 설정, Ansible 실행, Ansible의 기본 구조에 대한 내용이 포함되어 있습니다.

### HISTORY

코비젼 이라는 회사에 있을당시 서비스를 배포할 경우 여러 사이트에 동시 배포 해야하는 경우가 생김 (버전 호환성을 동기화 시킨다는 명목으로…..) 근데 그걸 전부 나한테 떠넘김 (ㅅㅂㅅㄲㄷ) 그래서 아래와 같이 한개의 배포 서버를 두고 여러 사이트에 동시 배포가 가능하게 하기 위한 ANSIBLE 을 설치 함 (Jenkins 는 메모리 사양 부족으로 인해 본 서비스에서는 생략했음)

### 그림

![Untitled](ansible%20%E1%84%80%E1%85%AE%E1%84%8E%E1%85%AE%E1%86%A8%202b87f4f2dc5e43b4a9c87c5ab0495a71/Untitled.png)

### Ansible 설치

```bash
sudo apt install ansible -y
```

### ssh 공개키 설정하고 매니저 노드에 전달

```bash
ssh-keygen	#RSA키 생성, 명령어 입력 후 나오는 것들 그냥 전부 엔터 누르자
ssh-copy-id study@10.0.2.21	#host1의 study 유저에게 제어 노드의 공개키 전달
ssh-copy-id study@10.0.2.22	#host2의 study 유저에게 제어 노드의 공개키 전달

ssh study@10.0.2.21	#명령어를 입력하고 host1의 study 유저 비밀번호를 입력하면 성공적으로 ssh 접속이 이루어진다.

ansible -m ping -i <inventory_file> all
```

### Ansible 실행

```bash
ansible-playbook -i <inventory_file> <playbook.yaml>
```

## Ansible 의 기본 개념

앤서블의 3요소 

 - [inventory](https://www.notion.so/ansible-2b87f4f2dc5e43b4a9c87c5ab0495a71?pvs=21) : 어디서

 - [playbook](https://www.notion.so/ansible-2b87f4f2dc5e43b4a9c87c5ab0495a71?pvs=21) : 무엇을

 - module : 어떻게 수행

### Inventory (어디서) : 대상 서버 목록을 정의

### **Ansible 기본 구조: Inventory**

 - Ansible이 관리할 호스트와 각 호스트에 접속하기 위한 정보를 정의. 

 - 각 호스트 그룹과 그에 속하는 서버들이 나열, 

 - 서버에 접속하기 위한 사용자 이름, 암호, 권한 상승 방법 등이 포함.

```bash
[**keydb_server**]
192.168.11.174 **ansible_ssh_user**=ssh_account **ansible_become**=true **ansible_become_password**=ssh_password
```

- SSH 사용자: **`ansible_ssh_user`**
- 권한 상승: 사용 (**`ansible_become=true`**)
- 권한 상승 암호: **`ansible_become_password`**

### Ansible > group_var를 통한 변수 관리

```bash
[**keydb_server**]
192.168.11.174 **ansible_ssh_user**=ssh_account **ansible_become**=true **ansible_become_password**=ssh_password

**[keydb_server:vars]**
root_dir=/prd

```

### Playbook.yaml (무엇을) : 대상 서버에서 할 작업들을 정의

```yaml
- name: Play Name
  hosts: enter_server, deploy_server, eidc_server
  gather_facts: true 
  tasks:
    - name: Set facts
      include_tasks: task/step-setfact.yml

    - name: Backup files
      include_tasks: task/step-backup.yml
      when: false

```

**`host`** :  대상 서버 (서버이름, all)

**`gather_facts`** :  대상 서버의 시스템 정보를 수집

**`tasks`** : 플레이북에서 실행되는 작업 목록(task) 

### Module (어떻게) : Playbook 에서 task 가 어떻게 수행될지 정의

Core Modules (핵심모듈) : 기본적으로 원격 호스트에서 실행됨

- **Shell** (쉘 명령을 실행) → **ansible.builtin.shell**
    
    ```yaml
    - name: Backup server.war
      **ansible.builtin.shell**: |
         cp "{{ root_dir }}/app/ui/chat/server.war" "{{ root_dir }}/app/backup/{{ backup_file }}/server.war" &&
         echo "Backup server.war -> success"
      ignore_errors: yes
      when: deploy_war_files.server_war_exist
    ```
    
- **Copy** (파일을 원격 시스템에 복사)
    
    ```yaml
    - name: Transfer legacy.war
      **ansible.builtin.copy**:
        src: /home/appservice/ansible/legacy.war
        dest: "{{ root_dir }}/app/ui/legacy/legacy.war"
      when: deploy_war_files.legacy_war_exist
    ```
    
- **File** (파일 및 디렉토리 관리 작업 수행)
    
    ```yaml
    - name: Create backup directory
      **ansible.builtin.file**:
        path: "{{root_dir}}/app/backup/{{ backup_file }}"
        state: directory
      when: not backup_dir.stat.exists
    
    ```
    
    **`state`** : directory, absent, file, link(soft link 생성, src 랑 같이 사용), touch
    
- **Service** (서비스 관리 작업 수행)
    
    ```yaml
    - name: Start Tomcat-Chat
      **service**:
        name: tomcat-chat
        state: started
      when: deploy_war_files.server_war_exist
    ```
    
    **`state`** : started, stopped, restarted
    
- **State** (파일의 상태 검사)
    
    파일 존재 여부, 파일의 크기, 소유자, 권한, 체크
    
    ```yaml
    - name: Check exists backup directory
      **ansible.builtin.stat**:
        path: "{{root_dir}}/app/backup/{{ backup_file }}"
      register: backup_dir
    ```
    
- **local_action (로컬 호스트 : Ansible 컨트롤러에서 실행)**
    
    로컬에서 실행되는 stat
    
    ```yaml
    - name: server.war file exist check
      **local_action**:
        module: **ansible.builtin.stat**
        path: /home/appservice/ansible/server.war
      register: server_war_file
    ```
    
    원격 호스트에서 실행되는 stat
    
    ```yaml
    - name: Check exists backup directory
      **ansible.builtin.stat**:
        path: "{{root_dir}}/app/backup/{{ backup_file }}"
      register: backup_dir
    ```
    
- **set_fact** (변수설정)
    
    ```yaml
    - name: Set custom fact for server war file existence
      **set_fact**:
        deploy_war_files:
          server_war_exist: "{{ server_war_file.stat.exists | default(false) }}"
          restful_war_exist: "{{ restful_war_file.stat.exists | default(false) }}"
          manager_war_exist: "{{ manager_war_file.stat.exists | default(false) }}"
          legacy_war_exist: "{{ legacy_war_file.stat.exists | default(false) }}"
        backup_file: "{{ ansible_date_time.date | regex_replace('[-:]','') }}"
    ```