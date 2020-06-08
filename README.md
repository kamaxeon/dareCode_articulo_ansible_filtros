# Introducción 

Hola y bienvenido a este primer artículo donde hablaré un poco de ansible. Ansible es una herramienta de gestión de la configuración y orquestación desarrollado en python y que pertenece a RedHat.  

Es un artículo introductorio, pero se supone que sabes los conceptos de playbook y roles. Si no los sabes, en este artículo te dejo una sección de enlaces para aprender ansible, o si lo prefieres escríbemos algún comentario e intentamos ayudarte 😛 

# Objetivo 

Vamos a ver como los filtros de ansible nos pueden ayudar, teniendo nuestro repositorio de ansible bajo control y evitando repeticiones inncesarias. 

# Metodología 

Vamos a intentar definir una serie de precondiciones descritas los más parecido en lenguaje natural, de manera que sea muy descriptivo para nosotros y nos valga como punto de partida. Así, si se nos complica la cosa, siempre tendremos claro cuales son nuestros objetivos para evitar desviarnos de ellos. 

Además intenteremos hacerlo todo con pasitos pequeños (baby steps) para nos perder detalle.

# Punto de partida 

Cuando desarrollamos cualquier roles, tenemos que hacerlo de la forma más desacoplada posible de una futura integración. Así tenemos nuestro role puede evolucionar de manera independiente. 

