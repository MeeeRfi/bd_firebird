# Код
---
- name: Установка и настройка Firebird
  hosts: localhost
  become: true
  vars:
    firebird_password: "123"
    firebird_database_path: "/var/lib/firebird/3.0/data/database.fdb"
    firebird_user: "test"
    firebird_user_password: "qq"

  tasks:
    - name: Create user
      user:
        name: test
        password: qq
        state: present
        createhome: yes

    - name: Установка пакета Firebird сервера
      apt_rpm:
        name: firebird-server
        state: present

    - name: Установка утилит Firebird
      apt_rpm:
        name: firebird-utils
        state: present

    - name: Создание каталога для базы данных
      file:
        path: /var/lib/firebird/3.0/data
        state: directory
        owner: firebird
        group: firebird
        mode: '0755'

    - name: Создание временного SQL-файла
      copy:
        dest: /tmp/db.sql
        content:
          CREATE DATABASE '{{ firebird_database_path }}'
          USER '{{ firebird_user }}' PASSWORD '{{ firebird_user_password }}'
          PAGE_SIZE 8192 DEFAULT CHARACTER SET WIN1251;
        mode: '0644'

    - name: Выполнение создания БД
      command: isql-fb -user "{{ firebird_user }}" -password "{{ firebird_user_password }}" -i /tmp/db.sql
      register: db_creation
      changed_when: "'already exists' not in db_creation.stderr"
      ignore_errors: yes

    - name: Проверка существования БД после создания
      stat:
        path: "{{ firebird_database_path }}"
      register: db_check
      retries: 3
      delay: 2
      until: db_check.stat.exists

    - name: Настройка прав доступа (если БД существует)
      file:
        path: "{{ firebird_database_path }}"
        owner: test
        group: test
        mode: '0660'
      when: db_check.stat.exists
