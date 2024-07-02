# chat-sample

This project is a demo of how to host your own LLM-based chat service with two models: a locally hosted llama3 model, and a remotely hosted GPT 3.5 turbo model. The project provides a Docker compose file with required configuration artifacts to host such service. 

Below is an illustration of the architecture:

```mermaid
graph LR;
    A(User from\nWeb Browser) --> |HTTPS\n TCP 443|B[Nginx \n Web Proxy];
    subgraph Virtual Machine with GPU
    subgraph Docker 
    B -->|HTTP| C[Open Web UI]
    C -->|Ollama API| D[Ollama\n llama3 model\nhosted locally];
    C -->|Open AI API| E[LiteLLM\n Model Proxy];
    E -->|SQL| F[(PostgreSQL\n Database)];
    end
    D -.-> G[NVIDIA driver]
    end
    E -.->|Open AI API| H[Open AI Platform\ngpt-3.5-turbo model\nhosted remotely];
```
Note this project is for demo purpose only and is not considered production ready. For production-ready service deployment to implement scalability, container platform best practices, security, IdP integration, etc. Contact [Digi Hunch Inc.](https://www.digihunch.com/).


## Configure EC2 instance

We need to have an EC2 instance on a public subnet for the website to be publicly accessible. Alternatively, use SSH port forwarding for testing. Since we'll run ollama model on this instance, it is highly recommended that the instance has an NVIDIA GPU, and has sufficient disk space (Ollama3 needs about 7G to launch but let's provision 100G). For example, the `g4dn.xlarge` instance type comes with a single GPU (NVIDIA T4). We'll use Ubuntu operating system and install NVIDIA driver.

For testing, use `ami-04b70fa74e45c3917` in region `us-east-1` for Ubuntu 24.04. Since all components have releases in Docker images, we run the entire system in Docker and use Docker compose to orchestrate the services. This saves a lot of efforts from managing dependencies of each component install in Ubuntu, and provides the best portability.

## Install Prerequisites

Once the instance is launched, let's install Docker and NVIDIA driver.

