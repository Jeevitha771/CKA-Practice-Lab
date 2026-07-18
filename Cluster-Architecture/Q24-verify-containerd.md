# Q24. Verify containerd is Properly Installed

## Task
Check containerd is properly installed:
- Check service status
- Check version
- Test image pull with crictl
- List running containers

---

## Step 1: Check containerd Service Status

```bash
systemctl status containerd
```

Expected:
```
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled)
     Active: active (running) since ...
```

If not running:
```bash
systemctl start containerd
systemctl enable containerd
```

---

## Step 2: Check containerd Version

```bash
containerd --version
# containerd containerd.io 1.7.6 091922f8...

crictl version
# Version:  0.1.0
# RuntimeVersion: 1.7.6
# RuntimeApiVersion: v1
```

---

## Step 3: Configure crictl to Use containerd

```bash
cat /etc/crictl.yaml
```

If missing or wrong:
```bash
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

---

## Step 4: Test Image Pull with crictl

```bash
crictl pull nginx:latest
# Image is up to date for docker.io/library/nginx:latest

# List all locally cached images
crictl images
# IMAGE                     TAG       IMAGE ID       SIZE
# docker.io/library/nginx   latest    abc123def456   56.9MB
```

---

## Step 5: List Running Containers

```bash
crictl ps
# CONTAINER  IMAGE    CREATED   STATE    NAME     ATTEMPT  POD ID
# a1b2c3d4   nginx    2m ago    Running  nginx    0        pod-xyz

# List all containers including stopped
crictl ps -a
```

---

## Verify containerd CGroup Driver

```bash
# Check containerd cgroup driver (should match kubelet)
cat /etc/containerd/config.toml | grep SystemdCgroup
# SystemdCgroup = true

# If missing, add systemd cgroup support
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

---

## Quick Health Check

```bash
# All-in-one verification
echo "=== Service Status ==="
systemctl is-active containerd

echo "=== Version ==="
containerd --version

echo "=== Socket ==="
ls -la /var/run/containerd/containerd.sock

echo "=== Images ==="
crictl images | head -5
```
