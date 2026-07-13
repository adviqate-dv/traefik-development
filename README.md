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


Go into the traefik.yaml file in the root of your project folder

and uncomment the "Automated Certificate Engine"

Make sure you edit the variables to point to the right server


---



### Adding Containers to Your Proxy


You need to make sure that the container has the same network as your proxy server, so it can see it

! Important
If you are using a self signed cert for testing, you MUST comment out the "certresolver" line!

``` text

    networks:
      - traefik_proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.traefik.traefik-development.orb.local`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=smallstepCA"

```

Here is the step-by-step breakdown of what each label does for your whoami application container:
#### 1. The activation switch

* "traefik.enable=true"
* What it does: This signals Traefik to explicitly monitor this container. Because you have exposedByDefault: false configured, Traefik will ignore this container entirely unless this label is present.

#### 2. The routing rule

* "traefik.http.routers.whoami.rule=Host(\whoami.traefik.traefik-development.orb.local`)"`
* What it does: This creates a new Router named whoami and assigns it a rule. It tells Traefik: "If a web request comes in and the browser's URL address exactly matches whoami.traefik.traefik-development.orb.local, intercept it and process it through this router."

#### 3. The entrypoint constraint

* "traefik.http.routers.whoami.entrypoints=websecure"
* What it does: This restricts where this specific router listens. It binds the rule exclusively to your websecure entrypoint (Port 443 / HTTPS). Traffic on this domain hitting unencrypted port 80 will bypass this rule and hit your global HTTPS redirect instead.

#### 4. Enabling encryption

* "traefik.http.routers.whoami.tls=true"
* What it does: This activates TLS/HTTPS encryption for this web traffic stream. It tells Traefik that it must handle a secure cryptographic handshake with the user's browser before forwarding any data to the underlying application.

#### 5. The certificate solver

* "traefik.http.routers.whoami.tls.certresolver=smallstepCA"
* What it does: This specifies how Traefik gets the TLS certificate for this domain. It tells Traefik: "Do not look at local files for this domain. Instead, hand this domain name over to the automated smallstepCA certificate resolver we configured. Go fetch a 35-day certificate from our private Smallstep container over the network, save it in acme.json, and renew it automatically when it gets close to expiring."

------------------------------
####  What happens behind the scenes for this specific setup:
Because you didn't define an explicit service label (like your dashboard did with service=api@internal), Traefik uses its built-in automation. It checks the container, discovers the whoami image runs on port 80, automatically builds a backend service, and maps your secure router right to it.
If you are ready to test this entire loop, would you like help setting up a quick curl test command that skips local browser trust warnings so you can verify the automated certificate was successfully issued?



