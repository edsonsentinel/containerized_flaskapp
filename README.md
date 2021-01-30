# Containerize

## Containerize a Python application

1 - Build the Docker containers by running the following commands:
    
<b>nginx container:</b>
    
   Uses nginx:alpine to reduce footprint with minimal alpine image and copies nginx config to image
    
   We use a docker volume via docker-compose to copy the SSL certs to the container but not embed it into the image
   

    
        cd /containerize/nginx
        docker build -t ad-hoc-nginx .
        docker images (to verify it was created)
    
   <b>app container:</b>
    
   Uses python:3.6.1-alpine to have python-pip out the box and reduce footprint with minimal alpine image

    
        cd /containerize/app
        docker build -t ad-hoc-app .
        docker images (to verify it was created)

2 - Run "Production" app by itself (detached):

        docker run -d -p 8000:8000 ad-hoc-app
        docker container ls (to verify container is running)
    
   you should see something similar to:
```Text
      CONTAINER ID   IMAGE        COMMAND              CREATED         STATUS         PORTS                    NAMES
      e75e57584042   ad-hoc-app   "python server.py"   6 seconds ago   Up 5 seconds   0.0.0.0:8000->8000/tcp   frosty_faraday
```

Verify environment is production and port 8000 by running 'docker logs containerid':
     
     docker logs e75e57584042
     Output:
      * Serving Flask app "server" (lazy loading)
      * Environment: production
        WARNING: This is a development server. Do not use it in a production deployment.
        Use a production WSGI server instead.
      * Debug mode: on
      * Running on http://0.0.0.0:8000/ (Press CTRL+C to quit)
      * Restarting with stat
      * Debugger is active!
      * Debugger PIN: 191-157-902
     

3 - Run "Development" app via Docker Compose:

To start both containers run:

    docker-compose up -d
    docker container ls
   
   You should see:
```Text
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                      NAMES
56d355a107c0   ad-hoc-app     "flask run --host=0.…"   4 seconds ago   Up 2 seconds   0.0.0.0:8000->8000/tcp                     containerize_app_1
66a687cfac7e   ad-hoc-nginx   "/docker-entrypoint.…"   4 seconds ago   Up 3 seconds   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   containerize_app_2
```

Verify environment is development and port 8000 by running 'docker logs containerid':
```Text
     docker logs 56d355a107c0
     Output:
     * Serving Flask app "./server.py" (lazy loading)
     * Environment: development
     * Debug mode: on
     * Running on http://0.0.0.0:8000/ (Press CTRL+C to quit)
     * Restarting with stat
     * Debugger is active!
     * Debugger PIN: 454-023-757
```

After running `docker-compose up` run  `curl -k https://localhost/` should see output similar to:

```Text
It's easier to ask forgiveness than it is to get permission.
X-Forwarded-For: 10.5.0.1
X-Real-IP: 10.5.0.1
X-Forwarded-Proto: https
```

The app container is only reachable from other containers as port 8000 is exposed but not published
    See docker-compose.yml
    
    ports:
      - 8000
      
4 - To stop both containers via docker-compose run:
    
    docker compose down    
   
To stop single run app started via docker run:
    
    docker container ls
        copy Contained ID # for associated container
    docker container stop containerId#

5 - Nginx.conf explained:



See comments:

```Text
http {
    #Enabling sendfile to skip copying data into buffer - for performance optimization
    sendfile on;

    upstream flask-app {
        server 10.5.0.10:8000;
    }


    #ssl protocols defined - no outdated TLS
    ssl_protocols TLSv1.2 TLSv1.3;

    #SSL cert location defined
    ssl_certificate /etc/nginx/ssl/localhost.crt;
    ssl_certificate_key /etc/nginx/ssl/localhost.key;

    #add optimized SSL session cache
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 1h;

    #turn on cipher for encryption, add cipher suite
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:ECDH+AES256-CBC:ECDH+AES128-CBC:DH+3DES:!ADH:!AECDH:!MD5;

    #enable HSTS
    add_header Strict-Transport-Security "max-age=63072000";


    #redirect port 80 to port 443
    server{
        listen 80 default_server;
        server_name _;
        return 301 https://$host$request_uri;
    }

    #listen on port 443
    server {

             listen 443 ssl default_server;
             server_name app;

             location / {
                 #send traffic to upstream
                 proxy_pass         http://flask-app;
                 proxy_redirect     off;

                 #set headers
                 proxy_set_header   X-Real-IP $remote_addr;
                 proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_set_header   X-Forwarded-Proto $scheme;


                 #remove version from header for security
                 server_tokens off;

                 #restrict from network deny rest - in production add production subnet address for LB or remove if directly accessible
                 allow 10.5.0.0/24;
                 deny all;


             }
         }
}
```

6 - Validation tool results
```shell
./validate.sh

Going to remove containerize_app_1, containerize_app_2
------------------------------------------------------------------------
| Testing SSL/TLS settings...                                          |
------------------------------------------------------------------------

 Start 2021-01-30 18:42:46        -->> 127.0.0.1:443 (localhost) <<--

 A record via:           /etc/hosts 
 rDNS (127.0.0.1):       localhost.
 Service detected:       HTTP


 Testing protocols via sockets except NPN+ALPN 

 SSLv2      not offered (OK)
 SSLv3      not offered (OK)
 TLS 1      not offered
 TLS 1.1    not offered
 TLS 1.2    offered (OK)
 TLS 1.3    offered (OK): final
 NPN/SPDY   http/1.1 (advertised)
 ALPN/HTTP2 http/1.1 (offered)

 Testing cipher categories 

 NULL ciphers (no encryption)                      not offered (OK)
 Anonymous NULL Ciphers (no authentication)        not offered (OK)
 Export ciphers (w/o ADH+NULL)                     not offered (OK)
 LOW: 64 Bit + DES, RC[2,4], MD5 (w/o export)      not offered (OK)
 Triple DES Ciphers / IDEA                         not offered
 Obsoleted CBC ciphers (AES, ARIA etc.)            not offered
 Strong encryption (AEAD ciphers) with no FS       not offered
 Forward Secrecy strong encryption (AEAD ciphers)  offered (OK)


 Testing HTTP header response @ "/" 

 HTTP Status Code             200 OK
 HTTP clock skew              0 sec from localtime
 Strict Transport Security    730 days=63072000 s, just this domain
 Public Key Pinning           --
 Server banner                nginx
 Application banner           --
 Cookie(s)                    (none issued at "/")
 Security headers             --
 Reverse Proxy banner         --



 Done 2021-01-30 18:42:53 [   8s] -->> 127.0.0.1:443 (localhost) <<--


------------------------------------------------------------------------
| Testing HTTP header and body content...                              |
------------------------------------------------------------------------
Pass: status code is 200
Pass: X-Forwarded-For is present and not 'None'
Pass: X-Real-IP is present and not 'None'
Pass: X-Forwarded-Proto is present and not 'None'
Pass: found "It's easier to ask forgiveness than it is to get permission." in response
------------------------------------------------------------------------
| Testing hot reload for local development...                          |
------------------------------------------------------------------------
Pass: found "It's harder to ask forgiveness than it is to get permission." in response
------------------------------------------------------------------------
| Docker image sizes:                                                  |
------------------------------------------------------------------------
Going to remove containerize_app_1, containerize_app_2
```

