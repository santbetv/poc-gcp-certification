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
- nano create-web-vms.sh
- chmod +x create-web-vms.sh
- ./create-web-vms.sh
