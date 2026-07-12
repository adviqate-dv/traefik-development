## Welcome to the Traefik container



### Things to Prep

- make yourself a password for the Dashboard so you can access it.  **Write down the password**


``` bash
htpasswd -nb admin "P@ssw0rd" | sed -e 's/\$/\$\$/g'

```

- paste the output into your docker compose file 

``` text
      - "traefik.http.middlewares.dashboard-auth.basicauth.users=<TheOutPutGoesHere>"
```




---

### Securing it with SSL


You have two options

#### Using a Self Signed Cert

Make a new cert for yourself in the command line in the "certs" folder

``` bash

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ./local.key -out ./local.crt \
  -subj "/CN=*.docker.localhost"

```

Make sure there is a "dynamic" folder and then

``` bash

cat > dynamic/tls.yml << EOF
tls:
  certificates:
    - certFile: /certs/local.crt
      keyFile: /certs/local.key
EOF

```

Under "volumes" in the compose

      # Add the following volumes
      - "./certs:/certs:ro"
      - "./dynamic:/etc/traefik/dynamic:ro"


Under "labels" in the compose
      # Add the following label
      - "traefik.http.routers.dashboard.tls=true"


#### Using Let's Encrypt or Your own CA

Make sure to remove all of the items that were setup in the self signed SSL setup, **including the folders in your project file**


Create a folder called "traefikfiles" in your project folder

Put a file called "acme.json" in it and chmod 600 the file

Under "volumes" in the compose


      # Map the acme.json database to your traefikfiles directory (Read/Write)
      - ./traefikfiles/acme.json:/traekikfiles/acme.json


Put the root CA certificate

Make sure there is a folder called "autocerts" in your project folder

Under "volumes" in the compose
      # Map your root CA certificate from the certs directory (Read-Only)
      - ./autocerts/root_ca.crt:/autocerts/root_ca.crt:ro

Under "commands"

``` text

      # ACME configuration
      # note: this is for when you want to use your own CA
      # Uncomment when you have a CA that's ready to issue certificates for your domain. If you want to use Let's Encrypt, you can comment these lines out and uncomment the Let's Encrypt lines below.

      # if you want to just use Let's Encrypt, change "smallstepCA" and change to "LetsEncrypt" to keep the naming clean
      - "--certificatesresolvers.smallstepCA.acme.email=your-email@example.com" # replace with your actual email
      - "--certificatesresolvers.smallstepCA.acme.storage=/traefikfiles/acme.json"
      - "--certificatesresolvers.smallstepCA.acme.httpchallenge.entrypoint=web"

      # if you are using your own CA you need to specify these things
      # Redirect Traefik to your Smallstep ACME provisioner endpoint
      - "--certificatesresolvers.smallstepCA.acme.caserver=https://yourdomain.local"

      # Tell Traefik to trust your self-signed Smallstep Root CA certificate
      - "--certificatesresolvers.smallstepCA.acme.cacertificates=/autocerts/root_ca.crt"

```


---

