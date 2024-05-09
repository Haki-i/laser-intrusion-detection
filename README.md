# laser intrusion detection (mail + caméra)

# **Objectif du projet**

L'objectif est de mettre en place un système de laser pour détecter lorsqu’un individu franchit une zone. Le faisceau atteint une photorésistance qui détecte les variations d’intensité lumineuse. Si le chemin du laser est coupé, une notification par mail est envoyée par une NodeMCU, avec le lien de l’adresse IP du flux vidéo de la caméra de surveillance.

# I- **Conception matérielle**

Dans un premier temps, nous cherchons à délimiter une zone dans laquelle il sera interdit de circuler pour un individu. Pour cela, nous utiliserons un laser ainsi que deux miroirs réfléchissants.

Ces miroirs ont été fabriqués en recouvrant un papier cartonné d’un film sans tain.

Nous positionnons le laser dans un coin de la pièce, et l’un des miroirs à l'autre extrémité, aligné avec le laser. En modifiant l'orientation de ce miroir, nous pouvons rediriger le faisceau vers une autre partie de la pièce.

Nous répétons le même processus avec le deuxième miroir de manière à obtenir une zone formée par trois rayons laser. 

![Screenshot (857)](https://github.com/Haki-i/laser-intrusion-detection/assets/137703849/9844e719-36f0-4107-b59f-2e35363cf9fd)

![20240508_005917-ezgif com-optimize](https://github.com/Haki-i/laser-intrusion-detection/assets/137703849/b7e8126c-e3be-41f7-b367-63ddfd4f2681)

Le faisceau final arrive ainsi sur une photorésistance. Si un individu franchit l’un de ces rayons, une différence d'intensité lumineuse sera détectée, et une alerte par notification nous sera envoyée.

![WhatsAppVideo2024-05-09at12 09 41_62a274b1-ezgif com-crop](https://github.com/Haki-i/laser-intrusion-detection/assets/137703849/83653f84-4cce-4437-b320-6fb9c6d60f12)

Remarque : Étant donné que nous n'utilisons pas de véritables miroirs, mais un film sans tain présentant certaines imperfections, le faisceau du laser s'écarte progressivement et perd de sa concentration initiale.

# II- Conception électronique

## a) Le laser

Pour éventuellement contrôler l'activation du laser, nous utiliserons un module filaire que nous brancherons à une Arduino Nano. .

Nous connectons le Ground et un pin digital.

![WhatsApp Image 2024-05-09 at 12 29 14_a415d37e](https://github.com/Haki-i/laser-intrusion-detection/assets/137703849/9a7370bf-5038-4e4b-8476-56ba9f7bda4a)

## b) La photorésistance

La photorésistance nous permettra de savoir si le chemin du laser est coupé en détectant le changement d'intensité lumineuse. 

Nous la connectons à une NodeMCU au Ground, au 3.3 V, et à la broche analogique A0.
![WhatsApp Image 2024-05-09 at 12 29 14_9e037e2e](https://github.com/Haki-i/laser-intrusion-detection/assets/137703849/32425252-351f-410c-90a5-b56c1e8e3a10)

## c) La caméra

Pour la caméra de surveillance, nous utiliserons l'ESP32-CAM. Celle-ci partage le flux vidéo par Wi-Fi sur une adresse IP locale que nous enverrons par mail dans la suite du projet.

# III- Conception informatique

## a) Activation du laser

L'activation du laser est très simple. Il suffit de déclarer la broche du module connecté à la carte comme sortie et d'utiliser la fonction `digitalWrite(pin, HIGH)`.

Remarque : Pour une future amélioration du projet, le rôle du laser pourrait être développé, par exemple en le contrôlant.

## b) Détection faisceau laser

La photorésistance est connectée à la broche analogique A0. Nous la déclarons comme sortie, et lisons ses données, comprises entre 0 et 1023, à l'aide de la fonction `analogRead(pin)`.

Sa résistance varie en fonction de la quantité de lumière qu'elle reçoit : plus la lumière est intense, plus sa résistance est faible, et donc, plus le courant circule.

Ainsi, il est possible de détecter la variation d'intensité lumineuse lorsque le faisceau laser est en contact ou non avec la photorésistance. Nous déterminons empiriquement un seuil limite (par exemple, 100). Si les valeurs reçues dépassent ce seuil, cela signifie que le faisceau laser a été interrompu.

## c) Envoi notification par mail

### Mise en place du SMTP

Pour envoyer un mail depuis notre NodeMCU, nous utiliserons un serveur SMTP (Simple Mail Transfer Protocol). Ce service est utilisé pour envoyer des mails d'un expéditeur à un destinataire, en agissant comme une passerelle entre la NodeMCU et la boîte de réception du destinataire.

Pour envoyer un mail, nous devons connaître les informations du serveur SMTP du fournisseur de service de messagerie, dans notre cas Gmail, ainsi que les informations de connexion.

- **Serveur SMTP :** **`smtp.gmail.com`**
- **Port SSL :** 465
- **Adresse mail de l’expéditeur**
- **Mot de passe :** mot de passe des applications
- **Adresse mail du destinataire**

À savoir que Gmail bloque par défaut l'accès aux applications qui n'utilisent pas des méthodes de connexion modernes. Nous ne pouvons donc pas utiliser notre mot de passe habituel, mais un mot de passe des applications.

Pour cela, nous devons nous rendre sur notre compte Google et, dans l’onglet **Sécurité**, activer la vérification en deux étapes.

Ensuite, il suffit de chercher la section **Mots de passe des applications** et de générer un nouveau mot de passe. C'est celui-ci que nous utiliserons dans notre code Arduino.

### **Envoi du mail**

Pour envoyer des e-mails avec une NodeMCU, nous utiliserons la bibliothèque **`ESP-Mail-Client`**.

Comme mentionné précédemment, nous devons déclarer les informations du serveur SMTP, les identifiants de l'expéditeur et l'adresse mail du destinataire. Nous devons également fournir le SSID et le mot de passe Wi-Fi pour se connecter à Internet.

Chaque fois que le seuil défini est dépassé, nous configurons les en-têtes du mail, notamment le nom de l'expéditeur et l'objet, ajoutons le message, puis l'envoyons.

Dans notre cas, le message contient un texte brut indiquant l'adresse IP vers laquelle la caméra transmet le flux vidéo.

```arduino
message.sender.name = F("Intru détecté !");
message.sender.email = AUTHOR_EMAIL;
message.subject = F("Caméra de surveillance activée");
message.addRecipient(F("ESP"), RECIPIENT_EMAIL);

String textMsg = "http://192.168.1.30/";
message.text.content = textMsg.c_str();
message.text.charSet = "us-ascii";
message.text.transfer_encoding = Content_Transfer_Encoding::enc_7bit;

if (!smtp.connect(&config)) {
      ESP_MAIL_PRINTF("Connection error, Status Code: %d, Error Code: %d, Reason: %s", smtp.statusCode(), smtp.errorCode(), smtp.errorReason().c_str());
      return;
```
![20240509_212312-ezgif com-optimize](https://github.com/Haki-i/laser-intrusion-detection/assets/137703849/bb46b10d-6154-44f7-a460-d3a380e1e1e8)
