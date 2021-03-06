Getting Started: Create and Manage Cloud Resources: Challenge Lab

Abrir GCLOUD:

#CONFIGURACIONES ANTES DE EMPEZAR:

#Configuración de la región y zona por defecto para los comandos gcloud según laboratorio us-east1-x
gcloud config set compute/region us-east1
gcloud config set compute/zone us-east1-b

#CREACIÓN DE LA INSTANCIA (GCE):

#Creación de instancia con nombre y tipo de máquina:
gcloud compute instances create nucleus-jumphost --machine-type f1-micro

#CREACIÓN DEL CLUSTER DE K8S MAS EXPOSICIÓN DE LA APLICACIÓN (GKE):

#Creación de cluster de k8s (por defecto tiene 3 nodos de n1-standard-1):
gcloud container clusters create nucleus-cluster

#Autenticarse contra el cluster de k8s
gcloud container clusters get-credentials nucleus-cluster

#Deploy de la aplicación en cluster de k8s
kubectl create deployment nucleus-app --image=gcr.io/google-samples/hello-app:2.0

#Creación del servicio para exponer la aplicación en cluster de k8s
kubectl expose deployment nucleus-app --type=LoadBalancer --port 8080

#Verificar que el servicio fue creado y obtener la IP Externa con la que se expone la aplicación.
kubectl get service

#Llamar al servicio para probar que esté funcionando correctamente
http://IP_DEL_SERVICIO

#CREACIÓN DEL BALANCEADOR DE CARGA HTTP/S:

#Startup Script:

cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

#Creación de Instance Template que utilizará el Startup Script
gcloud compute instance-templates create nginx-template \
         --metadata-from-file startup-script=startup.sh

#Creación de Target Pool (Permite un punto de acceso único a todas las instancias del grupo) (https://cloud.google.com/load-balancing/docs/target-pools):
gcloud compute target-pools create nginx-pool

#Creación de Managed Instancia Group utilizando el Intance Template:
gcloud compute instance-groups managed create nginx-group \
         --base-instance-name nginx \
         --size 2 \
         --template nginx-template \
         --target-pool nginx-pool

#Verificar las instancias creadas:
gcloud compute instances list

#Crear regla de firewall para permitir tráfico 80/TCO
gcloud compute firewall-rules create www-firewall --allow tcp:80

#Crear un health check (Permite verificar si las instancias están respondiendo a tráfico http o https) (https://cloud.google.com/load-balancing/docs/health-checks)
gcloud compute http-health-checks create http-basic-check

#Setear named ports (Se mapea el puerto 80 al grupo nginx-group) (https://blog.realkinetic.com/http-to-https-using-google-cloud-load-balancer-dda57ac97c)
gcloud compute instance-groups managed \
       set-named-ports nginx-group \
       --named-ports http:80

#Crear el backend service
gcloud compute backend-services create nginx-backend \
      --protocol HTTP --http-health-checks http-basic-check --global

#Agregar el instance Group al Backend Service
gcloud compute backend-services add-backend nginx-backend \
    --instance-group nginx-group \
    --instance-group-zone us-east1-b \
    --global

#Crear URL Map (Direcciona todos los request de entrada a todas tus instancias)
gcloud compute url-maps create web-map \
    --default-service nginx-backend

#Crear HTTP Proxy (Enruta tus request a tu URL Map)
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map

#Crear global forwarding rule (Maneja todos los request entrantes y los envía al proxy dependiendo de la dirección IP, protocolo y puerto especificado)
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80

#Verificar global forwarding rule
gcloud compute forwarding-rules list

#Llamar al servicio
http://IP_DEL_SERVICIO