Vamos a poner un ejemplo, para eso, nos vamos a basar en una persona bastante respetada dentro de la comunidad de ansible, su nombre es Jeff Geerling (https://twitter.com/geerlingguy) y el repositorio es cuestión es este https://github.com/geerlingguy/ansible-role-docker 

Entre sus variables, nos pide una lista de nombres de usuarios que serán añadidos dentro del grupo de docker para que puedan usar docker: 

 
```yml
docker_users:
  - user1
  - user2
```

En el contexto del role y su espacio de nombres, el nombre de la variable es perfecto. 

También, vamos a suponer que tenemos otro role con nombre “sudo” que y nos ofrecen otras dos variables donde podemos poner nuestros usuario que queremos que puedan usar sudo (con o sin contraseña): 


```yml
sudo_with_password_users:
  - user1
  - user2
sudo_without_password_users:
  - user1
  - user2
```
 

Por supuesto, tenemos un role o playbook que nos crea los usuarios de nuestros entornos, podemos tener algo como 


```yml
common_users: 
  - name: user1 
    state: present 
    group: group1  

  - name: user2 

  - name: user3 
    group: group2 
```
 

Para finalizar, tenemos el role de nuestra aplicación que nos va a jubilar del éxito, que necesita que le digamos el nombre del usuario que se usará para lanzarla. Ojo no es un lista, es un texto con el nombre del usuario, algo como 

```yml
app_username: user3 
```

## Aplicación en nuestro entorno 

Con todo esto de forma abstracta, queremos aplicar lo siguiente en nuestro caso: 

Usuarios a definir: 
* john
* peter
* anthony
* margaret
* july
* mary

Los usuarios que pueden usar docker son: 

* margaret
* july

Los usuarios que pueden usar sudo sin clave son: 

* john
* peter

Los usuarios que pueden usar sudo con clave son: 

* anthony

El usuario de nuestra aplicación es *mary*, y sólo puede existir uno. 


# Desarrollo 

## Definición de test en ansible 

Antes de hacer nada, y como vimos al principio vamos a definirmos un playbook sencillo que contenga nuestros tests, y que por supuesto fallará de forma estrepitosa. Pero lo que quiero es tener de una forma descriptiva y primitiva el resultado final, el resultado lo tienes en [01_playbook](01_playbook.yml)

```yml
---
- hosts: localhost
  connection: local
  gather_facts: no
  vars:
    dockers_users: []
    sudo_with_password_users: []
    sudo_without_password_users: []
    common_users: []
    app_username: ''

  tasks:
    - name: Checking common_users
      assert:
        that:
          - common_users is defined
          - common_users | length == 6
        fail_msg: Not all the users are included
      ignore_errors: yes

    - name: Checking docker users
      assert:
        that: 
          - docker_users is defined
          - docker_users | length == 2
          - '"margaret" in docker_users'
          - '"july" in docker_users'
        fail_msg: Margaret and July only should be in docker_users
      ignore_errors: yes

    - name: Checking sudo user without password
      assert:
        that: 
          - sudo_without_password_users is defined
          - sudo_without_password_users | length == 2
          - '"john" in sudo_without_password_users'
          - '"peter" in sudo_without_password_users'
        fail_msg: John and Peter only should be in sudo_without_password_users
      ignore_errors: yes

    - name: Checking sudo user with password
      assert:
        that: 
          - sudo_with_password_users is defined
          - sudo_with_password_users | length == 1
          - '"anthony" in sudo_with_password_users'
        fail_msg: Anthony only should be in sudo_with_password_users
      ignore_errors: yes

    - name: Checking that Mary is the app user in the hosts
      assert:
        that:
          - '"mary" == app_username'
        fail_msg: Mary should be the user for the app
      ignore_errors: yes
```

Si lo ejecutamos evidentemente fallaran todos los tests

```bash
$ ansible-playbook 00_test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ****************************************************************************************

TASK [Checking common_users] ****************************************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "common_users | length == 6",
    "changed": false,
    "evaluated_to": false,
    "msg": "Not all the users are included"
}
...ignoring

TASK [Checking docker users] ****************************************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "docker_users is defined",
    "changed": false,
    "evaluated_to": false,
    "msg": "Margaret and July only should be in docker_users"
}
...ignoring

TASK [Checking sudo user without password] **************************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "sudo_without_password_users | length == 2",
    "changed": false,
    "evaluated_to": false,
    "msg": "John and Peter only should be in sudo_without_password_users"
}
...ignoring

TASK [Checking sudo user with password] *****************************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "sudo_with_password_users | length == 1",
    "changed": false,
    "evaluated_to": false,
    "msg": "Anthony only should be in sudo_with_password_users"
}
...ignoring

TASK [Checking that Mary is the app user in the hosts] **************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "\"mary\" == app_username",
    "changed": false,
    "evaluated_to": false,
    "msg": "Mary should be the user for the app"
}
...ignoring

PLAY RECAP **********************************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=5 
```

Aunque parezca que es perder el tiempo, ya tenemos bastante avanzado.


## Primera iteración sin filtros

Ahora vamos a hacer que sólo funcione de una forma muy ruda y primitiva, funciona no será nuestra versión final. La versión es exactamente igual a la anterior, pero simplemente hemos rellenado las variables, ahora tienen esta pinta.

```yml
  vars:
    common_users:
      - name: margaret
      - name: july
      - name: john
      - name: peter
      - name: anthony
      - name: mary
    docker_users:
      - margaret
      - july
    sudo_without_password_users:
```

El fichero de este paso es [01_test.yml](01_test.yml)

### Conclusiones 

Hay muchísima de repetición de nombres, y cara a futuro es poco mantenible y propenso a errores, podría por ejemplo escribir *margaret* en common_users y *margarit* en docker_users, empezando a tener esos errores que tan poco nos gusta. Además hay veces que son evidentes y otras no.

### Ejecución completa de este paso

```bash
ansible-playbook 01_test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ****************************************************************************************

TASK [Checking common_users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking docker users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user without password] **************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user with password] *****************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking that Mary is the app user in the hosts] **************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

## Segunda iteración

Ahora que tengo funcionando me planteo si toda esta parte que hace referencia a usuarios, no podría en un único sitio y después ir explotando esa información. Como quiero dar pasos pequeños iré de los más fáciles a los más complicados. En este caso, lo más fácil es crearme una estructura users copiando como user y después referenciándolo. Aunque suene un poco raro, verás que es muy fácil.

```yml
    users:
      - name: margaret
      - name: july
      - name: john
      - name: peter
      - name: anthony
      - name: mary
    
    common_users: "{{ users }}"
```

Como siempre si quieres ver el fichero entero lo tienes en [02_test.yml](02_test.yml)

### Conclusiones

Aunque este pequeño cambio, parece que no es nada, ya hemos creado nuestra estructura que contendrá al resto. Ya vamos por buen camino.

### Ejecución completa de este paso

```bash
$ ansible-playbook 02_test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ****************************************************************************************

TASK [Checking common_users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking docker users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user without password] **************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user with password] *****************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking that Mary is the app user in the hosts] **************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Tercera iteración.

