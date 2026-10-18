# Servidor SNOMED CT (Snowstorm) con Docker en Linux Mint 20

**Guía de instalación · Versión 1.0 · Abril 2026**

## Contenido

1. [Introducción](#1-introducción)
2. [Instalación de Docker](#2-instalación-de-docker)
3. [Configuración de Snowstorm](#3-configuración-de-snowstorm)
4. [Importación de terminología SNOMED CT](#4-importación-de-terminología-snomed-ct)
5. [Verificación y acceso](#5-verificación-y-acceso)
6. [Anexo: errores frecuentes y soluciones](#anexo--errores-frecuentes-y-soluciones)
7. [Referencias](#referencias)

---

## 1. Introducción

[Snowstorm](https://github.com/IHTSDO/snowstorm) es un servidor de terminología SNOMED CT de código abierto desarrollado por SNOMED International (IHTSDO). Está construido sobre Elasticsearch y está diseñado para alta performance y escalabilidad empresarial.

Esta guía cubre la instalación completa en Linux Mint 20 usando Docker, incluyendo la carga de la edición Internacional y la extensión argentina.

> **Nota sobre licencias:** este repositorio **no incluye** los archivos de release de SNOMED CT (RF2). Deben descargarse por separado desde el [MLDS](https://mlds.ihtsdotools.org) o a través del Centro Nacional de Terminología en Salud, según los términos de licencia correspondientes.

### Requisitos del sistema

| Componente | Requisito mínimo |
|---|---|
| RAM | 16 GB recomendado (mínimo 8 GB) |
| Disco | 50 GB libres (Internacional ~25 GB, Extensión ~5 GB) |
| SO | Linux Mint 20 (Ubuntu 20.04 Focal) |
| Docker | Docker Engine + Docker Compose Plugin |

---

## 2. Instalación de Docker

### 2.1 Instalar Docker Engine

Linux Mint 20 está basado en Ubuntu 20.04 (Focal). Se debe usar el repositorio de Docker para Ubuntu:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu focal stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

> ⚠️ **Importante:** en Linux Mint, `lsb_release` devuelve `uma` o `vanessa`, no `focal`. Por eso se especifica `focal` manualmente en el repositorio.

### 2.2 Agregar usuario al grupo docker

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 2.3 Mover datos de Docker a /home (si la partición raíz es pequeña)

Si el disco raíz `/` tiene menos de 50 GB libres, es necesario mover Docker a una partición con más espacio. Esto es crítico porque Elasticsearch y los datos de SNOMED CT pueden ocupar más de 30 GB.

```bash
sudo systemctl stop docker.socket
sudo systemctl stop docker
sudo mkdir -p /home/docker
sudo rsync -aP /var/lib/docker/ /home/docker/
sudo mv /var/lib/docker /var/lib/docker.bak
echo '{"data-root": "/home/docker"}' | sudo tee /etc/docker/daemon.json
sudo systemctl start docker.socket && sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

> ✅ Verificar que dice: `Docker Root Dir: /home/docker`

Una vez verificado, se puede liberar el espacio de la copia anterior:

```bash
sudo rm -rf /var/lib/docker.bak
```

---

## 3. Configuración de Snowstorm

### 3.1 Clonar el repositorio

```bash
git clone https://github.com/IHTSDO/snowstorm.git
cd snowstorm
```

### 3.2 Configurar docker-compose.yml

Editar el archivo `docker-compose.yml` para optimizar la memoria y configurar el browser:

```bash
nano docker-compose.yml
```

Configuración recomendada para 16 GB de RAM:

```yaml
elasticsearch:
  environment:
    - "ES_JAVA_OPTS=-Xms3g -Xmx3g"
  mem_reservation: 3g

snowstorm:
  entrypoint: java -Xms2g -Xmx2g --add-opens java.base/java.lang=ALL-UNNAMED
    --add-opens=java.base/java.util=ALL-UNNAMED -cp @/app/jib-classpath-file
    org.snomed.snowstorm.SnowstormApplication --elasticsearch.urls=http://es:9200

browser:
  environment:
    - API_HOST=http://snowstorm:8080/
  ports:
    - 80:80
```

> ⚠️ Si el puerto 80 está ocupado (por Apache u otro servicio), cambiarlo a `8081:80` o detener el servicio con: `sudo systemctl stop apache2`

### 3.3 Configurar límite de memoria virtual

Elasticsearch requiere un límite de memoria virtual mayor al predeterminado:

```bash
sudo sysctl -w vm.max_map_count=262144
```

Para que persista tras reinicios:

```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 3.4 Levantar los contenedores

```bash
docker compose up -d
docker logs snowstorm -f 2>&1 | grep -i 'startup complete'
```

> ✅ Esperar el mensaje: `--- Snowstorm startup complete ---` antes de continuar.

---

## 4. Importación de terminología SNOMED CT

> 📋 **Flujo correcto:** siempre importar primero la edición Internacional en `MAIN`, luego crear el Code System de la extensión, y finalmente importar la extensión en su propio branch.

### 4.1 Ajustar watermarks de Elasticsearch

Antes de importar, configurar los límites de disco para evitar bloqueos:

```bash
curl -X PUT "http://localhost:9200/_cluster/settings" \
  -H "Content-Type: application/json" \
  -d '{
    "transient": {
      "cluster.routing.allocation.disk.watermark.low": "90%",
      "cluster.routing.allocation.disk.watermark.high": "95%",
      "cluster.routing.allocation.disk.flood_stage": "97%"
    }
  }'
```

### 4.2 Importar la edición Internacional

Ejecutar los dos comandos juntos, sin pausa entre ellos (reemplazar la ruta y el nombre del archivo por los propios):

```bash
IMPORT_ID=$(curl -s -X POST "http://localhost:8080/imports" \
  -H "Content-Type: application/json" \
  -d '{"branchPath": "MAIN", "createCodeSystemVersion": true, "type": "SNAPSHOT"}' \
  -D - 2>&1 | grep -i "location" | sed 's/.*imports\///' | tr -d '\r\n') \
&& echo "Import ID: $IMPORT_ID" \
&& curl -X POST "http://localhost:8080/imports/${IMPORT_ID}/archive" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@/home/usuario/Descargas/SnomedCT_InternationalRF2_PRODUCTION_XXXXXXX.zip"
```

> ⏳ Este proceso puede tardar entre 40 y 60 minutos. No interrumpir.

Monitorear en otra terminal:

```bash
docker logs snowstorm -f 2>&1 | grep -i 'import\|completed\|failed'
```

> ✅ Esperar: `RF2 SNAPSHOT import completed on branch MAIN`

### 4.3 Crear el Code System para Argentina

```bash
curl -X POST "http://localhost:8080/codesystems" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "SNOMED CT Argentina Edition",
    "shortName": "SNOMEDCT-AR",
    "branchPath": "MAIN/SNOMEDCT-AR",
    "countryCode": "ar",
    "defaultLanguageCode": "es",
    "defaultLanguageReferenceSets": ["450828004"]
  }'
```

### 4.4 Importar la extensión argentina

> ⚠️ La extensión argentina debe corresponder a la misma versión base de la Internacional. Usar **ExtensionRelease**, NO EditionRelease.

```bash
IMPORT_ID=$(curl -s -X POST "http://localhost:8080/imports" \
  -H "Content-Type: application/json" \
  -d '{"branchPath": "MAIN/SNOMEDCT-AR", "createCodeSystemVersion": true, "type": "SNAPSHOT"}' \
  -D - 2>&1 | grep -i "location" | sed 's/.*imports\///' | tr -d '\r\n') \
&& echo "Import ID: $IMPORT_ID" \
&& curl -X POST "http://localhost:8080/imports/${IMPORT_ID}/archive" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@/home/usuario/Descargas/SnomedCT_Argentina-ExtensionRelease_PRODUCTION_XXXXXX.zip"
```

> ⏳ Este proceso tarda entre 10 y 20 minutos.

> ✅ Esperar: `RF2 SNAPSHOT import completed on branch MAIN/SNOMEDCT-AR`

---

## 5. Verificación y acceso

### 5.1 Verificar la carga

```bash
# Contar conceptos en MAIN
curl -s "http://localhost:8080/MAIN/concepts?limit=1" | python3 -m json.tool | grep total

# Verificar Code Systems
curl -s "http://localhost:8080/codesystems" | python3 -m json.tool

# Verificar branches
curl -s "http://localhost:8080/branches" | python3 -m json.tool | grep path
```

### 5.2 URLs de acceso

| Servicio | URL |
|---|---|
| Navegador SNOMED CT | http://localhost |
| API Snowstorm (Swagger) | http://localhost:8080/swagger-ui/index.html |
| FHIR API | http://localhost:8080/fhir |
| Elasticsearch | http://localhost:9200 |

### 5.3 Comandos de gestión

```bash
# Iniciar los contenedores
docker compose up -d

# Detener
docker compose stop

# Ver logs
docker logs snowstorm -f

# Ver estado
docker ps
```

---

## Anexo — Errores frecuentes y soluciones

### A. Tabla de errores

#### `address already in use` (puerto 80)

- **Causa:** Apache2 u otro servicio ocupa el puerto 80.
- **Solución:** detener Apache con `sudo systemctl stop apache2`, o cambiar el puerto del browser a `8081:80` en `docker-compose.yml`.

#### `disk usage exceeded flood-stage watermark`

- **Causa:** el disco donde Docker guarda los datos está lleno. Elasticsearch bloquea los índices.
- **Solución:** mover Docker a una partición con más espacio (ver [sección 2.3](#23-mover-datos-de-docker-a-home-si-la-partición-raíz-es-pequeña)). Luego desbloquear los índices:

```bash
curl -X PUT localhost:9200/_all/_settings \
  -H "Content-Type: application/json" \
  -d '{"index.blocks.read_only_allow_delete": null}'
```

#### `There is a partial commit on this branch`

- **Causa:** un import anterior falló y dejó el branch `MAIN` bloqueado.
- **Solución:** ejecutar el rollback `POST /admin/rollback-partial-commit/MAIN`. Luego desbloquear: `POST /branches/MAIN/actions/unlock`. Si falla, editar directamente en Elasticsearch (ver [sección B](#b-desbloquear-branch-manualmente-en-elasticsearch)).

#### `all shards failed (search_phase_execution_exception)`

- **Causa:** Elasticsearch no tiene suficiente heap para el scroll del `SemanticIndexUpdateService`.
- **Solución:** aumentar la memoria de ES a 3g en `docker-compose.yml` (`ES_JAVA_OPTS=-Xms3g -Xmx3g`) y reducir Snowstorm a `-Xmx2g`.

#### `SnomedComponent.setChanged() because component is null`

- **Causa:** incompatibilidad de versiones: la extensión argentina referencia conceptos que no existen en la Internacional importada.
- **Solución:** verificar que las versiones sean compatibles. La extensión argentina de febrero 2026 requiere la Internacional de enero/febrero 2026, no la de abril 2026 (ver [sección D](#d-verificar-compatibilidad-de-versiones)).

#### `Import job not found`

- **Causa:** el job de import expiró antes de recibir el archivo ZIP.
- **Solución:** crear el job y subir el archivo en el mismo bloque de comandos usando la variable `$IMPORT_ID` (ver [sección 4.2](#42-importar-la-edición-internacional)).

#### `vm.max_map_count` demasiado bajo

- **Causa:** el sistema operativo limita la memoria virtual necesaria para Elasticsearch.
- **Solución:** `sudo sysctl -w vm.max_map_count=262144`. Para que persista, agregar la línea a `/etc/sysctl.conf`.

#### El browser muestra ediciones internacionales pero no datos locales

- **Causa:** el browser apunta a la API pública de SNOMED International.
- **Solución:** agregar en `docker-compose.yml`, bajo `browser:`, `environment: - API_HOST=http://snowstorm:8080/`.

### B. Desbloquear branch manualmente en Elasticsearch

Cuando el unlock vía API falla, se puede desbloquear el branch directamente en Elasticsearch:

```bash
# Buscar el documento bloqueado
curl -s "http://localhost:9200/branch/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"term": {"path": "MAIN"}}}' | python3 -m json.tool | grep -E '"_id"|"locked"'

# Actualizar el documento bloqueado (reemplazar ID con el _id real)
curl -X POST "http://localhost:9200/branch/_update/ID_BLOQUEADO" \
  -H "Content-Type: application/json" \
  -d '{"doc": {"locked": false, "containsContent": true}}'

# Reiniciar Snowstorm para que lea el cambio
docker restart snowstorm && sleep 90
```

### C. Reinicio limpio completo

Si los errores persisten y se necesita empezar desde cero:

```bash
cd ~/snowstorm
docker compose down
docker volume rm snowstorm_elastic
docker compose up -d
```

> ⚠️ Este proceso elimina todos los datos de Elasticsearch. Será necesario reimportar toda la terminología.

### D. Verificar compatibilidad de versiones

Para verificar qué versión de la Internacional requiere la extensión argentina:

```bash
unzip -p SnomedCT_Argentina-ExtensionRelease_XXXXXX.zip \
  "$(unzip -l SnomedCT_Argentina-ExtensionRelease_XXXXXX.zip | grep ModuleDependency | grep Snapshot | awk '{print $4}')" | head -5
```

La fecha en el archivo de `ModuleDependency` indica la versión base requerida de la Internacional.

---

## Referencias

### Documentación oficial de Snowstorm

1. SNOMED International (IHTSDO). *Snowstorm — Scalable SNOMED CT Terminology Server* [repositorio GitHub]. https://github.com/IHTSDO/snowstorm
2. SNOMED International (IHTSDO). *Using Docker — Snowstorm documentation*. https://github.com/IHTSDO/snowstorm/blob/master/docs/using-docker.md
3. SNOMED International (IHTSDO). *Loading SNOMED CT — Snowstorm documentation*. https://github.com/IHTSDO/snowstorm/blob/master/docs/loading-snomed.md
4. SNOMED International (IHTSDO). *Snowstorm — Docker Hub*. https://hub.docker.com/r/snomedinternational/snowstorm
5. SNOMED International (IHTSDO). *Snowstorm Configuration Guide*. https://github.com/IHTSDO/snowstorm/blob/master/docs/configuration-guide.md

### SNOMED CT — Terminología y ediciones

6. SNOMED International. *SNOMED CT Terminology Services — Implementation Guide*. https://www.implementation.snomed.org/terminology-services
7. SNOMED International. *SNOMED CT — RF2 Release Format Specification*. https://confluence.ihtsdotools.org/display/DOCRFMT
8. SNOMED International. *SNOMED CT Argentine Edition*. Centro Nacional de Terminología en Salud (CNTS) — Ministerio de Salud de Argentina. https://www.snomed.org/members/argentina
9. SNOMED International. *Member Licensing and Distribution Service (MLDS)*. https://mlds.ihtsdotools.org
10. SNOMED International. *SNOMED CT Browser — sct-browser-frontend* [repositorio GitHub]. https://github.com/IHTSDO/sct-browser-frontend

### Docker y Elasticsearch

11. Docker Inc. *Install Docker Engine on Ubuntu*. https://docs.docker.com/engine/install/ubuntu/
12. Docker Inc. *Docker Compose — Overview*. https://docs.docker.com/compose/
13. Elastic. *Elasticsearch Reference 8.11 — Docker*. https://www.elastic.co/guide/en/elasticsearch/reference/8.11/docker.html
14. Elastic. *Virtual memory — vm.max_map_count configuration*. https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html
15. Elastic. *Disk-based shard allocation settings*. https://www.elastic.co/guide/en/elasticsearch/reference/current/disk-allocator.html
16. Elastic. *Circuit breaker settings*. https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html

### Linux Mint y sistema operativo

17. Linux Mint Project. *Linux Mint 20 "Ulyana" — Release Notes*. https://www.linuxmint.com/rel_ulyana.php
18. Canonical Ltd. *Ubuntu 20.04 LTS (Focal Fossa) — Release Notes*. https://releases.ubuntu.com/20.04/
19. The rsync Project. *rsync — a fast, versatile, remote (and local) file-copying tool*. https://rsync.samba.org/documentation.html

### Estándares y normas relacionados

20. Health Level Seven International (HL7). *HL7 FHIR R4 — Terminology Services*. https://www.hl7.org/fhir/terminology-service.html
21. SNOMED International. *SNOMED CT Expression Constraint Language (ECL) Specification*. https://confluence.ihtsdotools.org/display/DOCECL
22. SNOMED International. *SNOMED CT Compositional Grammar Specification*. https://confluence.ihtsdotools.org/display/DOCCG
