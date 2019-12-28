# KUBEFLOW_0.7__MANUAL


What Is KUBEFLOW?

Kubeflow Is an Cutting-Edge Platform From Google That Lets You Deploy Distributed ML Pipeline Right From Training to Inferencing .
And the Good NEWS ID THAT…YOU Can DO IT ALL ON TOP OF YOUR FAVOURITE KUBERNETES CLUSTER both Open Source and Vendor Maintained.
With Kubeflow 0.7 it is even better you can install KUBEFLOW  without needing to get your hands dirty on KSonnet.
And The GREATEST NEWS IS THAT ..WE CAN GET OUR PYTORCH TRAINING PIPELINE UP AND RUNNING..Using the kubeflow/pytorch-operator And THAT too DISTRIBUTED !!!!!!

THIS IS AWESOME.


LETS BE A DOER THAN A TALKER..

LET SPIN UP OUR KUBEFLOW CLUSTER ON KUBERNETES :

Basic Requirement:

1.Docker 19+
2.Kubernetes 1.15
3.NVIDIA Docker Container Toolkit
4.NVIDIA Kubernetes Daemon.



#INSTALL KUBEFLOW FRAMEWORK:

mkdir ~/KUBEFLOW
cd  ~/KUBEFLOW
mkdir runtime
mkdir exec
chmod -R 777 *

wget https://github.com/kubeflow/kubeflow/releases/download/v0.7.0/kfctl_v0.7.0_linux.tar.gz

tar -xzvf  kfctl_v0.7.0_linux.tar.gz
cd kfctl_v0.7.0_linux

#Edit .bashrc for Setting New Environment Variables

sudo nano ~/.bashrc

#Append the Following Section in basic and Save it:

# The following command is optional. It adds the kfctl binary to your path.
# If you don't add kfctl to your path, you must use the full path
# each time you run kfctl.
# Use only alphanumeric characters or - in the directory name.
export PATH=$PATH:"<path-to-kfctl>"

# Set KF_NAME to the name of your Kubeflow deployment. You also use this
# value as directory name when creating your configuration directory.
# For example, your deployment name can be 'my-kubeflow' or 'kf-test'.
export KF_NAME=<your choice of name for the Kubeflow deployment>

# Set the path to the base directory where you want to store one or more 
# Kubeflow deployments. For example, /opt/.
# Then set the Kubeflow application directory for this deployment.
export BASE_DIR=<path to a base directory>
export KF_DIR=${BASE_DIR}/${KF_NAME}

# Set the configuration file to use when deploying Kubeflow.
# The following configuration installs Istio by default. Comment out 
# the Istio components in the config file to skip Istio installation. 
# See https://github.com/kubeflow/kubeflow/pull/3663
export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v0.7-branch/kfdef/kfctl_k8s_istio.0.7.1.yaml"

#Set Environment Changes:

source ~/.bashrc

#Create the Dynamic Volumes for the DB and Metadata Services[VERY ESSENTIAL FOR CREATING PIPELINE OR REUSING EXISTING PIPELINES] :

kubectl delete -f katib_mysql_pv.yaml -n kubeflow
kubectl create -f katib_mysql_pv.yaml -n kubeflow

kubectl delete -f metadata-mysql_pv.yaml -n kubeflow
kubectl create -f metadata-mysql_pv.yaml -n kubeflow

kubectl delete -f minio-pv-claim.yaml -n kubeflow
kubectl create -f minio-pv-claim.yaml -n kubeflow

kubectl delete -f mysql-pv-claim.yaml -n kubeflow
kubectl create -f mysql-pv-claim.yaml -n kubeflow

#Check if all the Persistent Volumes Claims are bound to avoid Out Of Bounds Volume Claim Exception:  

kubectl get pvc -n kubeflow

NAME             STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
katib-mysql      Bound    metadata-mysql   10Gi       RWO                           30m
metadata-mysql   Bound    katib-mysql      10Gi       RWO                           30m
minio-pv-claim   Bound    minio-pv-claim   20Gi       RWO                           30m
mysql-pv-claim   Bound    mysql-pv-claim   20Gi       RWO                           30m



