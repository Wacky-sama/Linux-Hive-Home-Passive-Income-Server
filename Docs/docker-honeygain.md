# Running Honeygain with Docker

## Pulling the docker Image
```bash
docker pull image honeygain/honeygain
```

## Running Honeygain
```bash
## Get the current copy of ToU:
docker run honeygain/honeygain -tou-get

## Start Honeygain container by running:
docker run honeygain/honeygain -tou-accept -email "ACCOUNT_EMAIL" -pass "ACCOUNT_PASSWORD" -device "DEVICE_NAME"

## Checking Logs
docker logs -f honeygain

## Auto-start on Boot (Optional)
docker update --restart unless-stopped honeygain
```
