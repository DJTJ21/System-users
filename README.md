## Exercice guidé: Gestion des utilisateurs et de l’authentification

Dans cet exercice, vous allez créer plusieurs utilisateurs sur vos hôtes gérés et renseigner les clés SSH autorisées pour ces derniers.

**Résultats**

Vous devez pouvoir :

-   Créer un groupe d’utilisateurs.
    
-   Gérer les utilisateurs avec le module  `user`.
    
-   Renseigner les clés autorisées SSH à l’aide du module  `authorized_key`.
    
-   Modifier les fichiers  `sudoers`  et  `sshd_config`  à l’aide du module  `lineinfile`.
    
Pour ce TP notre machine de travail est `workstation` et notre serveur cible est `servera`


**Instructions**

Votre organisation nécessite que tous les hôtes disposent des mêmes utilisateurs locaux. Ces utilisateurs doivent appartenir au groupe d’utilisateurs  `webadmin`  qui a la capacité d’utiliser la commande  `sudo`  sans spécifier de mot de passe. De plus, les clés publiques SSH des utilisateurs doivent être distribuées dans l’environnement, et l’utilisateur  `root`  ne doit pas être autorisé à se connecter directement à l’aide de SSH.

Vous êtes chargé d’écrire un playbook pour vous assurer que les utilisateurs et le groupe d’utilisateurs sont présents sur l’hôte distant. Le playbook doit garantir que les utilisateurs peuvent se connecter à l’aide de la clé SSH autorisée, ainsi qu’utiliser  `sudo`  sans spécifier de mot de passe, et que l’utilisateur  `root`  ne peut pas se connecter directement à l’aide de SSH.

    
1.  Consultez le fichier de variable  `vars/users_vars.yml`  existant.
```yml
[romuald@workstation system-users]$ cat vars/users_vars.yml
users:
  - username: user1
    groups: webadmin
  - username: user2
    groups: webadmin
  - username: user3
    groups: webadmin
  - username: user4
    groups: webadmin
  - username: user5
    groups: webadmin
```
      
   Il utilise le nom de variable  `username`  pour définir le nom d’utilisateur correct et la variable  `groups`  pour définir des groupes supplémentaires auxquels l’utilisateur devrait appartenir.
    
2.  Commencez à écrire le playbook  `users.yml`. Définissez un seul play dans le playbook qui cible le groupe d’hôtes  `webservers`. Ajoutez une clause  `vars_files`  définissant l’emplacement du nom de fichier  `vars/users_vars.yml`  qui a été créé pour vous et qui contient tous les noms d’utilisateur requis pour cet exercice. Ajoutez la clause  `tasks`  au playbook.
    
    Utilisez un éditeur de texte pour créer le playbook  `users.yml`. Le playbook doit contenir les éléments suivants :
    
    ``` yml
    - name: Create multiple local users
      hosts: webservers
      vars_files:
        - vars/users_vars.yml
      tasks:
      ```
    
3.  Ajoutez les deux tâches au playbook.
    
    Utilisez le module  `group`  dans la première tâche pour créer le groupe d’utilisateurs  `webadmin`  sur l’hôte distant. Cette tâche crée le groupe  `webadmin`.
    
    Utilisez le module  `user`  dans la deuxième tâche pour créer les utilisateurs du fichier  `vars/users_vars.yml`.
    
    Exécutez le playbook  `users.yml`.
    
    1.  Ajoutez la première tâche au playbook. La première tâche contient les éléments suivants:
        ``` yml
          - name: Add webadmin group
            group:
              name: webadmin
              state: present
        ```
        
    2.  Ajoutez une deuxième tâche au playbook qui utilise le module  `user`  pour créer les utilisateurs. Ajoutez une clause  `loop: "{{ users }}"`  à la tâche pour parcourir le fichier de variable pour chaque nom d’utilisateur trouvé dans le fichier  `vars/users_vars.yml`. Comme  `name:`  pour les utilisateurs, utilisez le nom de variable  `item.username`. De cette façon, le fichier de variables peut contenir des informations supplémentaires pouvant s’avérer utiles pour créer les utilisateurs, telles que les groupes auxquels les utilisateurs doivent appartenir. La deuxième tâche contient les éléments suivants :
        ``` yml
          - name: Create user accounts
            user:
              name: "{{ item.username }}"
              groups: "{{ item.groups }}"
            loop: "{{ users }}"
        ```
    3.  Exécutez le playbook :
        
 ```yml
    [romuald@workstation system-users]$  ansible-playbook users.yml
        PLAY [Create multiple local users] *******************************************

TASK [Gathering Facts] *******************************************************
ok: [servera.lab.example.com]

TASK [Add webadmin group] ****************************************************
changed: [servera.lab.example.com]

TASK [Create user accounts] **************************************************
changed: [servera.lab.example.com] => (item={u'username': u'user1', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user2', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user3', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user4', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user5', u'groups': u'webadmin'})

PLAY RECAP *******************************************************************
servera.lab.example.com    : ok=3    changed=2    unreachable=0    failed=0 
```

4.  Ajoutez une troisième tâche qui utilise le module  `authorized_key`  pour s’assurer que les clés publiques SSH ont été correctement distribuées sur l’hôte distant. Dans le répertoire  `files`, chaque utilisateur possède un fichier de clé publique SSH unique. Le module parcourt la liste des utilisateurs, trouve la clé appropriée en utilisant la variable  `username`  et transmet la clé à l’hôte distant.
    
    La troisième tâche contient les éléments suivants :
    ```yml
      - name: Add authorized keys
        authorized_key:
          user: "{{ item.username }}"
          key: "{{ lookup('file', 'files/'+ item.username + '.key.pub') }}"
        loop: "{{ users }}"
    ```
