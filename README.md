# Sistema Distribuido de 3 Capas con Caché — Modelo Cuadrático

**Materia:** Seminario De Actualización II 2026  
**Alumno:** *Fabricio Augusto*  
**Fecha:** 01/06/2026

---

## Descripción del Proyecto

Implementación de un sistema distribuido en 4 máquinas virtuales Linux (Ubuntu 22.04) corriendo en VirtualBox. El sistema expone un modelo matemático cuadrático `y = 2x² + 5x + 3` que puede ser consultado desde un navegador web. Las consultas se guardan en un caché Redis para evitar recalcular resultados ya conocidos.

| VMs | Rol | IP NAT | IP Interna | Puerto |
|---|---|---|---|---|
| VM2-Backend | API REST Node.js/Express | 10.0.2.16 | 192.168.56.11 | 3000 |
| VM3-Frontend | Servidor Web Apache | 10.0.2.17 | 192.168.56.12 | 8080 |
| VM1-BaseDatos | *(trabajo anterior — no se usa acá)* | 10.0.2.15 | 192.168.56.10 | — |
| VM4-Cache | Redis | 10.0.2.18 | 192.168.56.13 | 6379 |

---

## ¿Qué es un Caché?

Un **caché** es un almacenamiento temporal de resultados ya calculados. La idea es que si alguien ya consultó un valor, no tiene sentido calcularlo de nuevo: se guarda la respuesta y la próxima vez se devuelve directamente desde el caché, mucho más rápido.

**Redis** es una base de datos en memoria (RAM) que se usa como caché. Guarda datos en formato clave-valor y es extremadamente rápida porque no necesita acceder al disco.

En este sistema Redis guarda hasta **10 consultas recientes**. Si se supera ese límite, la consulta más antigua se elimina automáticamente.

---

## Flujo de Datos

### Primera consulta (X nunca fue consultado):
```
Navegador (Host)
      │
      │ localhost:8080
      ▼
VM3 - Apache (Frontend)
      │
      │ localhost:3000  (port forwarding → 10.0.2.16:3000)
      ▼
VM2 - Node.js/Express (Backend)
      │
      │ Consulta Redis: ¿Tenés el resultado para X?
      ▼
VM4 - Redis → No tengo ese valor
      │
      │ Calcula y = 2x² + 5x + 3
      │ Guarda resultado en Redis
      ▼
Responde: { x, y, fuente: "backend" }
```

### Segunda consulta (X ya fue consultado antes):
```
Navegador (Host)
      │
      ▼
VM3 - Apache (Frontend)
      │
      ▼
VM2 - Node.js/Express (Backend)
      │
      │ Consulta Redis: ¿Tenés el resultado para X?
      ▼
VM4 - Redis → Sí, acá está
      │
Responde: { x, y, fuente: "cache" }
```

---

## Infraestructura y Red

### Configuración de Red en VirtualBox

Cada VM tiene 2 adaptadores de red:

- **Adaptador 1 (NAT):** Para acceso SSH desde el host y salida a internet
- **Adaptador 2 (Red Interna - intnet):** Para comunicación entre VMs

### Reglas de Port Forwarding (NAT)

**VM3 - Frontend:**

| Nombre | Puerto Host | Puerto Invitado |
|---|---|---|
| SSH_FRONT | 2217 | 22 |
| WEB_SISTEMA | 8080 | 8080 |

**VM2 - Backend:**

| Nombre | Puerto Host | Puerto Invitado |
|---|---|---|
| SSH_BACK | 2216 | 22 |
| API_SERVICE | 3454 | 3454 |
| API_CUADRATICA | 3000 | 3000 |

**VM4 - Cache:**

| Nombre | Puerto Host | Puerto Invitado |
|---|---|---|
| SSH_CACHE | 2218 | 22 |
| REDIS | 6379 | 6379 |

---

## Paso a Paso de Implementación

### FASE 1 — Creación de las VMs

1. Descargar Ubuntu Server 22.04 LTS desde https://releases.ubuntu.com
2. En VirtualBox crear nueva VM con estas especificaciones:
   - RAM: 2048 MB
   - Disco: 10 GB (VDI, reservado dinámicamente)
   - Tipo: Linux 64-bit
3. En Almacenamiento montar el ISO descargado
4. Instalar Ubuntu Server con las siguientes opciones:
   - Idioma: English
   - Teclado: English (US)
   - Tipo: Ubuntu Server (no minimized)
   - Tildar **Install OpenSSH server**
   - Usuario: fabricio1404