To set up `apt` repository and install Docker community ediction, follow the [official guide](https://docs.docker.com/engine/install/ubuntu/). It is often easier to run Docker CLI commands as `root` user. To verify that docker compose has been installed, run the following command and it should return the version.
```sh
sudo -s
docker compose version

Docker Compose version v2.27.1
```

To install NVIDIA driver, run the following command:
```sh
sudo apt update
sudo apt install nvidia-driver-550 nvidia-dkms-550
```

In the command, 550 is the driver branch. The specific command may vary. Check out the `Manual driver installation` section on [this guide](https://ubuntu.com/server/docs/nvidia-drivers-installation) to figure out the correct command.

After the install, it requires a reboot to take effect. After the reboot, run the following command, which should return the driver version and CUDA version.
```sh
nvidia-smi

+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.171.04             Driver Version: 535.171.04   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla T4                       Off | 00000000:00:1E.0 Off |                    0 |
| N/A   33C    P8               9W /  70W |      5MiB / 15360MiB |      0%      Default |
|                                         |                      |                  N/A |
```

In addition to the driver, let's also install `nvidia-container-toolkit` following the [instruction](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt) to install with apt. Then configure the runtime and restart Docker daemon:

```sh
sudo nvidia-ctk runtime configure --runtime=docker

sudo systemctl restart docker
```
Note if you do not perform this step, you will come across this error when trying to bring up the model from docker:
`could not select device driver "nvidia" with capabilities: [[gpu]]`


## Bootstrap the application

First, download the files in this repository. 

```sh
git clone https://github.com/digihunch/chat-sample.git
cd chat-sample
```

In the `docker-compose.yaml` file, update the `OPENAI_API_KEY` environment variable under `litellm-proxy`. Then try to bring up the services (as root):
```sh
docker compose up
## Wait for the application bootstrap
```
Note, in this initial attempt, some services will fail. Finish the next few steps to complete the configuration.

## Configure Models
We need to configure two models. First, we need to get pull llama3 model which runs locally. Second, we need to get Open web UI to connect to gpt-3.5-turbo model.

To pull `llama3` model, we can run the `ollama` client from the container. 

```sh
docker exec -it ollama ollama pull llama3
docker exec -it ollama ollama list
```
With this, you should be able to pick `llama3` model from dropdown from the UI.

To configure access to `gpt-3.5-turbo` model (hosted remotely on openai), you must have an [OpenAI](https://platform.openai.com/) account and an [API key](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key). we already have a running proxy (litellm), we just need to tell litellm to create a key and provide this key to Open Web UI configuration.

To create a key:
```sh
curl 'http://0.0.0.0:4000/key/generate' \
--header 'Authorization: Bearer sk-liteLLM1234' \
--header 'Content-Type: application/json' \
--data-raw '{"models": ["gpt-3.5-turbo"], "metadata": {"user": "user@digihunch.com"}}'

```
From the response, take the value from key attribute, past it in as the value of environment variable `OPENAI_API_KEY` under `open-webui-svc`.


## Configure demo SSL key and certificate

Suppose the site name is `chatsample.digihunch.com`, create the demo certificate using the following command:
```sh
openssl req -x509 -sha256 -newkey rsa:4096 -days 365 -nodes -subj /C=CA/ST=Ontario/L=Waterloo/O=Digihunch/OU=Development/CN=chatsample.digihunch.com/emailAddress=chatsample@digihunch.com -keyout /home/ubuntu/chat-sample/nginx/certs/hostname-domain.key -out /home/ubuntu/chat-sample/nginx/certs/hostname-domain.crt
```
The command creates the `hostname-domain.key` and `hostname-domain.crt` file in the `/home/ubuntu/chat-sample/nginx/certs/` directory, which is referenced as relative path in the configuration in `nginx.conf` file included in the repo.

Then we restart docker compose to reflect the changes to litellm and nginx config:

```sh
docker compose down

docker compose up
```

Now the web site is up. You can either hit the server on public IP or port forwarding it to your client laptop's 8443 port like this:
```bash
ssh ubuntu@i-0ebba94b69620677e -L 8443:localhost:443
```


## Troubleshooting

### Ollama and llama3 model

In a step above we execute ollama from container to pull the llama3 model, we can also list models the same way:
```sh
docker exec -it ollama ollama pull llama3
```
Since we exposed Ollama locally on the server at host port 6789 (as declared in the Docker compose file), we can install ollama binary on the host OS then use it to configure

```sh
sudo curl -L https://ollama.com/download/ollama-linux-amd64 -o /usr/bin/ollama
sudo chmod +x /usr/bin/ollama
export OLLAMA_HOST=0.0.0.0:7869
ollama list
```

We can test the model with curl command:
```sh
curl http://localhost:7869/api/generate -d '{
  "model": "llama3",
  "prompt":"Why is the sky blue?"
}'
```

### The remote Open API model

To query for models available, you need the Open AI token to issue a curl command:
```sh
curl https://api.openai.com/v1/models\
      -H "Authorization: Bearer sk-proj-aabbccddeeffxxyyzziooospsabcefooxyzos"
```
The response contains available models. We can test a model as well with curl:
```sh
curl --location 'https://api.openai.com/v1/chat/completions' \
    --header "Authorization: Bearer sk-proj-aabbccddeeffxxyyzziooospsabcefooxyzos" \
    --header 'Content-Type: application/json' \
    --data '{
    "model": "gpt-3.5-turbo",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ]
}'
```
### LiteLLM

The LiteLLM service is also exposed on host port 4000 and also follows Open AI API format, we can check available model with the generated key:
```sh
curl http://localhost:4000/v1/models\
      -H "Authorization: Bearer sk-998877665544aabbcc"
```
We can test the model with
```sh
curl --location 'http://localhost:4000/v1/chat/completions' \
    --header "Authorization: Bearer sk-998877665544aabbcc" \
    --header 'Content-Type: application/json' \
    --data '{
    "model": "gpt-3.5-turbo",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ]
}'
```

If we run the curl command from a container within Docker, we'll need to use the host `litellm-proxy-svc:8000` instead.