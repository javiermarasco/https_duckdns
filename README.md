# Ingress controller con NGINX y cert-manager usando DuckDNS

Esta guía es una forma de configurar ingress controller y cert-manager (usando DuckDNS) para tener rápidamente (y gratis) una URL con HTTPS apuntando a nuestro cluster AKS donde podemos tener nuestras aplicaciónes expuestas a internet.
Esta guía la van a poder encontrar en forma de video en mi canal de YouTube (javi__codes) también. Mas links a mis otras redes sociales al final de la guía.

Ahora, empecemos:


## Pre requisitos y versiones:

- AKS cluster en version: 1.21.7
- Helm 3
- Ingress-controller nginx chart version 4.0.16
- Ingress-controller nginx app version 1.1.1
- cert-manager version 1.2.0
- cert-manager DuckDNS webhook version 1.2.2
  

## (1) Agregar helm repo de ingress nginx
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

## (2) Update de repos
```
helm repo update
```

## (3) Instalar nginx ingress-controller con helm

Esto va a instalar la ultima version disponible del chart en el repositorio.

```
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress --create-namespace 
```

## (4) Verificamos que los pods estén corriendo correctamente

```
kubectl get pods -n ingress
```

Deberían ver algo asi:

```
NAME                                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-ingress-nginx-controller-74fb55cbd5-hjvr9   1/1     Running   0          41m
```

## (5) Verificamos que nuestro ingress tiene una IP publica asignada

```
kubectl get svc -n ingress
```

Deberíamos ver algo asi, lo importante es que tengamos una IP asignada en "EXTERNAL-IP", a veces demora unos momentos en asignarnos una IP, lo que pasa por detrás es que Azure tiene que generar un recurso de tipo "Public IP" y asignarlo al cluster de AKS. Si no aparece una IP cuando ejecuten el comando, esperen un momento y vuelvan a intentarlo.

```
NAME                                               TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.0.33.214    20.190.211.14   80:32321/TCP,443:30646/TCP   38m
```

## (6) Desplegamos una aplicación de prueba 

Ahora vamos a desplegar una aplicación corriendo en un pod y un servicio que vamos a usar para acceder a los pods de esta aplicación. Si bien vamos a desplegar un único pod y tener un servicio para un único pod puede parecer algo sin sentido, piensen que ese pod puede ser eliminado y otro tomar su lugar, esto no siempre mantiene la IP interna del pod por lo que para acceder a el de forma directa tendríamos que estar constantemente actualizando la IP que usamos para acceder al pod en caso que este sea rescheduleado (borrado y otro creado en su lugar), un service evita justamente esto, nosotros siempre usamos el service para acceder al pod y no importa si tenemos 1, 10 o 70 pods, siempre vamos a usar el mismo service.

yaml de la aplicación de prueba:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-app
  namespace: default
spec:
  selector:
    matchLabels:
      app: echo-app
  replicas: 2
  template:
    metadata:
      labels:
        app: echo-app
    spec:
      containers:
      - name: echo-app
        image: hashicorp/http-echo
        args:
        - "-text=aplicación de prueba"
        ports:
        - containerPort: 5678
```

## (7) Desplegamos un service para nuestra aplicación

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-svc
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: echo-app
```

## (8) Ahora desplegamos un ingress resource 

En este paso vamos a desplegar un "ingress resource" esta es una forma de generar un ingress que va a decirle a nuestro ingress controller que el trafico que ingrese a nuestro cluster por la IP publica del ingress controller (la del paso 5 ) sea redireccionado a un servicio interno de nuestro cluster. Recuerden que el servicio de nuestra aplicación no tiene una IP publica asignada.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-echo
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: echo-svc
            port:
              number: 80

