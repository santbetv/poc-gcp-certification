# poc-gcp-certification
# Implementing Cloud Load Balancing for Compute Engine

## Creation apps

-   Crea las tres VM (`web1`, `web2`, `web3`) en la zona `europe-west1-b` bajo la red `default`.
    
-   Les añade la etiqueta `network-lb-tag`.
    
-   Usa Debian 11, tipo `e2-small`.
    
-   Inyecta un _startup script_ que instala Apache y escribe el hostname en la página de inicio.
    
-   Crea la regla de firewall para permitir HTTP (puerto 80) solo a instancias con esa etiqueta.

```bash
    #!/bin/bash
    
    # 1) Variables comunes
    ZONE=europe-west1-b
    MACHINE_TYPE=e2-small
    IMAGE_FAMILY=debian-11
    IMAGE_PROJECT=debian-cloud
    TAG=network-lb-tag
    
    # 2) Crear las tres instancias web1, web2 y web3
    gcloud compute instances create web1 web2 web3 \
      --zone=${ZONE} \
      --machine-type=${MACHINE_TYPE} \
      --tags=${TAG} \
      --image-family=${IMAGE_FAMILY} \
      --image-project=${IMAGE_PROJECT} \
      --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y apache2
    systemctl restart apache2
    echo "<h3>Web Server: $(hostname)</h3>" | tee /var/www/html/index.html
    '
    
    # 3) Crear la regla de firewall para HTTP
    gcloud compute firewall-rules create allow-http \
      --network=default \
      --direction=INGRESS \
      --action=ALLOW \
      --rules=tcp:80 \
      --target-tags=${TAG} \
      --description="Allow HTTP traffic to web instances"
    
    # 4) Listar instancias para verificar
    gcloud compute instances list --zones=${ZONE}
```
```bash
Other: Crear web1 desde la Consola web
Entra en la Cloud Console → Compute Engine → VM instances.

Haz clic en Create Instance.

Rellena exactamente estos valores:

Name: web1

Region: europe-west1

Zone: europe-west1-b

Machine configuration → Series: E2 → Machine type: e2-small

Boot disk → Public images → Debian → Version: Debian 11 (bullseye)

Firewall → Marca Allow HTTP traffic (o, si no sale, ve a Management, security, disks... → Networking → añade la etiqueta network-lb-tag)

En Management, security, disks... → Metadata → Startup script, pega:

bash
Copiar
Editar
#!/bin/bash
apt-get update
apt-get install -y apache2
systemctl restart apache2
echo "<h3>Web Server: web1</h3>" | tee /var/www/html/index.html
Haz Create.

Espera unos segundos y refresca el lab: Check my progress.
```

- nano create-web-vms.sh
- chmod +x create-web-vms.sh
- ./create-web-vms.sh
- curl http://[EXTERNAL_IP]

# Implementing Cloud Load Balancing for Compute Engine

## Implement Load Balancing on Compute Engine

```bash
    #!/bin/bash
    
    # Variables
    REGION=us-west4
    ZONE=us-west4-b
    TEMPLATE=lb-backend-template
    MIG=lb-backend-group
    FW_RULE=fw-allow-health-check
    STATIC_IP=lb-ipv4-1
    HC=http-basic-check
    BACKEND_SERVICE=web-backend-service
    URL_MAP=web-map-http
    HTTP_PROXY=http-lb-proxy
    FW_FORWARD=http-content-rule
    
    # 1) Crea el instance template
    gcloud compute instance-templates create $TEMPLATE \
      --machine-type=e2-medium \
      --image-family=debian-11 \
      --image-project=debian-cloud \
      --tags=allow-health-check \
      --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y apache2
    a2ensite default-ssl
    a2enmod ssl
    vm_hostname="$(curl -H "Metadata-Flavor:Google" http://169.254.169.254/computeMetadata/v1/instance/name)"
    echo "Page served from: $vm_hostname" | tee /var/www/html/index.html
    systemctl restart apache2
    '
    
    # 2) Crea el managed instance group
    gcloud compute instance-groups managed create $MIG \
      --template=$TEMPLATE \
      --size=2 \
      --zone=$ZONE
    
    # 3) Crea la regla de firewall para health checks
    gcloud compute firewall-rules create $FW_RULE \
      --network=default \
      --action=ALLOW \
      --direction=INGRESS \
      --source-ranges=130.211.0.0/22,35.191.0.0/16 \
      --target-tags=allow-health-check \
      --rules=tcp:80

        ```bash
            # Comprueba direcciones regionales
            gcloud compute addresses list --filter="name=lb-ipv4-1"
            # Si aparece en alguna región, bórrala:
            gcloud compute addresses delete lb-ipv4-1 --region=<that-region>
            # Reserva la IP como GLOBAL
            gcloud compute addresses create lb-ipv4-1 \
              --ip-version=IPV4 \
              --global
            # Verifica que exista
            gcloud compute addresses list --global --filter="name=lb-ipv4-1"
        ```
    
    # 4) Reserva la IP estática global
    gcloud compute addresses create $STATIC_IP \
      --ip-version=IPV4 \
      --global
    
    # 5) Crea el health check HTTP
    gcloud compute health-checks create http $HC \
      --port=80
    
    # 6) Crea el backend service y añade el MIG
    gcloud compute backend-services create $BACKEND_SERVICE \
      --protocol=HTTP \
      --global \
      --health-checks=$HC
    
    gcloud compute backend-services add-backend $BACKEND_SERVICE \
      --instance-group=$MIG \
      --instance-group-zone=$ZONE \
      --global
    
    # 7) Crea el URL map
    gcloud compute url-maps create $URL_MAP \
      --default-service=$BACKEND_SERVICE
    
    # 8) Crea el target HTTP proxy
    gcloud compute target-http-proxies create $HTTP_PROXY \
      --url-map=$URL_MAP
    
    # 9) Crea la forwarding rule global
    gcloud compute forwarding-rules create $FW_FORWARD \
      --address=$STATIC_IP \
      --global \
      --target-http-proxy=$HTTP_PROXY \
      --ports=80
```

