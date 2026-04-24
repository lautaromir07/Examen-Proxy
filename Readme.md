# Infraestructura Web con Balanceo de Carga - Proyecto Lautaro

## Diagrama de la Arquitectura

```text
                               +-------------------+
                               |                   |
                               |  Usuario / Web    |
                               |                   |
                               +---------+---------+
                                         |
                                         | Puerto 80
                                         v
+-----------------------------------------------------------------------------------+
| Red de Docker (frontend) / Host                                                   |
|                                                                                   |
|                              +-------------------+                                |
|                              |                   |                                |
|                              |  Proxy Inverso    |   (Nginx - proxy_lautaro)      |
|                              |  (proxy)          |                                |
|                              |                   |                                |
|                              +---------+---------+                                |
+----------------------------------------|------------------------------------------+
                                         |
                                         | Balanceo de Carga (Round Robin)
                                         v
+-----------------------------------------------------------------------------------+
| Red de Docker Interna (backend) - Aislada del exterior                            |
|                                                                                   |
|           +-------------------+                 +-------------------+             |
|           |                   |                 |                   |             |
|           |   Servidor Web 1  |                 |   Servidor Web 2  |  (Nginx)    |
|           |   (web1_lautaro)  |                 |   (web2_lautaro)  |             |
|           |                   |                 |                   |             |
|           +---------+---------+                 +---------+---------+             |
|                     |                                     |                       |
|                     +-----------------+-------------------+                       |
|                                       |                                           |
|                                       v                                           |
|                             +-------------------+                                 |
|                             |                   |                                 |
|                             | Volumen Bind Local|   (./html)                      |
|                             | (Archivos Web)    |                                 |
|                             |                   |                                 |
|                             +-------------------+                                 |
+-----------------------------------------------------------------------------------+
```

## Decisiones de Diseño

* **Imágenes Oficiales:** Se ha utilizado exclusivamente la imagen oficial `nginx:alpine` de Docker Hub para garantizar seguridad, ligereza y rendimiento. Nginx es excelente tanto como servidor web estático como proxy inverso.
* **Proxy Inverso (Nginx):** Es el único punto de entrada a la infraestructura (expuesto en el puerto 80). Se encarga de recibir las peticiones de los usuarios y distribuirlas equitativamente (balanceo de carga) entre los servidores backend `web1` y `web2`.
* **Backends y Aislamiento de Red:** Los contenedores `web1` y `web2` están conectados a una red interna llamada `backend` (`internal: true`). Esta red no expone puertos al host, lo que significa que es imposible acceder a los servidores web directamente desde el exterior, incrementando la seguridad. Solo el proxy inverso (conectado a ambas redes) puede comunicarse con ellos.
* **Volumen Compartido:** Se utiliza un montaje de tipo "bind" (`./html`) montado en `/usr/share/nginx/html` en ambos servidores web. Esto garantiza que ambos servidores sirvan exactamente el mismo contenido (alta disponibilidad de los datos) de forma persistente y facilita la edición local.
* **Demostración del Balanceo:** Para comprobar visualmente el balanceo de carga, se han implementado dos estrategias:
    1. **Cabeceras HTTP:** El proxy inyecta cabeceras `X-Backend-Server` que cambian según la IP del backend que responda.
    2. **SSI (Server Side Includes):** Se ha habilitado SSI en Nginx (mediante el archivo `web/default.conf`) para que el archivo `index.html` imprima dinámicamente el nombre del servidor (hostname) que está procesando la petición. Al refrescar la página, verás cómo alterna entre el ID del contenedor de `web1_lautaro` y `web2_lautaro`.

## Comandos de Ejecución y Verificación

### 1. Levantar la Infraestructura
Para iniciar todo el entorno en segundo plano, abre una terminal en esta carpeta y ejecuta:
```bash
docker compose up -d
```

### 2. Verificar el Entorno
Abre tu navegador web y visita: [http://localhost](http://localhost)

**¿Cómo comprobar el balanceo de carga?**
1. En la página web verás un texto que dice **"Servidor Backend Actual: [ID_DEL_CONTENEDOR]"**.
2. **Refresca la página** (`F5` o `Ctrl+R`) varias veces. Verás que el ID cambia constantemente, demostrando que la petición está siendo atendida por servidores diferentes de manera alterna.
3. **Para verlo "en el modo de conexión":**
   - Haz clic derecho en la página y selecciona **Inspeccionar** (o presiona `F12`).
   - Ve a la pestaña **Red** (Network).
   - Asegúrate de tener marcada la opción "Deshabilitar caché" si está disponible.
   - Refresca la página.
   - Haz clic en la primera petición (`localhost`).
   - En la sección **Cabeceras de respuesta** (Response Headers), busca `X-Backend-Server`. Verás la IP interna del backend cambiando en cada recarga, y la cabecera `X-Proxy-By` indicando "Proxy Inverso - Lautaro".


<img width="1413" height="816" alt="0 2" src="https://github.com/user-attachments/assets/6449a567-7902-407a-8091-e7b0d311f1b3" />

<img width="1526" height="786" alt="0 3" src="https://github.com/user-attachments/assets/64caf94b-ec35-4383-bdaa-69ec6f504e7c" />



### 3. Detener la Infraestructura
Para apagar y eliminar los contenedores y sus redes (conservando tus archivos en el volumen):
```bash
docker compose down
```