Aquí es realmente donde empezamos a usar filtros, vamos a intentar extraer los usuarios de nuestra estructura de usuarios *users*, para eso vamos a añadir la siguiente lógica. Añado un nuestro campo opcional, que sea con clave nombre y valor booleano. En caso de no poner el campo, se sobreentiende que no será un usuario que pueda ejecutar docker.

Este paso vamos a hacerlo con filtros, y podemos hacerlo de dos formas distintas con [jmespath](https://jmespath.org/) (recuerda instalarlo) o sin él. Vamos a ver como quedaría primero la versión sin jmespath.



```yml
  vars:
    users:
      - name: margaret
        docker: true
      - name: july
        docker: true
      - name: john
        docker: false
      - name: peter
      - name: anthony
      - name: mary
    
    docker_users:  "{{ users |
                       selectattr('docker', 'defined') |
                       selectattr('docker', 'equalto', True) |
                       map(attribute='name') |
                       list }}"
```

El filtro básicamente de docker_users hace lo siguiente:

* Recoge el listado de users, comprueba de cada elemento que tenga la clave docker definida, y después comprueba que sea igual a true. Posteriormente sólo se queda con el valor de la clave '*name*' de cada elemento y nos devuelve una lista.

Veamos ahora el ejemplo como jmespath

```yml
  vars:
    users:
      - name: margaret
        docker: true
      - name: july
        docker: true
      - name: john
        docker: false
      - name: peter
      - name: anthony
      - name: mary

    jmespath_docker_users: "{{ users | json_query(\"[?docker]\") | map(attribute='name') | list }}"
```

En este caso, vemos como ha quedado más claro con jmespath, pero lo importante es que el resultado sea el mismo. Como puede observar lo que extraigo con jmespath le puesto el prefijo jmespath, por lo que ahora debo modificar los tests para que comprueben ámbas variables.

El test quedaría de la siguiente manera

```yml
    - name: Checking docker users
      assert:
        that: 
          - docker_users is defined
          - docker_users | length == 2
          - '"margaret" in docker_users'
          - '"july" in docker_users'
          - docker_users == jmespath_docker_users
        fail_msg: Margaret and July only should be in docker_users
```

De esta manera compruebo que ámbas contienen lo mismo en el último aserto.

El fichero entero de esta iteración es [03_test.yml](03_test.yml)

### Conclusiones

Ya parece que esto empieza a coger forma, y estamos organizando las cosas.

### Ejecución completa de este paso

```bash
$ ansible-playbook 04_test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ****************************************************************************************

TASK [Checking common_users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking docker users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user without password] **************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user with password] *****************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking that Mary is the app user in the hosts] **************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

```

## Cuarta iteración

Aquí vamos a ejecutar un momento el módulo de debug de ansible, porque creo que estamos metiendo datos que no nos interesan o no eran los iniciales en nuestra variables common_users. Lo que ejecutaríamos tendríamos que escribir sería algo así en el playbook.

```yml
    - name: Show common_users var
      debug:
        msg: "common_users: {{ common_users }}"
```

Ejecutamos nuestro playbook y vemos lo siguiente:

```bash
TASK [Show common_users var] ****************************************************************************
TASK [Show common_users var] ****************************************************************************
ok: [localhost] => {
    "msg": "common_users: [{'name': 'margaret', 'docker': True}, {'name': 'july', 'docker': True}, {'name': 'john', 'docker': False}, {'name': 'peter'}, {'name': 'anthony'}, {'name': 'mary'}]"
}
....
```

Estamos pasando todo los campos de users, cuando en nuestro ejemplo, sólo queremos pasar '*name*', igual es tu caso da igual, pero vamos a corregir *common_users* para que sólo muestre el atributo *name* como estaba originalmente. Nuestra variable quedaría así.

```yml
    common_users: "{{ users | map(attribute='name') | list }}"
```

Volvemos a ejecutar y vemos que ya tenemos nuestro problema arreglado.

```bash
TASK [Show common_users var] ****************************************************************************
ok: [localhost] => {
    "msg": "common_users: ['margaret', 'july', 'john', 'peter', 'anthony', 'mary']"
}
```

Dejo como ejercicio del lector, hacer un test que compruebe esto. El fichero de este paso es [04_test.yml](04_test.yml)

### Conclusión

Aunque los tests nos ayudam, ahi que tener cuidado, porque igual no tenemos todos los casos contemplados.


## Quinta ejecución

Ya nos va quedando menos, hemos aprendido a hacer un filtro para variables de tipo booleano, y nos hemos dado cuenta de algún fallo o feature, que lo hemos ajustado sobre la marcha.

En esta ocasión, vamos a atacar el tema del sudo, en este caso, en este caso, vamos a suponer la siguiente lógica.

* Creamos una clave en nuestra estructura de usuarios llamada sudo, que puede tener los siguientes valores *with_password*, *without_password*. Si contiene otra cosa o no tiene la clave, daremos por sentado que no usará sudo.

Como en la iteración tercera, vamos a hacerlo con y sin jmespath.

Vamos a ver como quedaría la versión sin jmespath, para los dos grupos que debemos crear

```yml
    sudo_without_password_users: "{{ users |
                                     selectattr('sudo', 'defined') |
                                     selectattr('sudo', 'equalto', 'without_password') |
                                     map(attribute='name') |
                                     list }}"
    sudo_with_password_users:    "{{ users |
                                     selectattr('sudo', 'defined') |
                                     selectattr('sudo', 'equalto', 'with_password') |
                                     map(attribute='name') |
                                     list }}"
```

Ahora veamos el ejemplo con jmespath

```yml
    jmespath_sudo_without_password_users: "{{ users |
                                              json_query(\"[?sudo=='without_password']\") |
                                              map(attribute='name') |
                                              list }}"

    jmespath_sudo_with_password_users:    "{{ users |
                                              json_query(\"[?sudo=='with_password']\") |
                                              map(attribute='name') |
                                              list }}"
```

Ahora sólo nos quedaría modificar un poco el test, para que compruebe que ámbas variables (con y sin jmespath son iguales)

```yml
    - name: Checking sudo user without password
      assert:
        that: 
          - sudo_without_password_users is defined
          - sudo_without_password_users | length == 2
          - '"john" in sudo_without_password_users'
          - '"peter" in sudo_without_password_users'
          - sudo_without_password_users == jmespath_sudo_without_password_users
        fail_msg: John and Peter only should be in sudo_without_password_users

    - name: Checking sudo user with password
      assert:
        that: 
          - sudo_with_password_users is defined
          - sudo_with_password_users | length == 1
          - '"anthony" in sudo_with_password_users'
          - sudo_with_password_users == jmespath_sudo_with_password_users
        fail_msg: Anthony only should be in sudo_with_password_users
```

El fichero de este paso lo tienes en [05_test.yml](05_test.yml)

### Conclusiones
Cada vez esto está siendo más organizado, y casi no tenemos que repetir nada. Ánimo que ya casi hemos acabado.

### Ejecución completa de este paso

```bash
ansible-playbook 05_test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ****************************************************************************************

TASK [Show common_users var] ****************************************************************************
ok: [localhost] => {
    "msg": "common_users: ['margaret', 'july', 'john', 'peter', 'anthony', 'mary']"
}

TASK [Checking common_users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking docker users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user without password] **************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user with password] *****************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking that Mary is the app user in the hosts] **************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```


### Sexta ejecución

En esta última ejecución, vamos a poner a abordar el caso de usuario que se usará para arrancar nuestra aplicación, ahora lo que queremos es un valor, no una lista de elementos, para eso vamos a usar un filtro que no acabará en lista, y cogeremos el primer valor positivo que nos encontremos. Vamos a ver las versiones de nuestro filtro, como siempre, con y sin jmespath.

Hemos decidido que la clave para este nuestro valor será app_user y tendrá un valor booleano a verdadero.

Sin jmespath

```yml
    app_username: "{{ users |
                      selectattr('app_user', 'defined') |
                      selectattr('app_user', 'equalto', True) |
                      map(attribute='name') |
                      first }}"
```

Con jmespath

```yml
    jmespath_app_username: "{{ users | json_query(\"[?app_user]\") | map(attribute='name') | first }}"
```

Por último como siempre, modificamos los tests para ver que ámbas variables son iguales

```yml
    - name: Checking that Mary is the app user in the hosts
      assert:
        that:
          - '"mary" == app_username'
          - app_username == jmespath_app_username
        fail_msg: Mary should be the user for the app
```

Lanzamos nuestros test, y todo sale en verde, pero que pasaría si por error le pongo también a john el atributo de *app_user* a true, es decir, tendría esto en *users*

```yml
  vars:
    users:
      - name: margaret
        docker: true
      - name: july
        docker: true
      - name: john
        docker: false
        app_user: true
        sudo: without_password
      - name: peter
        sudo: without_password
      - name: anthony
        sudo: with_password
      - name: mary
        app_user: true
```

Vuelvo a lanzar y ...

```bash
$ ansible-playbook 06_test.yml
...

TASK [Checking that Mary is the app user in the hosts] **************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "\"mary\" == app_username",
    "changed": false,
    "evaluated_to": false,
    "msg": "Mary should be the user for the app"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=6    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
```

Nos podemos volver locos buscando donde está el fallo, porque mary está marcada como app_user, para corregir este caso, podemos poner una prueba más en los tests que nos compruebe que sólo un usuario tiene esa propiedad como *true*, vamos a ver como lo haríamos.

```yml
    - name: Checking that Mary is the app user in the hosts
      vars:
        number_of_users_with_app_username: "{{ users |
                                               json_query(\"[?app_user]\") | 
                                               map(attribute='name') |
                                               list |
                                               length }}"
      assert:
        that:
          - number_of_users_with_app_username == '1'
          - '"mary" == app_username'
          - app_username == jmespath_app_username
        fail_msg: Mary should be the user for the app
```

Ahora cuando el test falla lo hace porque ha encontrado varios usuarios con la propiedad, lo que nos da más vistas y es más fácil de depurar

```bash
$ ansible-playbook 06_test.yml
...
TASK [Checking that Mary is the app user in the hosts] **************************************************
fatal: [localhost]: FAILED! => {
    "assertion": "number_of_users_with_app_username == '1'",
    "changed": false,
    "evaluated_to": false,
    "msg": "Mary should be the user for the app"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=5    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0 
```

Corregimos el fallo y todo sale perfectamente.

El fichero de este paso es [06_test.yml](06_test.yml)

### Conclusiones

Esta última ejecución nos ha dejado nuestras variables bastante limpias y podrían terminar de la siguiente manera.

```yml
  vars:
    users:
      - name: margaret
        docker: true
      - name: july
        docker: true
      - name: john
        docker: false
        sudo: without_password
      - name: peter
        sudo: without_password
      - name: anthony
        sudo: with_password
      - name: mary
        app_user: true
    
    common_users: "{{ users | map(attribute='name') | list }}"

    docker_users: "{{ users | json_query(\"[?docker]\") | map(attribute='name') | list }}"
    
    sudo_without_password_users: "{{ users | 
                                     json_query(\"[?sudo=='without_password']\") |
                                     map(attribute='name') |
                                     list }}"

    sudo_with_password_users: "{{ users |
                                  json_query(\"[?sudo=='with_password']\") |
                                  map(attribute='name') |
                                  list }}"

    app_username: "{{ users | json_query(\"[?app_user]\") | map(attribute='name') | first }}"
```

Esto es mucho más sostenible y gestionable que nuestro punto de partida.

### Ejecución completa de este paso

```bash
ansible-playbook 06_test.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
does not match 'all'

PLAY [localhost] ****************************************************************************************

TASK [Show common_users var] ****************************************************************************
ok: [localhost] => {
    "msg": "common_users: ['margaret', 'july', 'john', 'peter', 'anthony', 'mary']"
}

TASK [Checking common_users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking docker users] ****************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user without password] **************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking sudo user with password] *****************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [Checking that Mary is the app user in the hosts] **************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY RECAP **********************************************************************************************
localhost                  : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
# Recursos para aprender ansible

# Recursos para este artículo

* https://stackoverflow.com/questions/57858362/ansible-how-to-use-selectattr-with-yaml-of-different-keys
* https://stackoverflow.com/questions/41581273/search-dictionary-values-in-ansible
* https://gist.github.com/halberom/3659c98073efcabd91ed1dec3ad63fa3
* https://stackoverflow.com/questions/54334751/ansible-selectattr-map-list-returns-empty-result
* https://oznetnerd.com/2017/04/18/jinja2-selectattr-filter/