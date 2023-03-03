### Intro
We will deploy MLflow in VK Cloud using S3 as artifact store, DBaaS Postgres as backend entity storage and Tracking Server on separate host.   
This is the most production ready scenario of deployment   
https://mlflow.org/docs/latest/tracking.html#scenario-4-mlflow-with-remote-tracking-server-backend-and-artifact-stores


### 1. Create VM for mlflow
https://mcs.mail.ru/help/ru_RU/create-vm/vm-quick-create   
Tested with OS - Ubuntu 18.04 and Ubuntu 20.04  
Record VM white ip and internal ip

### 2. Install MLflow 
Login to VM that was created on Step 1   
You can access VM with command  
```console
ssh -i REPLACE_WITH_YOUR_KEY ubuntu@REPLACE_WITH_YOUR_VM_IP
```

```console
sudo pip install mlflow
sudo pip install boto3

sudo apt install gcc
sudo pip install psycopg2-binary
```

### 3. Create Postgres as a backend store
https://mcs.mail.ru/help/ru_RU/dbaas-start/db-postgres   
Save credentials, DB name, internal IP of DB

### 4. Create S3 bucket and directory
The next step is creating a directory for our Tracking Server to log the Machine Learning models and other artifacts. 
Remember that the Postgres database is only used for storing metadata regarding those models.
We will use S3 as artifact storage.

Create bucket and directory inside it  
https://mcs.mail.ru/help/ru_RU/s3-start/create-bucket

Create account, access key id and secret key   
https://mcs.mail.ru/help/ru_RU/s3-start/s3-account 


### 5. Launch MLflow
Login to VM that was created on Step 1   
You can access VM with command  
```console
ssh -i REPLACE_WITH_YOUR_KEY ubuntu@REPLACE_WITH_YOUR_VM_IP
```

Set env variables
```console
sudo nano /etc/environment

#Copy and paste this, replace with your values
MLFLOW_S3_ENDPOINT_URL=https://hb.bizmrg.com
MLFLOW_TRACKING_URI=http://REPLACE_WITH_INTERNAL_IP_MLFLOW_VM:8000
```
Log out, login to apply changes


Create file with S3 credentials
```console
mkdir .aws

nano ~/.aws/credentials
```
Copy and paste this in ~/.aws/credentials
```
[default]
aws_access_key_id = REPLACE_WITH_YOUR_KEY
aws_secret_access_key = REPLACE_WITH_YOUR_SECRET_KEY
```

This is test run, if you leave terminal MLflow will be unaccessible   
Permanent serving of MLflow on the next step   
```console
mlflow server --backend-store-uri postgresql://pg_user:pg_password@REPLACE_WITH_INTERNAL_IP_POSTGRESQL/db_name --default-artifact-root s3://REPLACE_WITH_YOUR_BUCKET/REPLACE_WITH_YOUR_DIRECTORY/ -h 0.0.0.0 -p 8000
```

### 6. Enable MLflow as a systemd service
Create dirs for logs and errors
```console
mkdir ~/mlflow_logs/
mkdir ~/mlflow_errors/
```

Create file with systemd service
```console
sudo nano /etc/systemd/system/mlflow-tracking.service
```

Copy and paste in /etc/systemd/system/mlflow-tracking.service
```console
[Unit]
Description=MLflow Tracking Server
After=network.target
[Service]
Environment=MLFLOW_S3_ENDPOINT_URL=https://hb.bizmrg.com
Restart=on-failure
RestartSec=30
StandardOutput=file:/home/ubuntu/mlflow_logs/stdout.log
StandardError=file:/home/ubuntu/mlflow_errors/stderr.log
User=ubuntu
ExecStart=/bin/bash -c 'PATH=/usr/bin/python3:$PATH exec mlflow server --backend-store-uri postgresql://PG_USER:PG_PASSWORD@REPLACE_WITH_INTERNAL_IP_POSTGRESQL/DB_NAME --default-artifact-root s3://REPLACE_WITH_YOUR_BUCKET/REPLACE_WITH_YOUR_DIRECTORY/ -h 0.0.0.0 -p 8000' 
[Install]
WantedBy=multi-user.target
```

Enable MLflow service
```console
sudo systemctl daemon-reload
sudo systemctl enable mlflow-tracking
sudo systemctl start mlflow-tracking
sudo systemctl status mlflow-tracking
```

Check that evrything is ok in logs
```console
head -n 95 ~/mlflow_logs/stdout.log 
head -n 95 ~/mlflow_errors/stderr.log
```


### 7. Install JupyterHub on VM from step 1
https://tljh.jupyter.org/en/latest/install/custom-server.html 

```console
sudo apt install python3 python3-dev git curl

#Replace <admin-user-name> with the name of the first admin user for this JupyterHub. Choose any name you like (**don’t forget to remove the brackets!**). 
curl -L https://tljh.jupyter.org/bootstrap.py | sudo -E python3 - --admin <admin-user-name>
``````
This will take 5-10 minutes, and will say Done! when the installation process is complete
Copy the Public IP of your server, and try accessing http://<public-ip> from your browser. If everything went well, this should give you a JupyterHub login page.

### 8. Launch JupyterHub and test MLflow
Your JupyterHub available on VM external IP   

Launch a terminal in Jupyter and clone the mlflow repo   
```console
git clone https://github.com/stockblog/webinar_mlflow/ webinar_mlflow
```
Open mlflow_demo.ipynb and launch cells

  
### The following steps are not necessary to complete the homework. You can do them as an extra practice.

### 9. Serve model from artifact store
Find URI of model in MLFlow UI   
Connect to MLflow host created on step 1   
```console
#EXAMPLE, REPLACE S3 path with your own path to model
mlflow models serve -m s3://mlflow_webinar/artifacts/5/a7ee769713974c118d0eff20226eb474/artifacts/model -h 0.0.0.0 -p 8001

mlflow models serve -m s3://BUCKET/FOLDER/EXPERIMENT_NUMBER/INTERNAL_MLFLOW_ID/artifacts/model -h 0.0.0.0 -p 8001

```

### 10. Serve model from the model registry
Register model in MLflow Models UI. Copy model name and paste it to example string
```console
#EXAMPLE
mlflow models serve -m "models:/diabet_test/Staging"

mlflow models serve -m "models:/YOUR_MODEL_NAME/STAGE"
```

### 11. Test model 
You may need to replace port from 8001 to default port 5001 if you did not set -p parameter in prev step

```console
#EXAMPLE, replace ip adress with internal ip of your mlflow host. You could also use white ip, set firewall settings 
curl -X POST -H "Content-Type:application/json; format=pandas-split" --data '{"columns":["age", "sex", "bmi", "bp", "s1", "s2", "s3", "s4", "s5", "s6"], "data":[[0.0453409833354632, 0.0506801187398187, 0.0606183944448076, 0.0310533436263482, 0.0287020030602135, 0.0473467013092799, 0.0544457590642881, 0.0712099797536354, 0.133598980013008, 0.135611830689079]]}' http://0.0.0.0:8001/invocations

curl -X POST -H "Content-Type:application/json; format=pandas-split" --data '{"columns":["age", "sex", "bmi", "bp", "s1", "s2", "s3", "s4", "s5", "s6"], "data":[[0.0453409833354632, 0.0506801187398187, 0.0606183944448076, 0.0310533436263482, 0.0287020030602135, 0.0473467013092799, 0.0544457590642881, 0.0712099797536354, 0.133598980013008, 0.135611830689079]]}' http://0.0.0.0:8001/invocations
```
