# Gestor de Jobs de Kubernetes — Proyecto 202630

CLI en Python que reemplaza el copy-paste de manifiestos `Job` de Kubernetes: en vez de
editar un YAML cada vez que se quiere correr una tarea distinta, el programa presenta un
menú donde se elige el lenguaje, la tarea y la complejidad, y crea el `Job` directamente
con el cliente oficial de Kubernetes (`kubernetes` para Python) — sin invocar `kubectl`
desde el código.

El recorrido que hace el programa es siempre el mismo:

```text
gestor_jobs.py -> Job de Kubernetes -> Pod -> contenedor con argumentos -> logs
```

## Qué hace el menú

```text
=== Gestor de Jobs - namespace estudiantes-202630 ===
  1. Ejecutar una tarea nueva
  2. Revisar el estado de las tareas
  3. Ver los logs de una tarea
  4. Limpiar las tareas terminadas
  5. Salir
```

- **Ejecutar una tarea nueva**: pide lenguaje, tarea y complejidad en listas numeradas (el
  menú de complejidad muestra de una vez el N y la memoria que implica cada nivel), arma
  el `Job` con `client.V1Job` y lo envía al clúster con `create_namespaced_job`.
- **Revisar el estado**: lista los Jobs del namespace y traduce los contadores
  `active` / `succeeded` / `failed` a texto: en ejecución, completado, fallido, o
  pendiente si el Pod todavía no arrancó.
- **Ver logs**: encuentra el Pod del Job por la etiqueta `job-name` que Kubernetes le
  agrega automáticamente, y muestra su salida.
- **Limpiar terminadas**: borra (con confirmación) los Jobs que ya no tienen Pods activos,
  dejando intactos los que siguen corriendo.

## Tareas y complejidad

Los contenedores (`imagenes/python`, `imagenes/java`, `imagenes/c`) comparten el mismo
contrato: reciben `<tarea> <N>` como argumentos y terminan con un JSON de una línea en
stdout.

| tarea | qué hace | parámetro N |
|---|---|---|
| `hola` | saluda e imprime datos del runtime (versión, hostname, CPUs) | no usa N |
| `ordenar` | genera N enteros aleatorios, los ordena y verifica el resultado | cantidad de elementos |
| `fib` | calcula fib(N) recursivo sin memoización — satura una CPU | N |
| `matriz` | multiplica dos matrices N×N con triple bucle | dimensión |
| `primos` *(agregada por el grupo, solo en Python)* | cuenta los primos menores que N por división de tanteo | límite del conteo |

La complejidad (`baja` / `media` / `alta`) fija dos cosas al mismo tiempo: cuánta CPU y
memoria le pide el `Job` a Kubernetes (`requests`/`limits`), y qué tan grande es N. Por
eso un nivel alto no solo tarda más — también reserva más recursos, y bajar el
`mem_limit` por debajo de lo que la tarea necesita es la forma de forzar un `OOMKilled`.

## Cómo levantarlo (Windows + Docker Desktop + minikube)

```powershell
minikube start --driver=docker --container-runtime=docker
kubectl apply -f .\k8s\00-namespace.yaml
.\scripts\preparar-imagenes.ps1
pip install -r .\entrega\requirements.txt
python .\entrega\gestor_jobs.py
```

`--container-runtime=docker` no es opcional: si minikube arranca con `containerd` (el
default en versiones recientes), `docker-env` falla en Windows con un error de
`SSH_AGENT_START`, y aunque no fallara, las imágenes que construye
`preparar-imagenes.ps1` quedarían en el daemon `docker` mientras Kubernetes corre los
Pods con `containerd` — dos almacenes de imágenes distintos, así que el Pod terminaría en
`ErrImageNeverPull`. Si ya tienes un perfil de minikube creado con otro runtime, bórralo
primero con `minikube delete`.

`preparar-imagenes.ps1` construye las tres imágenes directo contra el demonio Docker de
minikube, así que hay que volver a correrlo cada vez que se modifique el programa de un
contenedor (`minikube image load` no sirve para esto: no reemplaza un tag existente).

Para verificar contra el clúster directamente:

```powershell
kubectl get jobs,pods -n estudiantes-202630
kubectl logs -n estudiantes-202630 job/<nombre-del-job>
```

## Estructura del repo

- `entrega/gestor_jobs.py` — el CLI, implementado sobre el esqueleto del curso.
- `imagenes/{python,java,c}/` — las tres imágenes de los contenedores.
- `catalogo/tareas.json` — catálogo de lenguajes, imágenes y tareas admitidas.
- `k8s/00-namespace.yaml` — namespace dedicado (`estudiantes-202630`).
- `scripts/preparar-imagenes.ps1` — construye y carga las imágenes en minikube.

## Tarea adicional: `primos`

Se agregó una quinta tarea, solo en la imagen de Python (`imagenes/python/tareas.py`):
cuenta los números primos menores que N probando divisores hasta la raíz cuadrada de
cada número (división de tanteo, sin criba de Eratóstenes), para que el trabajo escale
de forma predecible con N — igual que `fib` o `matriz`. Se tocaron tres archivos:

1. `imagenes/python/tareas.py` — función `primos(n)`, registrada en el diccionario
   `TAREAS`.
2. `catalogo/tareas.json` — se agregó `"primos"` a la lista de tareas de `python`.
3. `entrega/gestor_jobs.py` — fila `"primos": {"baja": 100000, "media": 500000, "alta":
   1500000}` en la tabla `TAMANOS`.

## Video de demo

[Video de demo en YouTube](AGREGAR-LINK-AQUI) — pendiente.

## Integrantes

- Agregar aquí los nombres del grupo.
