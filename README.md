BotHound
=======

Automatic DDOS attack detector and botnet classifier
-----------

# Descritpion
Bothound is an automatic attack detector and botnet classifier. Its purpose is to create a historical classification of the attacks with detailed information regarding the attackers (country-based, time-based, etc.).

Bothound's role is to detect and classify the attacks (incidents), using the anomaly-detection and machine-learning tool Grey Memory. BotHound attack classifier is reacting to anomalous detector and start gathering live information from Deflect network. It computes a behaviour vector for all visitors of the network when Greymemory detect an anomaly. BotHound group the client IPs in different groups (cluster) using unsupervised ML algorithms in order to profile the group of malicious visitors. It uses different measures to tag the groups which are more likely to be attackers. After that it feed all the behavoir vectors of bot ips into a classifier to detect if the botnet has a history of attacking deflect network in the past. It generate a report based on the conclusion to Sysops and gets feedback to improve its classification performance.

# Installation

## Python
Python 2.7 should be installed

## Libraries
The following libraries should be installed:  

```  
[sudo] apt-get install emacs python java javajdk libmysqlclient-dev build-essential python-dev python-numpy python-setuptools python-scipy libatlas-dev python-matplotlib python-mysqldb python-geoip libffi-dev python-dnspython libssl-dev python-zmq   
[sudo] apt-get install easy_install pip  
[sudo] pip install -U scikit-learn  
[sudo] apt-get install git  
 ```  
 

## Jupyter
* first make sure that you install jupyter locally because nbextension has a bug and only is able to install if there is a local installation.  
``` 
sudo pip install jupyter --user
```

* install jupyter system wide  
```
sudo pip install jupyter
```

* install jupyter nbextensions  
```
pip install https://github.com/ipython-contrib/IPython-notebook-extensions/archive/master.zip
```

* by mistake it copies the file in the local folder. Copy the files to the system wide folder.  
```
sudo cp /root/.local/share/jupyter /usr/local/share  
sudo chmod -R a+r /usr/local/share/jupyter 
```

## Get Source Code 
```
git clone https://github.com/equalitie/bothound  
cd bothound/
```

## Install Packages
Install required packages from requirements.txt:  

```
pip install -r requirements.txt  
```

## Configuration 
You need to create a configuration file bothound.yaml

1. Make a copy of conf/rename_me_to_bothound.yaml  
2. Rename the copy to bothound.yaml  
3. Update the file with your credentials 
	
# Initialization

## Creating database
To create a database, you need to any script which instantiate bothound\_tools object, for example:  
```
cd src  
python session_computer.py  
```

Make sure the database and the tables are created successfully.