```

Aca le estamos diciendo a Kubernetes que cree un ingress resource para enviar todo el trafico ingresante por el ingress controller en el puerto 80 al servicio `echo-svc` en su puerto 80.

## (9) Probamos que todo funcione

Ahora podemos probar acceder a nuestro sitio usando la IP publica del ingress controller del punto 5.

Usando un web browser https://IP

Usando linea de comando con curl https://IP

# Agregando certificados con cert-manager y duckDNS

Bueno, hasta aca todo muy bien, pero nuestro ingress tiene una IP (no es lo mejor para exponer un servicio, es difícil de recordar) y ademas es todo http, no tenemos cifrado.

Veamos de agregar cert-manager, una solución que nos permite obtener certificados TLS para nuestros dominios web y rotarlos automáticamente.

## (10) Instalemos cert-manager

```
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.2.0 --set 'extraArgs={--dns01-recursive-nameservers=8.8.8.8:53\,1.1.1.1:53}' --create-namespace --set installCRDs=true
```

Luego de que termine de crear sus recursos, podemos verificar que los pods de cert-manager estén corriendo correctamente

```
kubectl get pods -n cert-manager

```

Deberíamos ver algo as

```
NAME                                            READY   STATUS    RESTARTS   AGE
cert-manager-6c9b44dd95-59b6n                   1/1     Running   0          47m
cert-manager-cainjector-74459fcc56-6dfn8        1/1     Running   0          47m
cert-manager-webhook-c45b7ff-hrcnx              1/1     Running   0          47m
```
## (11) Necesitas un dominio temporal y gratuito? DuckDNS al rescate

Hasta este punto, estamos listos para solicitar un certificado TLS para nuestro sitio, PERO tenemos que tener un dominio propio en internet para apuntarlo a la IP publica de nuestro ingress controller (Punto 5) y asi acceder a nuestros servicios usando un nombre de dominio (o subdominio).
Otro punto importante es que cert-manager solo provee certificados TLS si puede comprobar que nosotros somos los propietarios del dominio que queremos usar (para evitar darnos un certificado TLS de google.com por ejemplo), para hacer esto tiene dos formas, pero hoy vamos a hablar de una llamada `DNS-01`, en este modelo cert-manager va a generar un registro TXT en nuestro DNS con un valor aleatorio, luego de intentar crear este registro TXT, va a intentar leerlo, si lo puede leer significa que nosotros tenemos acceso a ese dominio (porque cert-manager pudo crear el record TXT), en ese momento cert-manager genera un certificado, elimina el registro TXT y almacena el contenido del certificado (y su key) en un secret en nuestro cluster de kubernetes.

Una vez terminado este proceso, podemos crear un ingress resource diciéndole que queremos que el trafico que ingrese desde ese dominio (y un path especifico) use un certificado (que tenemos en un secret) y reenvíe el trafico a un servicio (el de nuestro pod).

## (11) Configuremos nuestra cuenta de DuckDNS

Vamos a tener que ir a [https://www.duckdns.org/](https://www.duckdns.org/) y loguearnos con algunos de los métodos que están en la parte superior de la pagina.
Una vez hecho esto, vamos a ver que tenemos un TOKEN, ese es el que vamos a usar en el paso 12 de esta guía.

Tambien mas abajo en la pagina vamos a ver un lugar donde podemos generar un subdominio de duckdns.org y asignarle una IP. Esa IP (IPv4) tiene que ser la IP de nuestro ingress controller (la del paso 5 de esta guía). Una vez hecho eso guardamos los cambios clickeando en el botón de al lado de donde pusimos la IP (IPv4).

Con esto ya le dijimos a DuckDNS que todos los requests que vayan a ese subdominio que configuramos sean redireccionados a la IP que pusimos.

## (12) Desplegar el webhook handler de DuckDNS

Vamos a desplegar el webhook de DuckDNS, esta pieza es la que nos permite interactuar con DuckDNS. Este webhook tiene un helm chart que podemos usar, pero en mi caso no funciono, la forma que si me funciono fue clonando el repositorio y usando los archivos de ese repositorio.

Clonamos el repositorio
```
git clone https://github.com/ebrianne/cert-manager-webhook-duckdns.git
```

Instalamos desde el repositorio que clonamos
```
cd cert-manager-webhook-duckdns