#SPIN UP THE KUBEFLOW CLUSTER:
cd ..
rm -rf ${KF_DIR}
mkdir -p ${KF_DIR}
cd ${KF_DIR}
kfctl apply -V -f ${CONFIG_URI}


#TO ACCESS THE KUBEFLOW URI WE HAVE FORWARD THE PORT 80 of istio-ingressgateway .
#This will give a single point of UI for accessing Jupiter Notebooks,buil Pipeline and Other Kubeflow Features.

kubectl get svc istio-ingressgateway -n istio-system

kubectl port-forward -n istio-system svc/istio-ingressgateway 15017:80


#Spin Down the Kubeflow Deployment:

cd ${KF_DIR}
# If you want to delete all the resources, run:
../kfctl delete -f ${CONFIG_FILE}


INSTALL THE KUBEFLOW PIPELINE SDK:

#Activate Python Virtual Environment:


#INSTALL KUBEFLOW PIEPLINE PYTHON SDK INSIDE PYTHON 3.6 > VIRTUAL ENV

pip install https://storage.googleapis.com/ml-pipeline/release/latest/kfp.tar.gz --upgrade

#Test if KubeFlow Pipeline is running properly:
which dal-compile

OutPut:
/home/system/PyEnv1/python36/bin/dsl-compile

#Create a KUBEFLOW PIPELINE from a Python3 File :


cd  ~/KUBEFLOW/pipelines/samples/core/sequential

dsl-compile --py sequential.py --output sequential.tar.gz

#Open KUBEFLOW UI
#Go To PIPELINE and UPLOAD THE PIPELINE:
#EXECUTE IT [REF: https://www.kubeflow.org/docs/pipelines/pipelines-quickstart/]


#EXECUTE PYTORCH OPERATOR AN RUN PYTORCH FILE:
[REF: https://github.com/kubeflow/pytorch-operator]

#CLONE THE KUBEFLOW PYTORCH OPERATOR REPO FROM GITHUB:

git clone https://github.com/kubeflow/pytorch-operator

#Go To the Directory and Build the Docker File

cd /KUBEFLOW/pytorch-operator/examples/mnist

docker build -f Dockerfile -t kaustav4284/pytorch-dist-mnist-test:1.0 ./

docker push kaustav4284/pytorch-dist-mnist-test:1.0

#EXECUTE THE PYTORCH OPERATOR JOB ON KUBERNETES CLUSTER:

kubectl create -f ./v1beta1/pytorch_job_mnist_gloo.yaml

#Monitor the Pytorch-Operator Job PoDs Status

kubectl get po -n kubeflow -default
 
kubectl get pods -l pytorch-job-name=pytorch-dist-mnist-gloo

#Monitor the Pytorch-Operator Job and See Status Details

#LOG INSPECTION:

PODNAME=$(kubectl get pods -l pytorch-job-name=pytorch-dist-mnist-gloo,pytorch-replica-type=master -o name)
kubectl logs -f ${PODNAME}

#JOB DETAIL INSPECTION:

kubectl get -o yaml pytorchjobs pytorch-dist-mnist-gloo



Pipelines Issue Due to Storage Class:

https://github.com/kubeflow/kubeflow/issues/4145

1. Create Folders /data/minio ,/data/katib ,/data/mysql ,/data/metada

2. manually create PVs for each PVC
Create PV for: katib-mysql, metadata-mysql, minio-pv-claim, mysql-pv-claim Follow the PV format like this, and create a pv-katib-mysql.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: katib-mysql
  labels:
    type: local
    app: katib
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/katib

Run the command kubectl apply -f pv-katib-mysql.yaml to create PV for katib-mysql. Check PVC status with kubectl -n kubeflow get pvc again and you will see katib-mysql STATUS become Bound. Repeat the steps for metadata-mysql(10Gi), minio-pv-claim(20Gi), mysql-pv-claim(20Gi) with different hostPath. When all PVC Pending issues solved, everything is Good on Pipeline dashboard.

cd ~/HELM/KUBEFLOW_VOLUME_MANIFESTS
kubectl -n kubeflow get pvc


kubectl get storageclass
kubectl patch storageclass slow -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: slow
  resources:
    requests:
      storage: 30Gi