5.  Ajoutez une quatrième tâche au play qui utilise le module  `copy`  pour modifier le fichier de configuration  `sudo`  et permettre aux membres du groupe  `webadmin`  d’utiliser  `sudo`  sans mot de passe sur l’hôte distant.
    
    La quatrième tâche apparaît comme suit :
    ```yml
      - name: Modify sudo config to allow webadmin users sudo without a password
        copy:
          content: "%webadmin ALL=(ALL) NOPASSWD: ALL"
          dest: /etc/sudoers.d/webadmin
          mode: 0440
    ```
6.  Ajoutez une cinquième tâche pour vous assurer que l’utilisateur  `root`  n’est pas autorisé à se connecter directement à l’aide de SSH. Utilisez  `notify: "Restart sshd"`  afin de déclencher un gestionnaire pour redémarrer SSH.
    
    La cinquième tâche se présente comme suit :
    ```yml
      - name: Disable root login via SSH
        lineinfile:
          dest: /etc/ssh/sshd_config
          regexp: "^PermitRootLogin"
          line: "PermitRootLogin no"
        notify: Restart sshd
    ```
7.  Dans la première ligne après l’emplacement du fichier de variable, ajoutez une nouvelle définition de gestionnaire. Nommez-la  `Restart sshd`.
    
    1.  Le gestionnaire doit être défini comme suit :
        ```yml
        _...output omitted..._
            - vars/users_vars.yml
          handlers:
          - name: Restart sshd
            service:
              name: sshd
              state: restarted
        ```
        Le playbook complet se présente maintenant comme suit :
        
        ```yml
        - name: Create multiple local users
          hosts: webservers
          vars_files:
            - vars/users_vars.yml
          handlers:
          - name: Restart sshd
            service:
              name: sshd
              state: restarted
        
          tasks:
        
          - name: Add webadmin group
            group:
              name: webadmin
              state: present
        
          - name: Create user accounts
            user:
              name: "{{ item.username }}"
              groups: "{{ item.groups }}"
            loop: "{{ users }}"
        
          - name: Add authorized keys
            authorized_key:
              user: "{{ item.username }}"
              key: "{{ lookup('file', 'files/'+ item.username + '.key.pub') }}"
            loop: "{{ users }}"
        
          - name: Modify sudo config to allow webadmin users sudo without a password
            copy:
              content: "%webadmin ALL=(ALL) NOPASSWD: ALL"
              dest: /etc/sudoers.d/webadmin
              mode: 0440
        
          - name: Disable root login via SSH
            lineinfile:
              dest: /etc/ssh/sshd_config
              regexp: "^PermitRootLogin"
              line: "PermitRootLogin no"
            notify: "Restart sshd"
        ```
    2.  Exécutez le playbook.
        
   ```yml
 [romuald@workstation system-users]$ ansible-playbook users.yml
 PLAY [Create multiple local users] *******************************************

TASK [Gathering Facts] *******************************************************
ok: [servera.lab.example.com]

TASK [Add webadmin group] ****************************************************
ok: [servera.lab.example.com]

TASK [Create user accounts] **************************************************
ok: [servera.lab.example.com] => (item={u'username': u'user1', u'groups': u'webadmin'})
ok: [servera.lab.example.com] => (item={u'username': u'user2', u'groups': u'webadmin'})
ok: [servera.lab.example.com] => (item={u'username': u'user3', u'groups': u'webadmin'})
ok: [servera.lab.example.com] => (item={u'username': u'user4', u'groups': u'webadmin'})
ok: [servera.lab.example.com] => (item={u'username': u'user5', u'groups': u'webadmin'})

TASK [Add authorized keys] ***************************************************
changed: [servera.lab.example.com] => (item={u'username': u'user1', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user2', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user3', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user4', u'groups': u'webadmin'})
changed: [servera.lab.example.com] => (item={u'username': u'user5', u'groups': u'webadmin'})

TASK [Modify sudo config to allow webadmin users sudo without a password] ***
changed: [servera.lab.example.com]

TASK [Disable root login via SSH] *******************************************
changed: [servera.lab.example.com]

RUNNING HANDLER [Restart sshd] **********************************************
changed: [servera.lab.example.com]

PLAY RECAP ******************************************************************
servera.lab.example.com    : ok=7    changed=4    unreachable=0    failed=0
 ```
        
8.  Comme l’utilisateur  `user1`, connectez-vous au serveur  `servera`  en utilisant SSH. Une fois connecté, utilisez la commande  `sudo su -`  pour prendre l’identité de l’utilisateur  `root`.
    
    1.  Utilisez SSH en tant qu’utilisateur  `user1`  et connectez-vous au serveur  `servera`.
   
      [romuald@workstation system-users]$ ssh user1@servera
       Activate the web console with: systemctl enable --now cockpit.socket
      [user1@servera ~]$
    ``` 
Basculez vers l’utilisateur `root`.
  ```yml 
   [user1@servera ~]$ 
   root@servera ~]#
  ```
        
   2. Déconnectez-vous de `servera`.   
3.  Essayez de vous connecter à  `servera`, en tant qu’utilisateur  `root`  directement. Cette étape doit échouer car la configuration démon SSH a été modifiée pour ne pas autoriser la connexion directe en tant qu’utilisateur  `root`.
    
  À partir de  `workstation`, utilisez SSH en tant que  `root`  pour vous connecter au serveur  `servera`.
        
  ```yml
[romuald@workstation system-users]$ ssh root@servera
root@servera's password: `redhat`
Permission denied, please try again.
root@servera's password:
```
   
