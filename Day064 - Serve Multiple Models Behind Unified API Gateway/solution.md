# Solution

A real platform serves many models, and clients shouldn't need to know where each one lives. A single API gateway fronts them: one public address that routes each request by URL prefix to the right model's container. This task wires a third model service into an nginx reverse-proxy fabric that already fronts two — adding its compose service and the nginx `upstream` + `location` block — then confirms each prefix reaches its own model.

> As an MLOps engineer, you put a gateway in front of multiple models so routing, addressing, and cross-cutting concerns live in one place — you are not comparing the models. The models and data are synthetic.

#### Follow the steps below

##### 1. Confirm the starting state.
From a VS Code terminal:
```
cd /root/code/serving/multi-model
ls
ls recommend/
cat docker-compose.yml
cat nginx.conf
```
All three service directories (`fraud/`, `churn/`, `recommend/`) exist with working `app.py` + `Dockerfile`. `docker-compose.yml` declares only `fraud`, `churn`, and `nginx`. `nginx.conf` routes only `/fraud/` and `/churn/`.

##### 2. Add the `recommend` service to `docker-compose.yml`.
Open `/root/code/serving/multi-model/docker-compose.yml` in the VS Code editor. Insert the `recommend` block next to the existing `churn` block, and add `recommend` to nginx's `depends_on`:
```yaml
services:
  fraud:
    build: ./fraud
    container_name: mm-fraud

  churn:
    build: ./churn
    container_name: mm-churn

  recommend:
    build: ./recommend
    container_name: mm-recommend

  nginx:
    image: nginx:alpine
    container_name: mm-nginx
    ports:
      - "8085:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - fraud
      - churn
      - recommend
```
Save.

##### 3. Add the `recommend` upstream + location to `nginx.conf`.
Open `/root/code/serving/multi-model/nginx.conf` and add the upstream alongside the existing ones, plus a matching `/recommend/` location:
```nginx
events {}

http {
    upstream fraud_backend {
        server fraud:5000;
    }

    upstream churn_backend {
        server churn:5000;
    }

    upstream recommend_backend {
        server recommend:5000;
    }

    server {
        listen 80;

        location /fraud/ {
            proxy_pass http://fraud_backend/;
        }

        location /churn/ {
            proxy_pass http://churn_backend/;
        }

        location /recommend/ {
            proxy_pass http://recommend_backend/;
        }
    }
}
```
Save.

##### 4. Bring the stack up.
```
docker compose up -d
docker compose ps
```
All four containers (`mm-fraud`, `mm-churn`, `mm-recommend`, `mm-nginx`) report `Up`. The first boot builds the `recommend` image (python:3.11-slim is already cached, so only Flask installs).

##### 5. Verify every route answers.
```
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"amount":3200,"hour":23,"num_tx_past_day":5}' \
  http://localhost:8085/fraud/predict

curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"tenure_days":30,"support_tickets":7}' \
  http://localhost:8085/churn/predict

curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"user_id":42}' \
  http://localhost:8085/recommend/predict
```
The three calls return JSON bodies with `"service": "fraud"`, `"service": "churn"`, and `"service": "recommend"` respectively — proof that nginx is routing each prefix to the right container.

#### References

- nginx reverse proxy — `upstream` blocks and `proxy_pass` (the routing added here): https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
- Compose file `services` reference — the `build` + `container_name` keys of the new service: https://docs.docker.com/reference/compose-file/services/
- `docker compose up` — building and starting the multi-service stack: https://docs.docker.com/reference/cli/docker/compose/up/
