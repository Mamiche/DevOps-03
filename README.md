# Déployez votre code dans un conteneur Docker via Jenkins sur AWS

![AWS](https://imgur.com/Hk28ffE.png)

**Dans ce blog, nous allons déployer une application web Java dans un conteneur Docker hébergé sur une instance EC2 grâce à Jenkins.**

### Plan

* Installer Jenkins
* Configurer Maven et Git
* Intégrer GitHub et Maven avec Jenkins
* Préparer un hôte Docker
* Intégrer Docker avec Jenkins
* Automatiser le processus de build et de déploiement avec Jenkins
* Tester le déploiement

### Prérequis

* Compte AWS
* Compte Git/GitHub avec le code source
* Machine locale avec un accès CLI
* Connaissances de base en Docker et Git

## Étape 1 : Installation du serveur Jenkins sur une instance EC2 AWS

- Créez une instance EC2 Linux (t2.micro)
- Ajoutez les règles de sécurité : SSH (22) et 8080 (Jenkins)
- Connectez-vous via SSH
- Installez Java, EPEL, puis Jenkins

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install epel-release
sudo yum install java-11-openjdk
sudo yum install jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

- Accédez à Jenkins via `http://<IP-PUBLIQUE>:8080`
- Déverrouillez Jenkins avec le mot de passe situé dans `/var/lib/jenkins/secrets/initialAdminPassword`
- Installez les plugins suggérés
- Créez un compte admin

## Étape 2 : Intégrer GitHub avec Jenkins

```bash
sudo yum install git
git --version
```

- Depuis Jenkins :
  - Manage Jenkins > Manage Plugins > Install "GitHub Integration"
  - Manage Jenkins > Global Tool Configuration > Ajouter Git

## Étape 3 : Intégrer Maven avec Jenkins

```bash
cd /opt
sudo wget https://downloads.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
sudo tar -xvzf apache-maven-3.9.6-bin.tar.gz
```

Ajoutez à `~/.bash_profile` :

```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export M2_HOME=/opt/apache-maven-3.9.6
export PATH=$PATH:$M2_HOME/bin
```

```bash
source ~/.bash_profile
mvn -version
```

- Dans Jenkins :
  - Installer "Maven Integration Plugin"
  - Déclarer Maven et Java dans Global Tool Configuration

## Étape 4 : Préparer un hôte Docker

```bash
sudo yum install docker
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

```bash
docker pull tomcat
docker run -d --name tomcat-container -p 8081:8080 tomcat
```

Si erreur 404 :

```bash
docker exec -it tomcat-container bash
cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
```

Dockerfile personnalisé :

```dockerfile
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
```

```bash
docker build -t tomcatserver .
docker run -d --name tomcat-server -p 8085:8080 tomcatserver
```

## Étape 5 : Intégrer Docker avec Jenkins

```bash
sudo adduser dockeradmin
sudo passwd dockeradmin
sudo usermod -aG docker dockeradmin
```

Modifier `/etc/ssh/sshd_config` :

```bash
PasswordAuthentication yes
# puis :
sudo systemctl reload sshd
```

- Jenkins > Manage Jenkins > Manage Plugins > Installer "Publish Over SSH"

## Étape 6 : Créer un Job Jenkins pour déployer les artefacts sur le hôte Docker

![aws](https://miro.medium.com/v2/resize:fit:750/format:webp/1*ly3rtkBmll9nUQHdCJLv5A.png)

Cliquez sur "Install without restart" pour installer le plugin.

Ensuite, allez dans **Manage Jenkins > Configure System** pour configurer votre hôte Docker :

![aws](https://miro.medium.com/v2/resize:fit:750/format:webp/1*3INsphP5TB5_lGQ6mL-KQw.png)

Dans la section **Publish over SSH**, ajoutez un nouveau serveur SSH comme ci-dessous :

![aws](https://miro.medium.com/v2/resize:fit:750/format:webp/1*3hnLNy9fWJ036nHBTMjLbA.png)

> **Remarque** : l’utilisation de clés SSH est recommandée, mais ici, l’authentification se fait par mot de passe. Utilisez l’adresse IP privée si vos serveurs sont dans le même sous-réseau.

Cliquez sur **Apply** puis **Save**.

---

### Étape 7 : Mise à jour du Dockerfile pour copier les artefacts

Créez un dossier `/opt/docker` sur le hôte Docker :

```bash
sudo mkdir /opt/docker
sudo chown dockeradmin:dockeradmin /opt/docker
```

Déplacez le `Dockerfile` :

```bash
mv Dockerfile /opt/docker/
cd /opt/docker/
chown -R dockeradmin:dockeradmin /opt/docker/
```

Configurez Jenkins pour envoyer les fichiers à `/opt/docker`.

```dockerfile
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
COPY ./*.war /usr/local/tomcat/webapps
```

Construisez l’image :

```bash
docker build -t tomcat:v1 .
```

Et lancez un conteneur :

```bash
docker run -d --name tomcatv1 -p 8086:8080 tomcat:v1
```

Accédez à l’application : `http://<IP_PUBLIC>:8086/webapp/`

---

## Étape 8 : Automatiser le processus complet avec Jenkins

Dans Jenkins, dans votre job, sous **Exec command**, ajoutez :

```bash
cd /opt/docker;
docker build -t regapp:v1 .;
docker run -d --name registerapp -p 8087:8080 regapp:v1
```

![aws](https://miro.medium.com/v2/resize:fit:750/format:webp/1*D2aGmqZ8Uh1p4Fmz_nHsVg.png)

Effectuez une modification dans le dépôt GitHub pour déclencher le build automatiquement.

Une fois le build terminé, accédez à l’application sur `http://<IP_PUBLIC>:8087/webapp/`

---

## Conclusion

**Nous avons automatisé le déploiement d’une application Java à l’aide de Jenkins, Docker, GitHub et AWS EC2.**
