# SheriffScanner

This is where my development progress will be shared for using machine learning and LIDAR to detect other FRC robots during a match. This is one stepping stone to one day having a fully AI-strategised and driven match. It all starts with one thing: robot localization of not just your own robot (that's easy) but everyone around you in high speed matches.

The scanner reads an [RPLidar C1](https://www.amazon.com/dp/B0DDKZV8HZ?lv=shuf&channelId=500&plpRedirect=mhFallback), clusters the points into targets with DBSCAN, and publishes their poses to NetworkTables (`LidarProcessor/poses`).

## One-time setup on the Orange Pi

Install Docker and make sure it starts at boot:

```bash
sudo systemctl enable --now docker
```

Find the LIDAR's serial device (Note: the driver comes preinstalled on the Orange Pi 5 Debian Bookworm image):

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

Use that path for `--device` below (this README assumes `/dev/ttyUSB0`).

## Step 1: Get the image onto the Pi

Use this step for both the first install and every later update. Pick **one** option.

### Option A: Pi has internet (pull directly from GHCR)

Run on the Pi:

```bash
docker pull ghcr.io/hjmcphail76/sheriffscanner/sheriff-scanner:latest
```

### Option B: Pi has no internet, push over SSH from a connected device

Run on an Internet-connected device. First pull the ARM64 image:

```bash
docker pull --platform linux/arm64 ghcr.io/hjmcphail76/sheriffscanner/sheriff-scanner:latest
```

Then send it to the Pi (replace `<pi-ip>` with the Pi's IP address):

```bash
docker save ghcr.io/hjmcphail76/sheriffscanner/sheriff-scanner:latest | gzip | ssh orangepi@<pi-ip> "gunzip | docker load"
```

### Option C: Pi has no internet, load a tarball (no Docker needed on your laptop)

1. Open the finished run in the repo's **Actions** tab and download the `scanner-image` artifact. Unzip it to get `scanner.tar.gz`.
2. Copy it to the Pi, with `scp` or a USB stick:

   ```bash
   scp scanner.tar.gz orangepi@<pi-ip>:~
   ```

3. On the Pi, load the image:

   ```bash
   gunzip -c scanner.tar.gz | docker load
   ```

## Step 2: Start (or replace) the container

Run on the Pi. This removes any existing `scanner-runner` container (it's fine if there isn't one) and starts a new one from the image you just loaded. The container restarts automatically on every boot.

```bash
   docker run -d --name scanner-runner --restart unless-stopped \
     --privileged -v /dev:/dev -p 8080:8080 -e SDL_VIDEODRIVER=dummy \
     ghcr.io/hjmcphail76/sheriffscanner/sheriff-scanner:latest
```



Optionally, free disk space by removing the old image left behind after an update:

```bash
docker image prune -f
```

Check that it's healthy:

```bash
docker ps
docker logs --tail 30 scanner-runner
docker inspect -f 'policy={{.HostConfig.RestartPolicy.Name}} restarts={{.RestartCount}}' scanner-runner
```

A restart count that keeps climbing means it is crash-looping; the logs will say why.

## Updating to a new build

Wait for the GitHub Action to finish (watch the **Actions** tab), then repeat [Step 1](#step-1-get-the-image-onto-the-pi) and [Step 2](#step-2-start-or-replace-the-container).

## Optional: automatic updates

If the Pi has a reliable internet connection, Watchtower can watch GHCR and swap the container when a new image appears. Complete Step 1 and Step 2 first so the container exists:

```bash
docker run -d --name watchtower --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower --interval 60 scanner-runner
```

## Cleanup

Remove all containers (this stops the scanner until you start a new one):

```bash
docker rm -f $(docker ps -aq)
```

Remove unused images to free disk space:

```bash
docker image prune -a
```
