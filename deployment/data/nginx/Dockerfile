# Custom nginx-Image für das Basilicom-Deploy.
#
# Grund: Onyx' nginx mountet seine Config-Templates per Host-Bind
# (`../data/nginx:/nginx-templates`). In Portainer-Git-Deploys zeigt dieser
# Host-Pfad ins Leere (Portainer-Storage != Docker-Host-Root) -> run-nginx.sh
# fehlt -> nginx crasht. Lösung: Templates ins Image backen, dann ist KEIN
# Bind-Mount mehr nötig.
#
# Wird per `build:` in docker-compose.basilicom.yml direkt von Portainer/Compose
# gebaut (Build-Context = dieses Verzeichnis, von Portainer lesbar).
FROM nginx:1.25.5-alpine
COPY . /nginx-templates/
