					                             Docker & Kubernetes
Well containerisation had been there for a while and this time I planned to get my hands dirty
Docker  is a tool designed to make it easier to create, deploy, and run applications by using containers. Containers allow a developer to package up an application with all of the parts it needs, such as libraries and other dependencies, and ship it all out as one package. By doing so, thanks to the container, the developer can rest assured that the application will run on any other Linux machine regardless of any customized settings that machine might have that could differ from the machine used for writing and testing the code.In a way, Docker is a bit like a virtual machine. But unlike a virtual machine, rather than creating a whole virtual operating system, Docker allows applications to use the same Linux kernel as the system that they're running on and only requires applications be shipped with things not already running on the host computer. This gives a significant performance boost and reduces the size of the application.

Kubernetes: (https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation.
Kubernetes provides you with:
•	Service discovery and load balancing
Kubernetes can expose a container using the DNS name or using their own IP address. If traffic to a container is high, Kubernetes is able to load balance and distribute the network traffic so that the deployment is stable.
•	Storage orchestration
Kubernetes allows you to automatically mount a storage system of your choice, such as local storages, public cloud providers, and more.
•	Automated rollouts and rollbacks
You can describe the desired state for your deployed containers using Kubernetes, and it can change the actual state to the desired state at a controlled rate. For example, you can automate Kubernetes to create new containers for your deployment, remove existing containers and adopt all their resources to the new container.
•	Automatic bin packing
Kubernetes allows you to specify how much CPU and memory (RAM) each container needs. When containers have resource requests specified, Kubernetes can make better decisions to manage the resources for containers.
•	Self-healing
Kubernetes restarts containers that fail, replaces containers, kills containers that don’t respond to your user-defined health check, and doesn’t advertise them to clients until they are ready to serve.
•	Secret and configuration management
Kubernetes lets you store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. You can deploy and update secrets and application configuration without rebuilding your container images, and without exposing secrets in your stack configuration




.. I tried to break down my journey into small tasks so I can keep learning and exploring various features of docker and kubernetes.


First installation on windows was kind of nightmare.

Need to have bios setting changed to Hyper-V to have this work .
Make sure your account is part of Docker Users group on the machine.
Click on  Docker Desktop, once it starts without any error enable the kubernetes for the docker
 

Kubecti is the utility to deal with Kubernetes 

First thing check if the configuration is setup correctly
I was trying this command kubectl get nodes in power-shell and expecting it to return me the node which is ready to serve our jobs, but since “every day may not be a good day but there is something good in every day”.. so I went to google help 
It says my configuration path is not setup .., so I setup a environment variable for current user,  KUBECONFIG =C:\Users\<username>\.kube\config [Replace <username> with your username]

[Output of command kubectl config view before  environment variable is setup

apiVersion: v1
clusters: []
contexts:
- context:
    cluster: ""
    user: ""
  name: my-context
current-context: ""
kind: Config
preferences: {}
users: []

Now reopen the instance of power-shell so it sources the new environment variable and setup will look better use the same command kubectl config view, we will see the clusters and context setup correctly.


Okay now we are ready to go…………..and meet our next hurdle and try to find solution to the problem…
Use case on which I would be working is to have some batch jobs and use very vanilla based scheduling 1000 of jobs where each job will have definitive input and output. We can use Radis queue etc to distribute the workload but for this demo we will keep it limited to use arguments based.

1.	Create simple python job with some third party dependency say pandas and see how we can run as a docker image and then put it as pod 

a.	Create this file as test.py in docker-sample1 folder

import pandas as pd

# initialize list of lists 
data = [['A', 10], ['B', 15], ['C', 14]] 
  
# Create the pandas DataFrame 
df = pd.DataFrame(data, columns = ['Name', 'Age']) 
  
# print dataframe. 
print(df)

  Run this file on your command prompt/power-shell and see it works fine and produce the desired result.

Name  Age
0    A   10
1    B   15
2    C   14

b.	Create requirements.txt file with pandas dependency, output of the file will look like
pandas==0.24.2

c.	Let’s create docker file in the same folder, few things to note that file name will be dockerfile
So I am trying to copy all the files to home directory, ideally we can create folder using mkdir and copy the files to that folder to keep it cleaner. 

FROM python
COPY . /home
RUN ls /home
RUN pip install -r /home/requirements.txt
CMD python /home/test.py

d.	In our powershell let’s go this folder “docker-sample1” where all above 3 files are present and run this command docker build -t testimg .
This command builds the docker image by downloading all the dependencies .

Let’s see if our docker image appears in the list of docker images.
# list all docker images
docker images

e.	Now let’s test the docker image by running below command and it should produce the same result as above but this time it is running as Container..
docker run testimg

WELL DONE!!, WE MANAGED TO CREATE OUR FIRST CONTAINER

Now what … ah we can run the same job in kubernetes ecosystem where can just push the docker image as batch job and it will run by kubernetes scheduler.

Let’s create different folder to keep things cleaner .. kube-job1 (Name whatever you like ) and create a file call testjob.yaml
File content will look like this

apiVersion: v1
kind: Pod
metadata:
 name: local-img-test
spec:
  containers:
  - name: local-img-container
    image: testimg
    imagePullPolicy: Never

	So here we have specified our kind of job as pod.. give some name to the pod.. and it will be using container local-img-container, where it will be running the testimg  which we had created via docker. Let’s create a pod using kubectl

kubectl create -f .\testjob.yaml

	we can run following command to see what 	all pods are there ..
	kubectl get pods

will show the list of pods. 

Mount Windows folder in docker & kubernetes
Okay nice start and now let’s make it more complicated .. 
Let’s dump the output of pandas to csv but hold on if I output the file and that stays on docker linux OS then I can’t read that result, how wonderful it will be if I can have it in my windows directory.
Let’s first deal with docker..


import pandas as pd

# initialize list of lists 
data = [['A', 10], ['B', 15], ['C', 14]] 
  
# Create the pandas DataFrame 
df = pd.DataFrame(data, columns = ['Name', 'Age']) 
  
# dump the output to csv
df.to_csv(r'/mountdrive/ex2.csv')   ## On my windows machine I don’t have /mountdrive


Docker file looks like below ..

FROM python
RUN mkdir -p /home/ex2
COPY . /home/ex2
RUN ls /home/ex2
RUN pip install -r /home/ex2/requirements.txt
CMD python /home/ex2/ex2.py


Goto the folder where example 2 is created and let’s create docker image out  of it ..
docker build -t ex2img .


now let’s try to run the code without mounting the drive … we will see following exception and obviously it is expecting /mountdrive

Traceback (most recent call last):
  File "/home/ex2/ex2.py", line 10, in <module>
    df.to_csv(r'/mounteddrive/ex2.csv')
  File "/usr/local/lib/python3.7/site-packages/pandas/core/generic.py", line 3020, in to_csv
    formatter.save()
  File "/usr/local/lib/python3.7/site-packages/pandas/io/formats/csvs.py", line 157, in save
    compression=self.compression)
  File "/usr/local/lib/python3.7/site-packages/pandas/io/common.py", line 424, in _get_handle
    f = open(path_or_buf, mode, encoding=encoding, newline="")
FileNotFoundError: [Errno 2] No such file or directory: '/mounteddrive/ex2.csv'


Create a output folder in C:\temp or one can choose any folder…
docker run -v C:\temp\output:/mounteddrive  ex2img

This will create ex2.csv in the folder mentioned below… 

Yassssssssss this is what I wanted to achieve,

Let’s try to deploy this as pod 


If there is any error while creating kubernetes pod, to see details .. use following command

kubectl describe pods ex2-pod-img

apiVersion: v1
kind: Pod
metadata:
  name: ex2-pod-img
spec:
  containers:
  - name: ex2-container
    image: ex2img
    imagePullPolicy: Never
    volumeMounts:
    - mountPath: /mounteddrive
      name: volume1
  restartPolicy: Never 
  volumes:
  - name: volume1
    hostPath:
      # directory location on host
      path: /C/temp/output   

Once we create pod, this will create output in C:\temp\Output


Cool, now we have seen how to present local file system to docker image as well as to kubernetes container, let’s deal with Arguments problem

Passing arguments to docker image & kubernetes container

In our process of mounting the local drive,we made our python job not testable in windows folder…

If only thing which is making non portable is the directory path, let’s pass this as an argument but … how do I do that in a docker image and kubernetes…

Let’s change first our python program to take output directory for csv as an argument..


import pandas as pd
from argparse import ArgumentParser

parser = ArgumentParser()
parser.add_argument("-o", "--outputDir", dest="dirpath",
				help="Output Directory Path", metavar="File")

args = parser.parse_args()

# initialize list of lists 
data = [['A', 10], ['B', 15], ['C', 14]] 
  
# Create the pandas DataFrame 
df = pd.DataFrame(data, columns = ['Name', 'Age']) 
  
# dump the output to csv
df.to_csv(rf'{args.dirpath}/ex3.csv')

run following command to test your python program….

python ex3.py -o c:\temp\output

This should create ex2.csv in C:\temp\output


Let’s make a docker image out of it …

Docker file now looks like below …, notice that CMD has been replaced with ENTRYPOINT, so we can pas arguments
FROM python
RUN mkdir -p /home/ex3
COPY . /home/ex3
RUN ls /home/ex3
RUN pip install -r /home/ex3/requirements.txt
ENTRYPOINT ["python","/home/ex3/ex3.py"]


Now we run the docker image by mounting the drive and then passing that drive path as output directory
docker run -v C:\temp\output:/mounteddrive  ex3img -o /mounteddrive

Now run this as a pod 

apiVersion: v1
kind: Pod
metadata:
  name: ex3-pod-img
spec:
  containers:
  - name: ex3-container
    image: ex3img
    imagePullPolicy: Never
    args: ["-o/mounteddrive"]
    volumeMounts:
    - mountPath: /mounteddrive
      name: volume1
  restartPolicy: Never 
  volumes:
  - name: volume1
    hostPath:
      # directory location on host
      path: /C/temp/output   


This is just a tip of ice-berg to let anyone going but there is lots which can be done with docker and kubernetes which is beyond the scope of this article.

