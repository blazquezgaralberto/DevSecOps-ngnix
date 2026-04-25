# DevSecOps con NGINX y Docker Scout

Práctica de DevSecOps que demuestra cómo integrar análisis de seguridad en el pipeline de CI/CD
de una aplicación web containerizada con NGINX.

## Objetivo

Comparar una imagen base insegura (`nginx:latest`) con una imagen hardened (`chainguard/nginx`)
y aprender a detectar vulnerabilidades antes de desplegar usando Docker Scout.

## Estructura del proyecto

```
.
├── Dockerfile          # Imagen INSECURE (nginx:latest)
├── CICD.sh             # Pipeline de build y test sin análisis de seguridad
├── webapp/             # Contenido estático de la app
└── solution/
    ├── Dockerfile      # Imagen SECURE (chainguard/nginx)
    └── CICD.sh         # Pipeline completo con Docker Scout integrado
```

## Requisitos previos

- Docker Desktop con Docker Scout habilitado
- Cuenta en Docker Hub (para autenticarse con Scout)

---

## Flujo de la práctica

### 1. Build de la imagen

```bash
docker build -t $APP_NAME:$VERSION .
```

Construye la imagen Docker a partir del `Dockerfile`. En la versión **INSECURE** se usa
`nginx:latest`, una imagen genérica que arrastra muchas dependencias y CVEs conocidos.
En la versión **SECURE** (`solution/`) se usa `cgr.dev/chainguard/nginx`, una imagen
minimalista y endurecida (distroless) que reduce drásticamente la superficie de ataque.

### 2. Análisis de vulnerabilidades con Docker Scout

```bash
docker scout cves $APP_NAME:$VERSION --output ./vulns.report
```

Escanea la imagen y genera un informe completo de todas las vulnerabilidades CVE conocidas
en sus capas y dependencias. El fichero `vulns.report` puede archivarse como evidencia
en el pipeline.

```bash
docker scout cves $APP_NAME:$VERSION --only-severity critical --exit-code
```

Igual que el anterior pero filtrando solo vulnerabilidades **críticas** y devolviendo
código de salida no-cero si se encuentran. Esto permite que el pipeline de CI/CD falle
automáticamente si la imagen no supera el umbral de seguridad (**security gate**).

### 3. Generación del SBOM

```bash
docker scout sbom --output $APP_NAME.sbom $APP_NAME:$VERSION
```

Genera el **Software Bill of Materials (SBOM)**: un inventario completo de todos los
paquetes y librerías incluidos en la imagen. Es un requisito creciente en normativas
de seguridad (NIST, EO 14028) y permite auditar la cadena de suministro del software.

### 4. Despliegue del contenedor

```bash
docker run -d -p 80:80 --name $APP_NAME $APP_NAME:$VERSION
```

Arranca el contenedor en segundo plano (`-d`) mapeando el puerto 80 del host al
contenedor. Solo debe ejecutarse si los pasos anteriores han pasado sin errores críticos.

---

## Comparativa de resultados

| | INSECURE (`nginx:latest`) | SECURE (`chainguard/nginx`) |
|---|---|---|
| CVEs críticos | Alto | Muy bajo o ninguno |
| Tamaño imagen | ~190 MB | ~50 MB |
| Shell incluida | Sí | No (distroless) |
| SBOM entries | Muchos | Mínimos |

## Concepto clave: Shift Left Security

Integrar el análisis de seguridad **dentro del pipeline de CI/CD** (en lugar de hacerlo
post-despliegue) es el principio de "shift left": detectar y corregir problemas lo antes
posible en el ciclo de desarrollo, cuando el coste de remediación es menor.