helm install cert-manager-webhook-duckdns --namespace cert-manager --set duckdns.token='TOKEN_DE_DUCKDNS' --set clusterIssuer.production.create=true --set clusterIssuer.staging.create=true --set clusterIssuer.email='NUESTRO_MAIL' --set logLevel=2 ./deploy/cert-manager-webhook-duckdns
```

En este punto vamos a tener un nuevo pod en nuestro namespace `cert-manager`, lo podemos ver con el siguiente comando

```
kubectl get pods -n cert-manager
```
y se vería algo asi

```
NAME                                            READY   STATUS    RESTARTS   AGE
cert-manager-webhook-duckdns-5cdbf66f47-kgt99   1/1     Running   0          56m
```

## (12) ClusterIssuers y detalles de cert-manager

Cert manager maneja conceptualmente dos tipos de formas de generar certificados, tienen un generador de certificados llamado XXXX-Staging y otro llamado XXXX-Production. La principal diferencia es que el de Production nos va a dar un certificado valido que podamos usar en el mundo real, uno que nuestros navegadores van a reconocer mientras que el de staging va a generar un certificado que nuestros navegadores van a reconocer como https, pero van a marcarlo como "de prueba".

La idea es que hagamos todas las pruebas con el de Staging dado que si nos equivocamos y pedimos muchos certificados erróneos, nos vamos a tener problemas, mientras que si hacemos lo mismo en el de production, es muy probable que nos baneen por un tiempo determinado.

Cuando instalamos el webhook de DuckDNS especificamos que queríamos crear los clusterissues de production y staging.

```
--set clusterIssuer.production.create=true --set clusterIssuer.staging.create=true
```

## (13) Creamos un ingress resource usando el cluster issuer de staging

Vamos a crear un archivo llamado staging-ingress.yaml con este contenido

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-https-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: cert-manager-webhook-duckdns-staging
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - superprueba.duckdns.org
    secretName: superprueba-tls-secret-staging
  rules:
  - host: superprueba.duckdns.org
    http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80

```

En este ejemplo mi sub dominio en DuckDNS es `superprueba` y lo que estoy definiendo es que usemos el clusterissuer `cert-manager-webhook-duckdns-staging` y que guardemos el certificado en un secret llamado `superprueba-tls-secret` y que todo el trafico que ingrese por `superprueba.duckdns.org` https, lo envíe al servicio `echo` en su puerto 80.

El nombre del secret puede ser cualquier cosa, no tiene que contener el nombre del subdominio, pero es una buena practica que sea descriptivo y que sea claro para que se usa.

Un detalle importante es que este ingress resource tiene que estar en el mismo namespace que el servicio al cual estamos redireccionando el trafico. El ingress CONTROLLER puede (y recomiendo) estar en un namespace diferente al igual que cert-manager.

Ahora aplicamos con:

```
kubectil apply -f staging-ingress.yaml
```

## (14) Verificando el proceso de creación del certificado


Ahora si en nuestro namespace donde desplegamos el ingress resource y los pods/servicios hacemos un `kubectl get challenge` vamos a ver algo asi
```
NAME                                                        STATE     DOMAIN                       AGE
superprueba-tls-secret-staging-6lmxj-668717679-4070204345   pending   superprueba.duckdns.org      4s
```
Este es el proceso que realiza cert-manager para generar el record TXT en DuckDNS y comprobar que nosotros poseemos ese dominio/subdominio (que el token que indicamos es correcto), una vez que este proceso termina y se verifica que somos los propietarios de ese dominio/subdominio, este `challenge` desaparece y se genera un certificado que es almacenado en un secret con el nombre que nosotros indicamos en el ingress resource (`superprueba-tls-secret-staging` en nuestro caso).

Si vemos el estado de nuestro certificado mientras el `challenge` existe, vamos a ver algo asi:

```
NAME                             READY   SECRET                           AGE
superprueba-tls-secret-staging   False    superprueba-tls-secret-staging   7m15s
```
Y una vez terminado el proceso y cuando `challenge` desaparece, lo vemos asi:

```
NAME                             READY   SECRET                           AGE
superprueba-tls-secret-staging   True    superprueba-tls-secret-staging   7m15s
```