5. La VM4-Cache se puede crear clonando una VM existente:
   - Clic derecho → Clonar
   - Política MAC: Generar nuevas direcciones MAC
   - Tipo: Clonado completo

---

### FASE 2 — Configuración de Red

En cada VM editar el archivo de red:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**VM3 - Frontend:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      addresses:
        - 192.168.56.10/24
```

**VM2 - Backend:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 10.0.2.16/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      addresses:
        - 192.168.56.11/24
```

**VM4 - Cache:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 10.0.2.18/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      addresses:
        - 192.168.56.13/24
```

Aplicar cambios en cada VM:

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

Verificar conectividad desde VM2 hacia VM4:

```bash
ping -c 3 192.168.56.13
```

Resultado esperado: `0% packet loss`

---

### FASE 3 — VM4: Servidor de Caché (Redis)

#### ¿Qué se instala y por qué?

Redis es el servidor de caché. Se instala en una VM separada para que el backend pueda consultarlo de forma independiente, siguiendo la arquitectura de 3 capas distribuidas.

#### Instalación de Redis

```bash
sudo apt update
sudo apt install redis-server -y
```

#### Habilitar acceso remoto

Por defecto Redis solo acepta conexiones locales. Hay que cambiar dos configuraciones:

```bash
sudo nano /etc/redis/redis.conf
```

**Cambio 1** — Buscar la línea `bind 127.0.0.1 -::1` y cambiarla por:
```
bind 0.0.0.0
```

**Cambio 2** — Buscar la línea `protected-mode yes` y cambiarla por:
```
protected-mode no
```

Guardar y reiniciar Redis:

```bash
sudo systemctl restart redis-server
sudo systemctl status redis-server
```

Resultado esperado: `active (running)` y `redis-server 0.0.0.0:6379`

---

### FASE 4 — VM2: Servidor Backend (Node.js/Express)

#### ¿Qué hace el backend?

El backend es el núcleo del sistema. Recibe consultas con un valor de X, verifica si el resultado ya está en Redis y si no lo está, lo calcula usando el modelo cuadrático `y = 2x² + 5x + 3`, lo guarda en Redis y lo devuelve.

#### Instalación de Node.js

```bash
sudo apt update
sudo apt install nodejs npm -y
node -v
npm -v
```

#### Crear el proyecto

```bash
mkdir ~/app-cuadratica
cd ~/app-cuadratica
npm init -y
```

#### Instalar dependencias

```bash
npm install express cors ioredis
```

- **express** → framework web para crear la API REST
- **cors** → permite que el frontend (otro origen) consuma la API
- **ioredis** → cliente para conectarse a Redis desde Node.js

#### Verificar instalación

```bash
node -e "require('express'); require('ioredis'); console.log('todo ok')"
```

#### Código de la API (`~/app-cuadratica/app.js`)

```js
const express = require('express');
const cors = require('cors');
const Redis = require('ioredis');

const app = express();
app.use(cors());
app.use(express.json());

// Conexión a Redis en VM4
const redis = new Redis({
    host: '192.168.56.13',
    port: 6379
});

const MAX_CACHE = 10;

// Modelo cuadrático: y = 2x² + 5x + 3
function calcular(x) {
    return 2 * x * x + 5 * x + 3;
}

app.get('/consultar/:x', async (req, res) => {
    const x = parseFloat(req.params.x);

    if (isNaN(x)) {
        return res.json({ error: 'El valor de X no es válido' });
    }

    const cacheKey = `x:${x}`;

    try {
        // Buscar en caché primero
        const cached = await redis.get(cacheKey);

        if (cached !== null) {
            // Ya estaba en caché, devolver sin calcular
            return res.json({
                x: x,
                y: parseFloat(cached),
                fuente: 'cache'
            });
        }

        // No estaba en caché, calcular
        const y = calcular(x);

        // Guardar en Redis y mantener solo las últimas 10 consultas
        await redis.lpush('consultas_recientes', cacheKey);
        await redis.ltrim('consultas_recientes', 0, MAX_CACHE - 1);
        await redis.set(cacheKey, y);

        return res.json({
            x: x,
            y: y,
            fuente: 'backend'
        });

    } catch (err) {
        return res.json({ error: err.message });
    }
});

app.listen(3000, '0.0.0.0', () => {
    console.log('Backend corriendo en puerto 3000');
});
```

#### Probar antes de crear el servicio

```bash
node app.js
```

Resultado esperado: `Backend corriendo en puerto 3000`

Desde otra terminal probar:

```bash
curl http://localhost:3000/consultar/5
```

Resultado esperado: `{"x":5,"y":78,"fuente":"backend"}`

#### Configurar como servicio del sistema

```bash
sudo nano /etc/systemd/system/api-cuadratica.service
```

```ini
[Unit]
Description=API Cuadratica Node.js
After=network.target

