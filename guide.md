# Overview
I am using a Linode instance running Ubuntu LTS 22. I have adjusted some settings depending on whether you run it on the cloud or on your own device.

## Installations

#### Step 1 (Skip to Step 2 if you have snap)
Since every tool I needed was in the Snap repository, the default installation on Ubuntu was fine. If you don't have Snap installed, you can do so using the following commands:

``` bash
sudo apt update && apt upgrade
sudo apt install snapd
```
#### Step 2 

``` bash
sudo snap install doctl
sudo snap install kubectl --classic
sudo snap install helm --classic
```

After the installation, you will need to create an API token so that your machine can talk to Digital Ocean. To do that, go to: [Digital Ocean API Tokens](https://cloud.digitalocean.com/account/api/tokens)

PS: If you haven't made an account yet, the link will send you to create an account. Click the link again and generate a new token with read and write privileges, then copy the token.

## Initialization
#### Step 1
Now, go to your terminal and type in:

``` bash
doctl auth init
```
You will get a prompt for the access token, paste the token here. If you get an error message about `mkdir /root/.config`, run the command:

#### Step 2
``` bash
mkdir /root/.config
```

#### Step 3
Add `sudo` if needed. Re-run `doctl auth init`, then paste the token.

``` bash
doctl auth init 
```

#### Step 4
Using kubectl requires the kube-config personal-files connection for doctl:
``` bash
sudo snap connect doctl:kube-config
```
## Setting Context
If you type in:

``` bash
kubectl config current-context
```
#### Step 1

You should get an error of 'not-set'. Go to the Digital Ocean website and create a Kubernetes cluster using the following settings:

- Server should be in Sydney.
- Version should be default.
- Then select autoscale for cluster capacity.
- Choose 8GB RAM, 4 vCPUs, and 160GB Storage for node plan.
- Choose a basic SSD for machine type.
- Then choose a min node of 1 and a max node of 5.
- Save the name (you can change it to `juice-k8s`, but I stuck to the default on mine).
- Then create the cluster.

#### Step 2
Back in the terminal, set the context with the following line:

``` bash
doctl kubernetes cluster kubeconfig save <cluster name>
```
#### Step 3
You will get an error `mkdir /root/.kube`. Run the command:

``` bash
mkdir /root/.kube
```
#### Step 4
Re-run the `doctl` command to set the context. Once that is done, try the next command to check if the context is set:

``` bash
kubectl config current-context
```
If you see the name you used, you can continue; if not, retry everything from cluster creation. from Step 1

## Installing Multijuicer
#### Step 1 
on this step you can either install the default helm settings or git clone the repo to edit values, if you use 
the default skip to `kubectl get pods` **change to the step**

Cloning the repo to change values

``` bash
git clone https://github.com/juice-shop/multi-juicer/tree/main
```
#### Step 2
Navigate to the file `value.yaml` found in `./multi-juicer/helm/multi-juicer/` and edit key values listen below


1. Cookie True flag for https

``` yaml
balancer:
  cookie:
    # SET THIS TO TRUE IF IN PRODUCTION
    # Sets secure Flag in cookie
    # -- Sets the secure attribute on cookie so that it only be send over https
    secure: false
    # -- Changes the cookies name used to identify teams. Note will automatically be prefixed with "__Secure-" when balancer.cookie.secure is set to `true`
    name: balancer
    # -- Set this to a fixed random alpha-numeric string (recommended length 24 chars). If not set this gets randomly generated with every helm upgrade, each rotation invalidates all active cookies / sessions requiring users to login again.
    cookieParserSecret: null
  repository: ghcr.io/juice-shop/multi-juicer/juice-balancer
  tag: null
```

1. Change `maxinstances` to the number of users attending the workshop
2. Change the `ctfKey` so that people dont just get flags from writeups

``` yaml
juiceShop:
  # -- Specifies how many JuiceShop instances MultiJuicer should start at max. Set to -1 to remove the max Juice Shop instance cap
  maxInstances: 10
  # -- Juice Shop Image to use
  image: bkimminich/juice-shop
  tag: v16.0.0
  # -- Change the key when hosting a CTF event. This key gets used to generate the challenge flags. See: https://pwning.owasp-juice.shop/companion-guide/latest/part4/ctf.html#_overriding_the_ctf_key
  ctfKey: "zLp@.-6fMW6L-7R3b!9uR_K!NfkkTr"
  # -- Specify a custom Juice Shop config.yaml. See the JuiceShop Config Docs for more detail: https://pwning.owasp-juice.shop/companion-guide/latest/part4/customization.html#_yaml_configuration_file
  # @default -- See values.yaml for full details
```

1. Change `gracePeriod` so that you dont have idle instances taking resources, usually 40 hr for a 1hr workshop  

``` yaml
juiceShopCleanup:
  repository: ghcr.io/juice-shop/multi-juicer/cleaner
  tag: null
  enabled: true
  # -- Specifies when Juice Shop instances will be deleted when unused for that period.
  gracePeriod: 1d
  # -- Cron in which the clean up job is run. Defaults to once in an hour. Change this if your grace period if shorter than 1 hour
  cron: "0 * * * *"
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  resources:
    requests:
      memory: 256Mi
    limits:
      memory: 256Mi
```

#### Step 3
then run: 
``` bash 
helm install -f values.yaml multi-juicer
```
OR {navigate to the directory that has values.yml and run}
``` bash
helm install -f values.yaml 
```
It will take a few minutes, but if it is pending for some time, run:

#### Step 4

run the command below to check if it has succesfully deployed 

``` bash
kubectl describe pods juice-balancer
```
#### Step 5
If you get an "insufficient CPU warning", re-create the Kubernetes cluster with the settings in Initalisation;
it doesn't have enough space. If everything is running, you can check it out on your machine.

## Running Multijuicer on your machine (To check if it works, don't run it for your workshop. You will need a balancer) 
#### Step 1
If you are running the configuration on your local machine, run:

``` bash
kubectl port-forward service/juice-balancer 3000:3000
```
Then go to [http://localhost:3000/](http://localhost:3000/) to access Multijucier and follow the prompts.

If you are running the configuration on cloud instance like me, run:

``` bash
kubectl port-forward service/juice-balancer 3000:3000 --address 0.0.0.0
```

Then go to [http://<server_ip>:3000/](http://<server_ip>:3000/). 
You might need to configure ports if the site is not coming up; run:

``` bash
sudo ufw allow 3000
```

Then re-run the port forward line.

Once you get onto the site, you can either log in as a user or an admin.

- User domain looks like [http://<server_ip>:3000/#/](http://<server_ip>:3000/#/). You won't be able to use the admin username and password here; it is just for users.

- The admin domain will look like [http://<server_ip>:3000/balancer/](http://<server_ip>:3000/balancer/).
  - Username for admin: admin
  - Password for admin: It will be mentioned on your machine,
  - but you will need to run on the configuration device your comupter or the server that got value.yaml:
    ```
    kubectl get secrets juice-balancer-secret -o=jsonpath='{.data.adminPassword}' | base64 --decode
    ```
## Setting up Balancer

#### Step 1
***need to finish this part but if you know how to set up a certificate this should be fine***


#### not complete
Get your digitalocean cert id
``` bash
doctl compute certificate list
```

Download edit the cert id in do-lb.yaml to the cert id of your domain
```bash
wget https://raw.githubusercontent.com/juice-shop/multi-juicer/main/guides/digital-ocean/do-lb.yaml
```

use your preferred text editor and edit `do-lb.yaml`

Create the loadbalancer
This might take a couple of minutes
``` bash
kubectl create -f do-lb.yaml
```
If it takes longer than a few minutes take a detailed look at the loadbalancer

``` bash
kubectl describe services multi-juicer-loadbalancer
```
