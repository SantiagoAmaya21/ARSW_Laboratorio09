### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

### Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonnaci.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

### Parte 1 - Escalabilidad vertical

1. Diríjase a el [Portal de Azure](https://portal.azure.com/) y a continuación cree una maquina virtual con las características básicas descritas en la imágen 1 y que corresponden a las siguientes:
    * Resource Group = SCALABILITY_LAB
    * Virtual machine name = VERTICAL-SCALABILITY
    * Image = Ubuntu Server 
    * Size = Standard B1ls
    * Username = scalability_lab
    * SSH publi key = Su llave ssh publica

![Imágen 1](images/part1/part1-vm-basic-config.png)

2. Para conectarse a la VM use el siguiente comando, donde las `x` las debe remplazar por la IP de su propia VM (Revise la sección "Connect" de la virtual machine creada para tener una guía más detallada).

    `ssh scalability_lab@xxx.xxx.xxx.xxx`

3. Instale node, para ello siga la sección *Installing Node.js and npm using NVM* que encontrará en este [enlace](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).
4. Para instalar la aplicación adjunta al Laboratorio, suba la carpeta `FibonacciApp` a un repositorio al cual tenga acceso y ejecute estos comandos dentro de la VM:

    `git clone <your_repo>`

    `cd <your_repo>/FibonacciApp`

    `npm install`

5. Para ejecutar la aplicación puede usar el comando `npm FibinacciApp.js`, sin embargo una vez pierda la conexión ssh la aplicación dejará de funcionar. Para evitar ese compartamiento usaremos *forever*. Ejecute los siguientes comando dentro de la VM.

    ` node FibonacciApp.js`

6. Antes de verificar si el endpoint funciona, en Azure vaya a la sección de *Networking* y cree una *Inbound port rule* tal como se muestra en la imágen. Para verificar que la aplicación funciona, use un browser y user el endpoint `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`. La respuesta debe ser `The answer is 8`.

![](images/part1/part1-vm-3000InboudRule.png)

7. La función que calcula en enésimo número de la secuencia de Fibonacci está muy mal construido y consume bastante CPU para obtener la respuesta. Usando la consola del Browser documente los tiempos de respuesta para dicho endpoint usando los siguintes valores:
    * 1000000
    * 1010000
    * 1020000
    * 1030000
    * 1040000
    * 1050000
    * 1060000
    * 1070000
    * 1080000
    * 1090000    

8. Dírijase ahora a Azure y verifique el consumo de CPU para la VM. (Los resultados pueden tardar 5 minutos en aparecer).

![Imágen 2](images/part1/part1-vm-cpu.png)

9. Ahora usaremos Postman para simular una carga concurrente a nuestro sistema. Siga estos pasos.
    * Instale newman con el comando `npm install newman -g`. Para conocer más de Newman consulte el siguiente [enlace](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
    * Diríjase hasta la ruta `FibonacciApp/postman` en una maquina diferente a la VM.
    * Para el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` cambie el valor del parámetro `VM1` para que coincida con la IP de su VM.
    * Ejecute el siguiente comando.

    ```
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
    ```

10. La cantidad de CPU consumida es bastante grande y un conjunto considerable de peticiones concurrentes pueden hacer fallar nuestro servicio. Para solucionarlo usaremos una estrategia de Escalamiento Vertical. En Azure diríjase a la sección *size* y a continuación seleccione el tamaño `B2ms`.

![Imágen 3](images/part1/part1-vm-resize.png)

11. Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9.
12. Evalue el escenario de calidad asociado al requerimiento no funcional de escalabilidad y concluya si usando este modelo de escalabilidad logramos cumplirlo.
13. Vuelva a dejar la VM en el tamaño inicial para evitar cobros adicionales.

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?

Cuando se crea una Máquina Virtual (VM) en Azure, el portal automáticamente despliega varios recursos asociados, normalmente entre 5 y 7 dependiendo de las configuraciones elegidas.
En este laboratorio, los recursos creados fueron:

Virtual Machine (VM)

Public IP Address

Network Interface (NIC)

Network Security Group (NSG)

Virtual Network (VNet)

Subnet (dentro de la VNet)

Storage Account / Disco administrado (OS Disk)

2. ¿Brevemente describa para qué sirve cada recurso?

| Recurso                          | Descripción                                                                                    |
| -------------------------------- | ---------------------------------------------------------------------------------------------- |
| **Virtual Machine (VM)**         | Es el servidor virtual donde se ejecuta el sistema operativo (Ubuntu) y la aplicación Node.js. |
| **Public IP Address**            | Permite acceder a la VM desde Internet (por SSH o por el puerto 3000).                         |
| **Network Interface (NIC)**      | Conecta la VM con la red virtual y asocia las configuraciones de red (IP, NSG, etc.).          |
| **Network Security Group (NSG)** | Define reglas de entrada y salida. Controla el tráfico que puede acceder a la VM.              |
| **Virtual Network (VNet)**       | Red privada que conecta los recursos dentro del grupo (por ejemplo, varias VMs).               |
| **Subnet**                       | Segmento dentro de la VNet que agrupa recursos con el mismo rango IP.                          |
| **Storage / OS Disk**            | Disco duro virtual donde se almacena el sistema operativo y los archivos de la aplicación.     |
| **Resource Group**               | Contenedor lógico para organizar y administrar todos los recursos relacionados.                |


3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?

Cuando ejecutamos npm FibonacciApp.js, el proceso se asocia directamente a la sesión SSH activa.
Cuando se cierra la conexión, el proceso también se termina, por eso la aplicación se “cae”.

La Solución es usar un gestor de procesos como forever o pm2, que mantiene la aplicación corriendo en segundo plano incluso si se cierra la sesión SSH.

El Inbound Port Rule permite que el tráfico externo pueda acceder al puerto 3000 (donde corre la app).
Sin esta regla, el Network Security Group bloquearía las peticiones HTTP y no podríamos acceder desde el navegador o Postman.

4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.

| n (Fibonacci) | Tiempo de respuesta (ms) |
| ------------- | ------------------------ |
| 1,000,000     | ...                      |
| 1,010,000     | ...                      |
| 1,020,000     | ...                      |
| 1,030,000     | ...                      |
| 1,040,000     | ...                      |
| 1,050,000     | ...                      |
| 1,060,000     | ...                      |
| 1,070,000     | ...                      |
| 1,080,000     | ...                      |
| 1,090,000     | ...                      |

esto se deve a que la función de Fibonacci está implementada de forma recursiva sin memoización ni iteración, por lo tanto su complejidad es exponencial O(2ⁿ).
Cada llamada genera dos más, haciendo que el tiempo crezca de manera desproporcionada.
Esto explica los tiempos tan altos y el uso intensivo del CPU.

5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.

La función recursiva genera una gran cantidad de operaciones matemáticas, lo que mantiene el procesador al 100%.
Node.js es de un solo hilo (single-threaded), por lo que no puede aprovechar varios núcleos. Toda la carga se concentra en un solo hilo, causando saturación de CPU.

6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.

Tiempos de ejecución: las peticiones tardan varios segundos debido a la alta carga de CPU.

Fallos: si hubo errores (timeout o conexión fallida), se deben a que el servidor no alcanzó a responder antes de que Postman cerrara la conexión.

Conclusión: el rendimiento es bajo porque la función no es eficiente y la VM (B1ls) tiene pocos recursos de CPU y memoria.

7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?

B1ls: Tamaño básico con 1 CPU virtual y 0.5 GB de RAM. Ideal para pruebas ligeras o tareas de baja demanda.

B2ms: Tiene 2 CPUs virtuales y 8 GB de RAM, por lo que maneja más procesos y mayor carga de trabajo.

B2ms no solo ofrece más capacidad, sino que también mejora la concurrencia y reduce el tiempo de espera por CPU, permitiendo que Node.js ejecute más operaciones sin bloquearse tanto.
Sin embargo, no soluciona la ineficiencia algorítmica: el problema de rendimiento principal sigue siendo la función de Fibonacci.

8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?

No del todo.
Aumentar el tamaño mejora temporalmente el rendimiento, pero no escala de forma eficiente.
La aplicación sigue usando un algoritmo ineficiente y Node.js no aprovecha múltiples núcleos.
Al cambiar el tamaño de la VM, la FibonacciApp se detiene temporalmente (reinicio) y debe volver a iniciarse.

9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?

Azure reinicia la máquina para aplicar el nuevo tamaño.

El hardware subyacente cambia (CPU, RAM, almacenamiento temporal).

Efectos negativos:

Tiempo de inactividad (downtime).

Riesgo de pérdida de datos si se usa almacenamiento temporal.

Costo mayor de operación.

No mejora la escalabilidad frente a múltiples usuarios concurrentes.

10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?

Sí, se observa una reducción en los tiempos y en la saturación de CPU.

Pero: la mejora es por capacidad, no por eficiencia.
El algoritmo sigue siendo costoso, por lo que el incremento no escala de manera lineal.
En cargas mayores, el rendimiento vuelve a degradarse.

11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

El sistema no mejora proporcionalmente.

Al tener más peticiones simultáneas, la CPU se satura más rápido y las respuestas se degradan.

Node.js maneja las conexiones concurrentes con un solo hilo de ejecución, por lo tanto no hay paralelismo real en el cálculo, solo en la gestión de peticiones.

## Procedimento P1:

![image.png](procedimiento/parte%201/image.png)
![image (1).png](procedimiento/parte%201/image%20%281%29.png)
![image.jpg](procedimiento/parte%201/image.jpg)
![image (1).jpg](procedimiento/parte%201/image%20%281%29.jpg)
![image (4).jpg](procedimiento/parte%201/image%20%284%29.jpg)
![image (2).jpg](procedimiento/parte%201/image%20%282%29.jpg)
![image (2).png](procedimiento/parte%201/image%20%282%29.png)
![image (3).jpg](procedimiento/parte%201/image%20%283%29.jpg)
![image (3).png](procedimiento/parte%201/image%20%283%29.png)
![image (4).png](procedimiento/parte%201/image%20%284%29.png)
![image (5).jpg](procedimiento/parte%201/image%20%285%29.jpg)
![image (5).png](procedimiento/parte%201/image%20%285%29.png)
![image (6).jpg](procedimiento/parte%201/image%20%286%29.jpg)
![image (7).jpg](procedimiento/parte%201/image%20%287%29.jpg)
![image (8).jpg](procedimiento/parte%201/image%20%288%29.jpg)
![image (10).jpg](procedimiento/parte%201/image%20%2810%29.jpg)
![image (11).jpg](procedimiento/parte%201/image%20%2811%29.jpg)
![image (12).jpg](procedimiento/parte%201/image%20%2812%29.jpg)
![image (13).jpg](procedimiento/parte%201/image%20%2813%29.jpg)
![image (14).jpg](procedimiento/parte%201/image%20%2814%29.jpg)
![image (15).jpg](procedimiento/parte%201/image%20%2815%29.jpg)
![image (17).jpg](procedimiento/parte%201/image%20%2817%29.jpg)
![image (9).jpg](procedimiento/parte%201/image%20%289%29.jpg)
![image (16).jpg](procedimiento/parte%201/image%20%2816%29.jpg)
![image (18).jpg](procedimiento/parte%201/image%20%2818%29.jpg)

### Parte 2 - Escalabilidad horizontal

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Probar el resultado final de nuestra infraestructura

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?, ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?
* ¿Cuál es el propósito del *Backend Pool*?
* ¿Cuál es el propósito del *Health Probe*?
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.
* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?
* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?
* ¿Cuál es el propósito del *Network Security Group*?
* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.