[Service]
User=fabricio1404
WorkingDirectory=/home/fabricio1404/app-cuadratica
ExecStart=/usr/bin/node /home/fabricio1404/app-cuadratica/app.js
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable api-cuadratica
sudo systemctl start api-cuadratica
sudo systemctl status api-cuadratica
```

Resultado esperado: `active (running)`

---

### FASE 5 — VM3: Servidor Frontend (Apache)

#### ¿Qué hace el frontend?

El frontend es la interfaz visual del sistema. Es una página HTML con un campo para ingresar X y un botón para consultar. Muestra el resultado y si vino del backend o del caché. También muestra un historial de las últimas consultas realizadas en la sesión.

#### Apache ya está instalado del trabajo anterior

Solo crear la carpeta y el archivo:

```bash
sudo mkdir /var/www/html/Cuadratica
sudo nano /var/www/html/Cuadratica/index.html
```

#### Código del Frontend (`/var/www/html/Cuadratica/index.html`)

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Consulta Modelo Cuadrático</title>
</head>
<body>
    <h1>Modelo Cuadrático</h1>
    <h2>y = 2x² + 5x + 3</h2>

    <input type="number" id="x" placeholder="Ingresá el valor de X">
    <button onclick="consultar()">Consultar</button>

    <div id="resultado"></div>

    <h2>Historial de Consultas</h2>
    <table border="1">
        <thead>
            <tr>
                <th>X</th>
                <th>Y</th>
                <th>Fuente</th>
            </tr>
        </thead>
        <tbody id="historial"></tbody>
    </table>

    <script>
        const API = 'http://localhost:3000';
        const historial = [];

        async function consultar() {
            const x = document.getElementById('x').value;

            if (x === '') {
                document.getElementById('resultado').innerHTML = '<p style="color:red">Ingresá un valor de X</p>';
                return;
            }

            const res = await fetch(`${API}/consultar/${x}`);
            const data = await res.json();

            if (data.error) {
                document.getElementById('resultado').innerHTML = `<p style="color:red">${data.error}</p>`;
                return;
            }

            document.getElementById('resultado').innerHTML = `
                <p><strong>Resultado:</strong> y = ${data.y}</p>
                <p><strong>Fuente:</strong> ${data.fuente === 'cache' ? '⚡ Cache (Redis)' : '🔧 Backend (calculado)'}</p>
            `;

            historial.unshift(data);
            if (historial.length > 10) historial.pop();

            const tbody = document.getElementById('historial');
            tbody.innerHTML = '';
            historial.forEach(h => {
                tbody.innerHTML += `<tr>
                    <td>${h.x}</td>
                    <td>${h.y}</td>
                    <td>${h.fuente === 'cache' ? '⚡ Cache' : '🔧 Backend'}</td>
                </tr>`;
            });
        }
    </script>
</body>
</html>
```

---

## Acceso al Sistema

```
http://localhost:8080/Cuadratica
```

---

## ¿Cómo se diferencia backend de caché?

La respuesta incluye el campo `fuente` que indica de dónde vino el resultado:

- `fuente: "backend"` → el valor de X nunca fue consultado, se calculó con el modelo y se guardó en Redis
- `fuente: "cache"` → el valor de X ya fue consultado antes, Redis lo tenía guardado y lo devolvió sin calcular

Ejemplo consultando `x = 5`:

**Primera consulta:**
```json
{ "x": 5, "y": 78, "fuente": "backend" }
```

**Segunda consulta (mismo valor):**
```json
{ "x": 5, "y": 78, "fuente": "cache" }
```

---

## Verificación del modelo matemático

El modelo implementado es `y = 2x² + 5x + 3`. Para verificar manualmente:

| X | Cálculo | Y |
|---|---|---|
| 0 | 2(0)² + 5(0) + 3 | 3 |
| 1 | 2(1)² + 5(1) + 3 | 10 |
| 5 | 2(5)² + 5(5) + 3 | 78 |
| 10 | 2(10)² + 5(10) + 3 | 253 |
| 20 | 2(20)² + 5(20) + 3 | 903 |
| 40 | 2(40)² + 5(40) + 3 | 3403 |

---

## Evidencia — Sistema Funcionando

**Acceso:** `http://localhost:8080/Cuadratica`

<img width="605" height="592" alt="Captura de pantalla 2026-06-01 115737" src="https://github.com/user-attachments/assets/06f9cde0-9a32-46a2-8ab2-c7da62e9e663" />
