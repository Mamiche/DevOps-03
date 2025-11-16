# Déploiement d’une application Java dans un conteneur Docker via Jenkins sur AWS

![AWS](https://imgur.com/Hk28ffE.png)

Ce projet déploie automatiquement une application web Java dans un conteneur Docker sur une instance EC2 AWS, à l’aide de **Jenkins** et **Ansible**.

---

## Plan

* Provisionner l’infrastructure AWS avec Terraform
* Installer et configurer Jenkins, Docker et Tomcat via Ansible
* Déployer l’application Java dans un conteneur Docker
* Automatiser le processus de build et de déploiement avec Jenkins
* Accéder à l’application depuis le navigateur

---

## Prérequis

* Compte AWS avec permissions pour EC2, VPC, Subnet, Security Group
* Compte Git/GitHub avec le code source
* Machine locale avec Terraform et Ansible installés
* Clé SSH pour se connecter à l’instance EC2

---

## 1️⃣ Provisionnement de l’infrastructure (Terraform)

Le script Terraform crée :

* **VPC** avec subnet public et Internet Gateway
* **Security Group** autorisant SSH (22) et Jenkins/Tomcat (8080-9000)
* **Instance EC2** Amazon Linux 2 (`t2.micro`) avec clé SSH

```bash
terraform init
terraform plan -out plan.tf
terraform apply plan.tf
```

**Output :**

```
public_ip = <IP_PUBLIQUE>
```

---

## 2️⃣ Provisionnement de Jenkins, Docker et Tomcat (Ansible)

Le playbook Ansible installe et configure :

* Jenkins (avec service démarré)
* Docker
* Conteneur Tomcat personnalisé exposé sur le port 8081
* Utilisateur `dockeradmin` avec accès Docker et SSH

```bash
ansible-playbook -i inventory.ini jenkins.yml
```

**Inventory exemple :**

```ini
[jenkins]
<IP_PUBLIQUE> ansible_user=ec2-user ansible_ssh_private_key_file=devops_cle_rsa.pem
```

---

## 3️⃣ Accès aux services

* **Jenkins :** `http://<IP_PUBLIQUE>:8080`
  Mot de passe initial :

  ```bash
  sudo cat /var/lib/jenkins/secrets/initialAdminPassword
  ```

* **Tomcat :** `http://<IP_PUBLIQUE>:8081`

* **SSH dockeradmin :**

```bash
ssh dockeradmin@<IP_PUBLIQUE>
# Mot de passe : pass
```

---

## 4️⃣ Déploiement d’une application Java dans Docker via Jenkins

1. Créez un job Jenkins lié à votre dépôt GitHub.
2. Configurez **Publish over SSH** pour envoyer les artefacts vers `/opt/docker` sur l’instance EC2.
3. Placez le `Dockerfile` et les fichiers `.war` dans `/opt/docker` :

```dockerfile
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
COPY ./*.war /usr/local/tomcat/webapps
```

4. Dans Jenkins, ajoutez un **build step** pour construire et lancer le conteneur :

```bash
cd /opt/docker
docker build -t myapp:v1 .
docker run -d --name myapp -p 8087:8080 myapp:v1
```

5. Accédez à l’application :

```
http://<IP_PUBLIQUE>:8087/webapp/
```

---

## 5️⃣ Conclusion

Grâce à Terraform, Ansible et Jenkins, nous avons :

* Provisionné automatiquement l’infrastructure sur AWS
* Installé et configuré Jenkins, Docker et Tomcat
* Déployé une application Java dans un conteneur Docker
* Mis en place un processus automatisé de build et déploiement depuis GitHub
