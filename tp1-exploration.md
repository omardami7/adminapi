\# TP1 : Exploration des API Docker et NetBox



\*\*Objectif\*\* : parler à la main, avec cURL, aux deux API du projet (Docker Engine et NetBox),

pour comprendre les requêtes, les réponses et les codes HTTP avant d'écrire du Python.



Environnement : Windows, Docker Desktop (API Docker 1.56), NetBox 4.7, curl.



```bash

D=http://127.0.0.1:2375        # API Docker (via docker-socket-proxy)

N=http://127.0.0.1:8080/api    # API NetBox

```



\## 1. Conteneurs de test



```bash

docker run -d --name web1 --label adminapi.managed=true nginx:alpine

docker run -d --name intrus nginx:alpine

```



`web1` porte le label `adminapi.managed=true`, `intrus` non. Cela permet de vérifier qu'on sait

distinguer les conteneurs gérés par notre API de tous les autres conteneurs de la machine.



\## 2. Lecture de l'API Docker



| Commande | Code | Observation |

|---|---|---|

| `curl -s $D/version` | 200 | Version du moteur Docker (29.9.2) et de son API (1.56). Docker tourne dans WSL2. |

| `curl -s "$D/containers/json?all=true"` | 200 | 6 conteneurs : `web1`, `intrus`, et les 4 de l'environnement (netbox, postgres, valkey, docker-api). `all=true` inclut les conteneurs arrêtés (équivalent de `docker ps -a`). |

| `curl -s --get $D/containers/json --data-urlencode 'all=true' --data-urlencode 'filters={"label":\["adminapi.managed=true"]}'` | 200 | Seul `web1` reste : `intrus` n'a pas le label. |

| `curl -s $D/containers/web1/json` | 200 | Fiche complète (équivalent de `docker inspect`). Le label est dans `Config.Labels`, l'état dans `State.Status` (`running`). |



\*\*Intérêt pour le projet\*\* : notre API ne doit voir et toucher que les conteneurs portant le label.

Le filtre protège les autres conteneurs de la machine (par exemple la base de NetBox).

`GET /containers` reprendra la requête filtrée, et `GET /containers/{id}` vérifiera

`Config.Labels` pour répondre 404 si le label est absent.



\## 3. Arrêt de web1 et conteneur inexistant



| Commande | Code | Signification |

|---|---|---|

| `curl -i -X POST $D/containers/web1/stop` (1re fois) | 204 No Content | L'arrêt a réussi. Docker ne renvoie aucun corps de réponse. |

| `curl -i -X POST $D/containers/web1/stop` (2e fois) | 304 Not Modified | Le conteneur était déjà arrêté : rien n'a changé. Ce n'est pas une erreur. |

| `curl -i $D/containers/nexistepas/json` | 404 Not Found | Le conteneur n'existe pas. Docker ajoute un message JSON : `No such container: nexistepas`. |



\*\*Idempotence\*\* : arrêter deux fois donne le même état final (le conteneur est arrêté).

Répéter l'action ne casse rien : l'arrêt est idempotent.



\*\*Intérêt pour le projet\*\* : pour un conteneur déjà arrêté, Docker répond 304, mais le contrat

de notre API impose \*\*409 Conflict\*\*. Notre API devra donc traduire les réponses de Docker

dans sa propre langue : 304 → 409, 404 → 404.



Le conteneur a ensuite été redémarré avec `curl -X POST $D/containers/web1/start` (204).



\## 4. NetBox : interface, documentation et token



\- Interface : http://127.0.0.1:8080 (admin / admin)

\- Documentation Swagger : http://127.0.0.1:8080/api/schema/swagger-ui/



Obtention d'un token :



```bash

curl -s -X POST $N/users/tokens/provision/ \\

&#x20;    -H "Content-Type: application/json" -H "Accept: application/json" \\

&#x20;    -d '{"username": "admin", "password": "admin"}'

```



Le token s'écrit `nbt\_<key>.<token>`. Il est rangé dans `.env`, qui est dans le `.gitignore` :

il n'est jamais versionné. Chargement dans le terminal :



```bash

set -a; source .env; set +a

A="Authorization: Bearer $NETBOX\_TOKEN"

```



| Commande | Code | Observation |

|---|---|---|

| `curl -s $N/virtualization/clusters/ -H "$A"` | 200 | Le token est accepté. |

| `curl -s $N/virtualization/clusters/` (sans token) | 403 | NetBox refuse la requête. |



\*\*401 ou 403 ?\*\* En principe, 401 Unauthorized signifie « je ne sais pas qui tu es »

(pas de token ou token invalide), et 403 Forbidden « je sais qui tu es, mais tu n'as pas le droit ».

NetBox répond 403 sans token. Le contrat de notre API est plus précis : 401 sans token,

403 pour un token lecteur qui tente une écriture.



\*\*Problèmes rencontrés et solutions\*\* :

\- `curl` de MSYS2 bloqué (`Permission denied`) : utilisation du curl de Windows

&#x20; (`alias curl=/c/Windows/System32/curl.exe`).

\- 403 avec token : la variable était vide, puis le token était incomplet (il manquait

&#x20; `nbt\_<key>.`). Corrigé en assemblant `nbt\_` + key + `.` + token.

\- Le Bloc-notes ajoute un caractère invisible `\\r` en fin de ligne (fins de ligne Windows),

&#x20; qui fausse le token sous Bash. Supprimé avec `sed -i 's/\\r$//' .env`.



\## 5. Type de cluster et cluster



```bash

curl -i -X POST $N/virtualization/cluster-types/ -H "$A" \\

&#x20;    -H "Content-Type: application/json" -H "Accept: application/json" \\

&#x20;    -d '{"name": "Docker", "slug": "docker"}'



curl -i -X POST $N/virtualization/clusters/ -H "$A" \\

&#x20;    -H "Content-Type: application/json" -H "Accept: application/json" \\

&#x20;    -d '{"name": "docker-local", "type": 1}'

```



| Création | Code | id obtenu |

|---|---|---|

| Type de cluster `Docker` | 201 Created | 1 |

| Cluster `docker-local` | 201 Created | 1 |



\- \*\*201 et pas 200\*\* : 200 répond à une lecture, 201 indique qu'une ressource a été \*\*créée\*\*.

\- L'en-tête `location` donne l'adresse de la nouvelle ressource

&#x20; (`/api/virtualization/clusters/1/`).

\- Les objets sont reliés par leur \*\*id\*\* : on envoie `"type": 1`, et NetBox renvoie le type complet

&#x20; dans la réponse.



Le cluster `docker-local` est visible dans l'interface (Virtualisation → Clusters).



\*\*Intérêt pour le projet\*\* : chaque conteneur créé par notre API aura sa machine virtuelle

dans ce cluster. Notre code devra retrouver l'id de `docker-local` pour l'y rattacher,

et notre route de création devra elle aussi répondre 201.



\## 6. Dépôt GitHub



URL du dépôt : https://github.com/<omardami7>/adminapi