## Running Jupyter
1. Make sure Jupyter instance is running on the bothound server. 
To run the instance, use:  
```
jupyter notebook --no-browser --port=8889
```
2. Establish a tunnel to Jupyter instance from your local computer:  
```
ssh -N -L 8889:127.0.0.1:8889 anton@bothound.deflect.ca
```
3. Open the local URL [http://localhost:8889/](http://localhost:8889/)
Make sure you see a list of files and folders.

# Definitions
* Session - a IP and a vector of features values recorded and calculated during a period of the IP activity  
* Feature - an individual measurable property of a session   
* Incident - a set of sessions recorded during a time interval  
* Attack - a subset of sessions in an incident which was labeled as an attack  
* Botnet - a list of IPs participated in similar attacks   

# Incidents 
Incidents are created manually using Adminer interface. In the future incidents will be created automaically based on messages from GreyMemory anomaly detector.

## Creating incidents 
* Insert a new record into into "incidents" table. 
* Make sure you filled at least "start", "stop", "target" fields.
* The target URL should not contain "www." at the beginning. If you have multiply targets, you can add them sepatated by comma.
* Set "process" field to 1.

## Creating incidents from nginx logs
* Insert a new record into into "incidents" table. 
* Make sure you filled "file_name" with the full path to a nginx log file.
* Set "process" field to 1.

# Sessions
## Session Computer
Session Computer calculates sessions for all the records in incidents table containing 1 in the "Process" field.

* Run session computer with 
```
python session_computer.py
```   
* Session computer will recalculate all the incidents records containing 1 in the field "process"
* For regular incidents Session Computer runs elastic search queries. For nginx incidents Session Computer will parse the corresponding log file.
* The sessions will be stored in "sessions" table

## IP Encryption
For security reasons Bothound stores only encrypted IPs in the session table in "ip\_encrypted", "ip\_iv","ip\_tag" fields. 
The hash of the IP is also stored in the field "ip".
The encryption key is set in the configuration file "conf/bothound.yaml" ("encryption\_passphrase").
Bothound suports multiply encryption keys. Encryption table contains the hash value of the key which was used to encrypt IPs of an incident. 

In order to get the decrypted IPs of the incident use extract_attack_ips() function in bothound_tools.py 

# Attacks
Bothound uses clustering methods in order to separate attackers from regular traffic.
This process of labeling a subset of incident sessions as an attack is manual. 
The user opens a Jyputer notebook, chooses an incident, clusters the sessions with different clustering algorithms and manually assigns an arbitrary attack number to the selected clusters. 

## Jupyter notebooks
The [Jupyter Notebook](http://jupyter.org/) is a web application that allows you to create and share documents that contain live code, equations, visualizations and explanatory text. 
Notebook contains a list of cells(markdown, python code, graphs). 
Use Shift+Enter to execute a cell.
You can fold/unfold the contect of a cell using an "arrow" character on the left.

## Loading incident
* Open Jupyter interface URL: [http://localhost:8889/](http://localhost:8889/)
* Open src/Clustering.ipynb  
* Execute Initialization chapter  
* Configuration chapter: change the assignment of variable "id\_incident = ..." to your incident number  
* Configuration chapter: uncomment the features you want to use: "features = [...]"  
* Execute Configuration chapter  
* Execute "Load Data"chapter 

## Clustering
* Execute DBSCAN Clustering chapter. 
After the clustering is done, you will see a bar plot of clusters. 
Y-axes coresponds to the size the cluster. Every cluster has it's own color from a predefined palette.

* Use plot3() function in the second cell of the chapter to create different 3D scatter plots of the calculated clusters:  
```python
plot3([0,1,3], X, clusters, [])  
```
The first argument of this function is an array of indexes of the 3 features to display at the scatter plot. Note, that these are the indexes in the array of uncommented features from "Configuration" chapter. If you have more than 3 uncommented features, choose different indexes and re-execute plot3() cell.

* Choose your features carefully. 
It's always better to experiment and play with different features subsets (uncommented in "Configuration" chapter). Clustering is very sensitive to feature selection. 
Different attacks might have different distinguishable features. 
If you change your features selection in "Configuration" chapter you must re-execute "Congiguration", "Load Data", "Clustering" chapters. 

* Double clustering.
In some cases DBSCAN clustering is not good enough. The suspected cluster might have a weird shape and even contain two different botnets. In order to devide such a cluster futher you can use the second iteration, which we call "Double Clustering". You should choose the target cluster after first clustering, and the number of clusters for K-Means clustering algorithm.  
The second cell in this chapter is the same plot3() function which display a 3D scatter plot of double clustering.  
```python
plot3([0,1,3], X2, clusters2, [])
```
Note, that you should use X2 and clusters2 arguments.

## Attack saving
* Choose your attack ID(s)
Attack ID is an arbitrary number you assign to the botnet. The attack is identified by incident ID and attack ID.
It is possible to have more than one attack in a single incident. 

* Modify the tools.label\_attack() function arguments  
If you have more than 1 attack number to save, you should have add a call to label/attack() function for every attack.  
For example, for attack #1 you choose cluster #3:  
```python 
tools.label\_attack(id\_incident, attack\_number = 1, selected\_clusters = [3], selected\_clusters2 = [])  
```
If you used double clustering, don't forget to specify the indexes for selected_clusters2.
For example, for attack #1 you choose cluster #3 and double clusters #4 and #5:   
tools.label\_attack(id\_incident, attack\_number = 1, selected\_clusters = [3], selected\_clusters2 = [4,5])  

* Execute "Save Attack" chapter. 

## Feature exploration
In this section user can explore the distribution of a single feature over the clusters to verify the quality of the clustering results.   
box\_plot\_feature(clusters, num_clusters = 4, X = X, feature\_index = 2)  
The function will display a boxplot of feature values distribution per cluster.
Using this graph you can get more understanding about the quality of the clustering you used.  
For instance, if you know in advance that the attack you are clustering should have a significant higher hit rate, then you can expect that a proper attack cluster should have a similar high boxplot of "request\_interval" feature.

## Common IPs with other incidents
If two attacks share a significant portion of identical IPs, they are likely to belong to the same botnet.  
plot\_intersection(clusters, num\_clusters, id\_incident, ips, id\_incident2 = ..., attack2 = -1)  
This function will create a bar plot highliting portions of the clusters which share identical IPs with another incident(specified by variable id_incident2). It's also possible to specify a particular attack index.

## Countries
This graph explores the country distribution over the clusters. 

## Banjax
Even if an IP was banned during the incident, Bothound does not use this information for clustering.
Nevertherless, the distribution of banned IPs over the clusters might be usufull.
This graph will display portions of IPs, banned by Banjax per cluster.



