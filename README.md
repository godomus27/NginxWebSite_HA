# NginxWebSite_HA
Prenons le cas on veuille mettre en ligne un site en haute disponibilité avec Docker SWARM ET NGINX: ou NGINX assure la fonction du serveur web et Docker la Haute disponibilité.

Après avoir installer notre système d’exploitation habituel les machines (ville-webservices01 à 05 ) du cluster il faut installer docker engin sur elles.
Pour aller plus vite dans le test du code ou dans le développement d’un project DOCKER , afin de palier au limitation de temps et de ressource  ou dans le cas d’un environnement de preuve de concepts je propose d’utiliser une plateforme en ligne CLOUD 100%  WEB disponible sous le lien http://labs.play-with-docker.com
Autrement il faut installer ses serveurs construire con cluster avec.

Faire un SSH sur le SERVEUR MANAGER du Docker Cluster et executer la commande suivante : 
** Créons le répertoire du code 
ville-webservices01:~ user$ mkdir webservices

Supposant que notre code web se trouve au lien suivant : https://github.com/BlackrockDigital/startbootstrap-stylish-portfolio/archive/gh-pages.zip
ville-webservices01:~ user$ cd webservices
ville-webservices01:~ user$ wget https://github.com/BlackrockDigital/startbootstrap-stylish-portfolio/archive/gh-pages.zip
ville-webservices01:~ user$ unzip gh-pages.zip
ville-webservices01:~ user$ mv startbootstrap-stylish-portfolio-gh-pages web

**Création du fichier Dockerfile
ville-webservices01:~ user$ vi  Dockerfile

FROM nginx
MAINTAINER MAXIME GODONOU-DOSSOU
ADD  web /usr/share/nginx/html

**Création de l’image Web avec docker 
ville-webservices01:~ user$ docker build -t godomus27/web01 .

**creation du DOCKER SWARM  cluster 
ville-webservices01:~ user$   docker swarm init --advertise-addr eth0

# Se Connecter Ajout des workers 
ville-webservices02:~ user$     docker swarm join --token SWMTKN-1-47nk9uq15vsu3lpn2dwqieyp04bgxaix6bnnms3u2ueizx1aiq-944kxi4xpwb7awvlwgmo5xv45 10.0.110.4:2377
ville-webservices03:~ user$     docker swarm join --token SWMTKN-1-47nk9uq15vsu3lpn2dwqieyp04bgxaix6bnnms3u2ueizx1aiq-944kxi4xpwb7awvlwgmo5xv45 10.0.110.4:2377
ville-webservices04:~ user$     docker swarm join --token SWMTKN-1-47nk9uq15vsu3lpn2dwqieyp04bgxaix6bnnms3u2ueizx1aiq-944kxi4xpwb7awvlwgmo5xv45 10.0.110.4:2377
ville-webservices05:~ user$     docker swarm join --token SWMTKN-1-47nk9uq15vsu3lpn2dwqieyp04bgxaix6bnnms3u2ueizx1aiq-944kxi4xpwb7awvlwgmo5xv45 10.0.110.4:2377

# Creation du reseau OVERLAY pour une reseau lineaire 
$ docker network ls

## Création du réseau de type OVERLAY qui crée une autre couche d'abstraction sur réseau auquel sont connecter les serveurs du docker swarm cluster
$ docker network create ville-net -d overlay
52pbz7nzieijnzapu4a3o79w3

## Pour voir le nouveau réseau créé 
$ docker network ls

#Mise en place sur le  CLuster des outils  Monitoring & visualisation
ville-webservices01:~user$
docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
  
**Google Monitor
ville-webservices01:~user$ docker service create \
								—network ville-net \
 								--mode global \
								--mount type=bind,source=/,destination=/rootfs,ro=1 \
 			 					--mount type=bind,source=/var/run,destination=/var/run \
  								--mount type=bind,source=/sys,destination=/sys,ro=1 \
 								--mount type=bind,source=/var/lib/docker/,destination=/var/lib/docker,ro=1 \
 								--publish mode=host,target=8080,published=8080 \
  								--name=cadvisor  google/cadvisor:latest

**Create the docker registry
ville-webservices01:~user$  docker service create --name registry  --network ville-net --publish 5000:5000 registry:2

**Changer le nom de mon image pour le deposer sur le registry
ville-webservices01:~user$  docker tag godomus27/web01  localhost:5000/web01

**Pouser l’image vers le repot
ville-webservices01:~user$  docker push localhost:5000/web01

**Creating a service “sitewebville”
ville-webservices01:~user$  docker service create --replicas 12 --network ville-net --name sitewebville --publish 80:80/tcp localhost:5000/web01

**Pour voir le status du service avec le nombre d’instances et le status de chaque instance 
ville-webservices01:~user$ docker service ps sitewebville

**Pour voir le status augmenter le nombre d’instances de 12 à 100  pour le service “ sitewebville"
ville-webservices01:~user$ docker service scale sitewebville=100
 
Pour voir le status augmenter le nombre d’instances de 100 à 5  pour le service “ sitewebville"
ville-webservices01:~user$ docker service scale sitewebville=5

On peut tester la page dans un navigateur avec le http://IP_SWARM_MASTER 

On peut voir status en LIVE des chaque instance du service sitewebville dans un navigateur avec le http://IP_SWARM_MASTER:5010

On peut voir la consommation de resource sur le cluster de docker swarm dans un navigateur avec le http://IP_SWARM_MASTER:8080

Pour mettre à jour le site à jour dans interruption de service 
On telecharge la dernière version du code avec GIT CLONE;  ou WGET; SVN CO; 
On se déplace dans le répertoire ou on a notre docker file
Creation de la nouvelle version de l’image et son déplacement dans dans notre dépôt d'images
ville-webservices01:~user$   docker build -t godomus27/web02 .
ville-webservices01:~user$  docker tag godomus27/web02 localhost:5000/web02
ville-webservices01:~user$  docker push  localhost:5000/web02

Mis à jour du site sans interruption de service
ville-webservices01:~user$  docker service update --replicas 12  --update-delay 10s --update-parallelism 2 --name sitewebville --image localhost:5000/web02

On peut tester la page dans un navigateur avec le http://IP_SWARM_MASTER 
On peut voir status en LIVE des chaque instance du service sitewebville dans un navigateur avec le http://IP_SWARM_MASTER:5010
On peut voir la consommation de resource sur le cluster de docker swarm dans un navigateur avec le http://IP_SWARM_MASTER:8080