En este punto podemos verificar acceder a nuestro dominio `superprueba.duckdns.org` usando un navegador o con curl.

Con curl veriamos algo asi:

```
 curl https://superprueba.duckdns.org/
curl: (60) schannel: SEC_E_UNTRUSTED_ROOT (0x80090325) - The certificate chain was issued by an authority that is not trusted.
```
Esto es correcto, tenemos un certificado, pero no es un certificado "productivo", sino uno para comprobar que todo esta configurado correctamente en cert-manager y estamos listos para pedir un certificado definitivo.

## (15) Ajustando nuestro ingress resource para solicitar un certificado productivo

Ahora vamos a crear un archivo nuevo llamado `production-ingress.yaml` con el siguiente contenido:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-https-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: cert-manager-webhook-duckdns-production
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  tls:
  - hosts:
    - superprueba.duckdns.org
    secretName: superprueba-tls-secret-production
  rules:
  - host: superprueba.duckdns.org
    http:
      paths:
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: echo
            port:
              number: 80
```

Y por supuesto lo aplicamos con:

```
kubectl apply -f production-ingress.yaml
```

Una vez hecho esto, podemos aplicar los mismos pasos de antes para revisar que todo esta bien y por ultimo comprobamos con nuestro navegador o con curl si nuestro sitio https ya esta funcionando, si todo salio bien, tendríamos que ver algo asi:

```
curl https://superprueba.duckdns.org/
aplicación de prueba
```

## (16) Troubleshooting

Hasta aca todo el camino feliz, donde todo funciono perfectamente y no tuvimos ningún problema, no tuvimos errores de tipeo, pusimos las IPs correctas en todos lados y no confundimos los nombres de los secrets (no es que me haya pasado alguna vez, absolutamente no ...). Pero que pasa cuando algo asi nos sucede? como hacemos para poder identificar donde esta el problema? Bueno, aca mis recomendaciones (las que me ayudaron en todas las veces que NO me paso de tener algo mal configurado)

a) Revisar los logs de los pods de cert-manager webhook. En el del webhook de DuckDNS vamos a ver todos los pasos que esta haciendo cert-manager contra duckDNS y podemos ver si nuestro TOKEN por ejemplo esta mal, o si duckDNS no esta contestando correctamente los requests.

b) Revisar los logs del pod de cert-manager, el que se llama simplemente `cert-manager-XXXX`, este nos va a mostrar información sobre lo que esta haciendo cert-manager, este pod nos va a indicar de la creación o modificación del secret donde va a estar el certificado, el webhook solo se encarga de la comunicación y verificación con DuckDNS, pero este pod se encarga del trabajo dentro de nuestro cluster de kubernetes.

c) Verificar el log de ingress-controller: Aca vamos a ver cuando un request llegue a nuestro ingress controller, de que IP viene, si hay algún error en el request, básicamente si algo esta mal configurado en el ingress, aca es donde vamos a ver que esta pasando.

d) usar una herramienta para verificar que la IP que pusimos en DuckDNS esta apuntando a la IP publica de nuestro ingess-controller, una que uso yo es [https://digwebinterface.com/](https://digwebinterface.com/) en esta herramienta podemos comprobar si realmente el cambio que hicimos en DuckDNS esta correctamente configurado.

## Notas finales

Como siempre, si esta guía les sirvió o la ven interesante les agradecería mucho que la compartan con todas las personas que puedan, cuanta mas gente la vea mas posible es que me hagan llegar recomendaciones de como mejorar mis próximos artículos o si hay algún error en este y me alientan a seguir escribiendo y compartiendo con el resto de la comunidad.
Esta guía la voy a convertir en un video y subirlo a mi canal de YouTube, los veo por ahi (de paso... subscríbanse, denle like y dejen un comentario si les gusta el video, todo eso me ayuda)

Aprovecho a dejarles links a mis redes sociales y formas de contacto:

Twitter -> @javi__codes
Instagram -> javi__codes
LinkedInd -> javiermarasco
Youtube -> javi__codes
