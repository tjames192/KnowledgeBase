# use Ollama, Open-WebUI and Agent Zero in podman using pods ..

```
# pod port 11434 is ollama
# pod port 8080 is open-webui

podman pod create -p 11434:11434 -p 3000:8080 --name ollamapod
```

## ollama pod

```
# ollama
# not sharing storage between pods so Z not z
podman run -d \
	--pod ollamapod \
	-v ollama-data:/root/.ollama:Z \
	-e OLLAMA_BASE_URL=http://ollama:11434 \
	-e NVIDIA_VISIBLE_DEVICES=all \
	-e NVIDIA_DRIVER_CAPABILITIES=compute,utility \
	--security-opt label=disable \
	--device nvidia.com/gpu=all \
	--health-cmd 'ollama list' \
	--health-interval 30s \
	--health-timeout 10s \
	--health-retries 3 \
	--health-start-period 40s \
	--restart=unless-stopped \
	--name ollama \
	docker.io/ollama/ollama:latest
```

## open-webui pod

```
# open-webui 
# not sharing storage between pods so Z not z
podman run -d \
	--pod ollamapod \
	-v open-webui-data:/app/backend/data:Z \
	-e OLLAMA_BASE_URL=http://ollama:11434 \
	-e WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY:-secret-key-change-me-in-production} \
	-e ENABLE_SIGNUP=${ENABLE_SIGNUP:-true} \
	--restart=unless-stopped \
	--name open-webui \
	ghcr.io/open-webui/open-webui:latest
```

## agent 0

```
podman run -d \
	--pod ollamapod \
	-v agent0-data:/a0/usr:Z \
	--name agent-zero \
	agent0ai/agent-zero:latest
```

## modify pod ports
```
podman pod stop ollamapod
podman pod rm ollamapod
# pod port 11434 is ollama
# pod port 8080 is open-webui
# pod port 80 is agent0
podman pod create -p 50080:80 -p 11434:11434 -p 3000:8080 --name ollamapod
```

## a gotcha while using pasta networking vs host in podman
in order for containers to uses localhost resources.
```
http://host.containers.internal:port
```
