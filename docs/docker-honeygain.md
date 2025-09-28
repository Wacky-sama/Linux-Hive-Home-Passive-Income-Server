# Running Honeygain with Docker and setting up the folder

## Step 1: Create folder
```bash
sudo mkdir -p /srv/docker/honeygain
sudo chown $USER:$USER /srv/docker/honeygain
```

## Step 2: Pulling the docker Image
```bash
docker pull honeygain/honeygain
```

## Running Honeygain
```bash
# Get the current copy of Terms of Use(ToU):
docker run honeygain/honeygain -tou-get

# Start Honeygain container by running:
docker run -d --name honeygain -v /srv/docker/honeygain:/data honeygain/honeygain -tou-accept -email "ACCOUNT_EMAIL" -pass "ACCOUNT_PASSWORD" -device "DEVICE_NAME"
```
**Note:** Replace **ACCOUNT_EMAIL**, **ACCOUNT_PASSWORD**, and **DEVICE_NAME** with your own details.

## Checking Logs
```bash
docker logs -tf honeygain
```

## Auto-start on Boot (Recommended)
```bash
docker update --restart unless-stopped honeygain
```