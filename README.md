# chat-sample
Sample app 


## Configure EC2 instance

In this step, we need to have an EC2 instance on a public subnet for the website to be publicly accessible. Alternatively, use SSH port forwarding for testing. Since we'll run ollama model on this instance, it is highly recommended that the instance has an NVIDIA GPU, and has sufficient disk space (Ollama3 needs about 7G to launch but let's provision 100G). We'll use Ubuntu operating system and install NVIDIA driver.

For example, the `g4dn.xlarge` instance type comes with a single GPU (NVIDIA T4)

For testing, use `ami-04b70fa74e45c3917` in region us-east-1 for Ubuntu 24.04. We also run the system in Docker. This is because there are several components all with releases in Docker image, and I find these Docker image releases work better than their PIP package releases. All we need is a compose file to port it to a different server.


## Install prerequisites

Once the instance is launched, install Docker and NVIDIA driver.

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

In addition to the driver,



also install `nvidia-container-toolkit` following the [instruction](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt) to install with apt. Then configure the runtime and restart Docker daemon:

```sh
sudo nvidia-ctk runtime configure --runtime=docker

sudo systemctl restart docker

```
Note if you do not perform this step, you will come across this error when trying to bring up the model from docker:
`could not select device driver "nvidia" with capabilities: [[gpu]]`




## Bring up the application.

First, download the files in this repository. 

```sh
git clone https://github.com/digihunch/chat-sample.git
cd chat-sample
```

In the `docker-compose.yaml` file, update the `OPENAI_API_KEY` environment variable under `litellm-proxy`. Then try to bring up the services (as root):
```sh
docker compose up
## Wait for 
```
Note, even though all services are up, we still have to configure model access.

## Configure models
We need to configure two models. First, we need to get pull llama3 model which runs locally. Second, we need to get Open web UI to connect to gpt-3.5-turbo model.

To pull llama3 model, we use ollama from the container. 

```sh
docker exec -it ollama ollama pull llama3
docker exec -it ollama ollama list
```
With this, you should be able to pick Llama3 model from dropdown.


To configure access to gpt-3.5-turbo model (hosted on openai), we already have a running proxy (litellm), we just need to tell litellm to create a key and give this key to open web ai.

To create a key:
```sh
curl 'http://0.0.0.0:4000/key/generate' \
--header 'Authorization: Bearer sk-liteLLM1234' \
--header 'Content-Type: application/json' \
--data-raw '{"models": ["gpt-3.5-turbo"], "metadata": {"user": "ylu@test.com"}}'

```
From the response, take the value from key attribute, past it in as the value of environment variable `OPENAI_API_KEY` under open-webui-svc.


## Configure demo SSL key and certificate

Suppose the site name is chatsample.digihunch.com, create the demo certificate using the following command:
```sh
openssl req -x509 -sha256 -newkey rsa:4096 -days 365 -nodes -subj /C=CA/ST=Ontario/L=Waterloo/O=Digihunch/OU=Development/CN=chatsample.digihunch.com/emailAddress=chatsample@digihunch.com -keyout /home/ubuntu/chat-sample/nginx/certs/hostname-domain.key -out /home/ubuntu/chat-sample/nginx/certs/hostname-domain.crt
```
The command creates the `hostname-domain.key` and `hostname-domain.crt` file in the `/home/ubuntu/chat-sample/nginx/certs/` directory, which is referenced as relative path in the configuration in `nginx.conf` file included in the repo.

Then we restart docker compose to reflect the changes to litellm and nginx config:

```sh
docker compose down

docker compose up
```


