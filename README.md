# ESP8266 simple RF to MQTT gateway
Passerelle RF > MQTT et répéteurs via wifi UDP réalisés à l'aide de cartes ESP8266 à intégrer à Home Assistant pour contrôler sa domotique avec les boutons/interrupteurs RF.

Ces interrupteurs ont pour avantages d'être très abordables et peu énergivores.
Avec cette passerelle à moindre coût, ils peuvent être positionnés là où vous le souhaitez pour piloter n'importe quel appareil connecté à votre instance Home Assistant via une automatisation (scène, lumière, TV, aspirateur robot...).

Attention, seuls les appuis courts sont supportés.

J'ai créé et mis en route ce système de passerelle il y a plus d'un an (août 2023) sans remarquer de problème de fonctionnement particulier.

Je vie dans un appartement avec étage dans une résidence récente, la passerelle est connectée sur l'USB de la box TV (tout le temps sous tension) au rez-de-chaussée et capte les codes RF depuis toutes les pièces sans avoir besoin de répéteur.

# 1. Synoptique

![Synoptique_base](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/Synoptique_base.PNG)


# 2. Matériel nécessaire

Il vous faudra une instance Home Assistant correctement configurée avec votre broker MQTT ainsi qu'une connexion WiFi 2,4 Ghz.

## 2.1 Passerelle et répéteur(s) :

- un ESP8266 ou tout autre type de carte de la sorte (Wemos D1 mini utilisé dans cet exemple) --> ~1,72€ ([Lien Aliexpress](https://fr.aliexpress.com/item/4000420770002.html?spm=a2g0o.order_list.order_list_main.211.13865e5bQncWCH&gatewayAdapt=glo2fra))

- un récepteur RF 433 type SYN480R --> ~0,45 € ([Lien Aliexpress](https://fr.aliexpress.com/item/1005006703068436.html?spm=a2g0o.productlist.main.5.1f7e429dGx7aNw&algo_pvid=289506d5-f996-461a-9a10-aa2d77ab60d4&algo_exp_id=289506d5-f996-461a-9a10-aa2d77ab60d4-2&pdp_npi=4%40dis%21EUR%210.82%210.80%21%21%210.84%210.82%21%402103246417323572127173771eb1ba%2112000042917820694%21sea%21FR%211923784476%21X&curPageLogUid=EaRn8tNHNF7O&utparam-url=scene%3Asearch%7Cquery_from%3A))


- du fil silicone pour les connexions et l'antenne du récepteur RF
- un boitier à imprimer en 3D (facultatif)


## 2.2 Bouton / Interrupteur RF :

La projet de passerelle RF a été testé avec succès sur ce type d'interrupteur 433 mhz (~2,60 €) : ([Lien Aliexpress](https://fr.aliexpress.com/item/1005005183893959.html?spm=a2g0o.order_detail.order_detail_item.7.25dc7d568X3G77&gatewayAdapt=glo2fra))

Attention, prévilégiez les modèles alimentés par une pile 27A 12V. J'ai pu remarquer que ceux alimentés en 3V avec une pile CR2032 ont une portée plus faible.

![Exemple interrupteur RF](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/Exemple_interrupteurRF.PNG)

# 3. Software (ESP_RFgateway & ESP_RFextender)

Il y a principalement deux contraintes à prendre en compte :
- les boutons/interrupteurs RF envoient un code unique par voie à une fréquence fixe (~10/sec). Ce qui fait que sur une pression courte classique de 200-250ms, le code peut être envoyé 3 ou 4 fois.
- en cas d'utilisation d'un ou plusieurs répéteurs, il est possible que les codes envoyés par un interrupteurs soient réceptionnés par la passerelle ET le(s) répéteur(s).

Afin d'éviter de devoir traiter les doublons dans Home Assistant, j'ai préféré ajouter des temporisations directement dans le code de la passerelle.
Le répéteur n'intègre pas de temporisation, il transmet à la passerelle via UDP tous les codes RF captés.

La temporisation est réglée à 1000ms : évite les doublons tout en permettant d'allumer puis éteindre une lumière rapidement. Vous pouvez la modifier dans le code.

![Synoptique_temporisation](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/Synoptique_temporisation.PNG)


## 3.1 Programmes

Vous aurez besoin d'installer l'IDE Arduino et les bibliothèques suivantes :
- RC Switch.h
- ESP8266Wifi.h
- PubSubClient.h
- ArduinoOTA.h (facultatif, permet de mettre à jour/modifier le code à distance en Wifi)
- WifiUDP.h (uniquement pour les répéteurs)

Je ne suis pas expert en codage Arduino, les exemple fournis sont fonctionnels mais méritent peut-être quelques optimisations, libre à vous de les réaliser ! ;)

N'oubliez-pas de renseigner les informations de connexion WiFi et MQTT avant de le flasher les programmes sur vos cartes.


### 3.1.1 ESP_RFgateway :

```cpp
//Code fonctionnel

#include <RCSwitch.h>
#include <ESP8266WiFi.h>
#include <PubSubClient.h>
#include <ArduinoOTA.h>
#include <WiFiUdp.h>

// Variables WiFi
const char* ssid = "XX"; // A modifier avec vos propres identifiants
const char* password = "XX"; // A modifier avec vos propres identifiants

// Variables MQTT
const char* mqtt_server = "192.168.1.39"; // A modifier avec vos propres identifiants
const char* mqtt_username = "XX"; // A modifier avec vos propres identifiants
const char* mqtt_password = "XX"; // A modifier avec vos propres identifiants
const char* mqtt_topic = "rfgateway/code";
unsigned long lastSendMQTT = 0; //Variable pour stocker en ms le dernier envoi d'un message MQTT
unsigned long delayMinSendMQTT = 1000; //Délai minimum en ms entre deux publications MQTT d'un même code reçu par RF


//Variables UDP
const int port = 1996;
char messageUdp[255];


//Variables diverses
const int RFrxPin = 4; //Pin du récepteur RF : D2
int previousCode = 0;


// Configuration RC Switch
RCSwitch mySwitch = RCSwitch();

// Déclaration du client MQTT
WiFiClient espClient;
PubSubClient client1(espClient);

// Configuration UDP
WiFiUDP udp;

//Fonction de connexion Wifi
void wifiConnect() {

  WiFi.mode(WIFI_STA);             
  Serial.print("Connexion à ");
  Serial.print(ssid); Serial.println(" ...");

  unsigned long millisWifiStart = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - millisWifiStart < 1*20*1000) { //Délai de 20 secondes pour se connecter au Wi-Fi, autrement l'ESP8266 redémarre
    delay(100);
    Serial.print(".");
  }
  if (WiFi.status() == WL_CONNECTED) {
  Serial.println('\n');
  Serial.println("Connection établie !");  
  Serial.print("Adresse IP :\t");
  Serial.println(WiFi.localIP());
  digitalWrite(BUILTIN_LED, HIGH);
  }
  else {
    Serial.println("Impossible de se connecter au réseau Wi-Fi, l'ESP va redémarrer.");
    digitalWrite(BUILTIN_LED, LOW);
    delay(1000);
    ESP.restart();
  }
}

//Fonction de connexion au broker MQTT
void MQTTconnect() {
  
  while (!client1.connected() && WiFi.status() == WL_CONNECTED) {
    Serial.println("Tentative de connexion MQTT...");
    if (client1.connect("esp8266", mqtt_username, mqtt_password)) {
      Serial.println("Connecté au serveur MQTT");
      //client.subscribe(mqtt_topic);
      digitalWrite(BUILTIN_LED, HIGH);
    } else {
      digitalWrite(BUILTIN_LED, LOW);
      Serial.print("Échec, rc=");
      Serial.print(client1.state());
      Serial.println(" Réessayer dans 5 secondes");
      delay(5000);
    }
  }
 Serial.println("Impossible de se connecter au broker MQTT (serveur injoignable ou WiFi non connecté).");
}


void setup() {
  // Initialisation de la liaison série
  Serial.begin(115200);
  pinMode(BUILTIN_LED, OUTPUT);

  // Initialisation de la connexion WiFi
  WiFi.begin(ssid, password);
  wifiConnect();

  // Initialisation du client MQTT
  client1.setServer(mqtt_server, 1883);
  //client1.setCallback(callback);
  
  // Initialisation serveur UDP
  udp.begin(port);
  Serial.println("Serveur UDP OK");

  // Initialisation de RC Switch
  mySwitch.enableReceive(RFrxPin);  // Broche de réception du signal radio, récepteur RF sur GPIO4 (D2)

  // ******* MAJ OTA ******** /

  ArduinoOTA.setHostname("ESP_RFGateway"); // Nom du module (affiché dans les ports en ligne d' l'IDE Arduino)
  
    ArduinoOTA.onStart([]() {
    String type;
    if (ArduinoOTA.getCommand() == U_FLASH) {
      type = "sketch";
    } else { // U_FS
      type = "filesystem";
    }
    Serial.println("Start updating " + type);
  });
  ArduinoOTA.onEnd([]() {
    Serial.println("\nEnd");
  });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) {
      Serial.println("Auth Failed");
    } else if (error == OTA_BEGIN_ERROR) {
      Serial.println("Begin Failed");
    } else if (error == OTA_CONNECT_ERROR) {
      Serial.println("Connect Failed");
    } else if (error == OTA_RECEIVE_ERROR) {
      Serial.println("Receive Failed");
    } else if (error == OTA_END_ERROR) {
      Serial.println("End Failed");
    }
  });
  ArduinoOTA.begin();
  Serial.println("Fonction OTA OK");
  

  Serial.println("Fin d'initialisation...");
}

void loop() {

  ArduinoOTA.handle();

  //Maintenir la connexion WIFI
  if (WiFi.status() != WL_CONNECTED) {
    wifiConnect();
  }

     // Maintenir la connexion MQTT
  if (!client1.connected()) {
    MQTTconnect();
  }

 //// RECEPTION PAR VOIE RF /////////
 
  if (mySwitch.available()) { // Dès la réception d'un code par voie RF
    
    unsigned long receivedValueRF = mySwitch.getReceivedValue();
    
    if (receivedValueRF != 0) {
      Serial.print("Code reçu par RF : ");
      Serial.println(receivedValueRF);
    
      // Publier le code radio sur le serveur MQTT, envoi d'un même code par 1000ms (1sec) maximum
      
    unsigned long currentMillis = millis();    
   
    if (receivedValueRF != previousCode || currentMillis - lastSendMQTT > delayMinPublishMQTT) { // Pas d'envoi MQTT si le code reçu par RF vient d'être envoyé par RF ou UDP (code identique) dans les 1 sec auparavant.
    previousCode = receivedValueRF;
    lastSendMQTT = millis();
      if (client1.connect("esp8266", mqtt_username, mqtt_password)) {
        client1.publish(mqtt_topic, String(receivedValueRF).c_str()); //Publication du code reçu sur le topic mqtt défini précédemment
        Serial.println("Code radio publié sur MQTT (via réception RF)");
      } else {
        Serial.println("Erreur de connexion MQTT");
      }
    }
      mySwitch.resetAvailable();
    }
  }

//// RECEPTION UDP PAR REPETEUR /////////
  
  int packetSize = udp.parsePacket();
  if (packetSize) {
    int lenUdp = udp.read(messageUdp, 255);
    if (lenUdp > 0)
    {
      messageUdp[lenUdp] = '\0';
    }
    unsigned long receivedValueUDP = atoi(messageUdp);
    Serial.print("Code radio reçu par UDP : ");
    Serial.println(receivedValueUDP);

    // Publier le code reçu par UDP sur le serveur MQTT, envoi d'un même code par 1000ms (1 sec) max
        
    unsigned long currentMillis2 = millis();
    if (receivedValueUDP != previousCode || currentMillis2 - lastSendMQTT > delayMinPublishMQTT) { // Pas d'envoi MQTT si le code reçu par UDP vient d'être envoyé par RF ou UDP (code identique) dans les 1,5 sec auparavant.
    previousCode = receivedValueUDP;
    lastSendMQTT = millis();
      if (client1.connect("esp8266", mqtt_username, mqtt_password)) {
        client1.publish(mqtt_topic, String(receivedValueUDP).c_str());
        Serial.println("Code radio publié sur MQTT (via réception UDP)");
      } else {
        Serial.println("Erreur de connexion MQTT");
      }
    }
       
}
  // Gestion des messages MQTT
  client1.loop();
}
```

### 3.1.2 ESP_RFextender  :

```cpp
// Code pour répéteur, envoit tous les codes reçu à la passerelle (RFGateway) via WiFi par protocole UDP
// Pas de temporisation, 1 code reçu = 1 code envoyé (temporisation faite par la passerelle)

#include <RCSwitch.h>
#include <ESP8266WiFi.h>
#include <ArduinoOTA.h>
#include <WiFiUdp.h>

// Variables WiFi
const char* ssid = "XX"; // A modifier avec vos propres identifiants
const char* password = "XX"; // A modifier avec vos propres identifiants

//Variables UDP
const IPAddress ip(192, 168, 1, 48); // Adresse IP de ESP_RFGateway qui va recevoir le code via UDP
const int port = 1996;


//Variables diverses
const int RFrxPin = 4; //Pin du récepteur RF : D2


// Configuration RC Switch
RCSwitch mySwitch = RCSwitch();

// Configuration UDP
WiFiUDP udp;

//Fonction de connexion Wifi
void wifiConnect() {

  WiFi.mode(WIFI_STA);             
  Serial.print("Connexion à ");
  Serial.print(ssid); Serial.println(" ...");

  unsigned long millisWifiStart = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - millisWifiStart < 1*20*1000) { //Délai de 20 secondes pour se connecter au Wi-Fi, autrement l'ESP8266 redémarre
    delay(100);
    Serial.print(".");
  }
  if (WiFi.status() == WL_CONNECTED) {
  Serial.println('\n');
  Serial.println("Connection établie !");  
  Serial.print("Adresse IP :\t");
  Serial.println(WiFi.localIP());
  digitalWrite(BUILTIN_LED, HIGH);
  }
  else {
    Serial.println("Impossible de se connecter au réseau Wi-Fi, l'ESP va redémarrer.");
    digitalWrite(BUILTIN_LED, LOW);
    delay(1000);
    ESP.restart();
  }
}


void setup() {
  // Initialisation de la liaison série
  Serial.begin(115200);
  pinMode(BUILTIN_LED, OUTPUT);

  // Initialisation de la connexion WiFi
  WiFi.begin(ssid, password);
  wifiConnect();

 
  // Initialisation serveur UDP
  udp.begin(port);
  Serial.println("Sereveur UDP OK");

  // Initialisation de RC Switch
  mySwitch.enableReceive(RFrxPin);  // Broche de réception du signal radio, récepteur RF sur GPIO4 (D2)
  
  // ******* MAJ OTA ******** /

  ArduinoOTA.setHostname("ESP_RFextender"); // Nom du module (affiché dans les ports en ligne d' l'IDE Arduino)
  
    ArduinoOTA.onStart([]() {
    String type;
    if (ArduinoOTA.getCommand() == U_FLASH) {
      type = "sketch";
    } else { // U_FS
      type = "filesystem";
    }
    Serial.println("Start updating " + type);
  });
  ArduinoOTA.onEnd([]() {
    Serial.println("\nEnd");
  });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) {
      Serial.println("Auth Failed");
    } else if (error == OTA_BEGIN_ERROR) {
      Serial.println("Begin Failed");
    } else if (error == OTA_CONNECT_ERROR) {
      Serial.println("Connect Failed");
    } else if (error == OTA_RECEIVE_ERROR) {
      Serial.println("Receive Failed");
    } else if (error == OTA_END_ERROR) {
      Serial.println("End Failed");
    }
  });
  ArduinoOTA.begin();
  Serial.println("Fonction OTA OK");


  Serial.println("Fin d'initialisation...");
}

void loop() {

  ArduinoOTA.handle();

  //Maintenir la connexion WIFI
  if (WiFi.status() != WL_CONNECTED) {
    wifiConnect();
  }

   
 //// RECEPTION PAR VOIE RF ET ENVOI A LA PASSERELLE /////////
 
  if (mySwitch.available()) {    // Dès la réception d'un code par voie RF
    
    unsigned long receivedValueRF = mySwitch.getReceivedValue();
    
    if (receivedValueRF != 0) {
      Serial.print("Code radio reçu par RF : ");
      Serial.println(receivedValueRF);

      int code2UDP = receivedValueRF; //Stockage du code RF reçu dans une variable int pour envoi par UDP
      // Envoi UDP
      udp.beginPacket(ip, port);
      udp.print(code2UDP);
      udp.endPacket();
      Serial.print("Code RF envoyé par UDP à la passerelle RF principale : ");
      Serial.println(code2UDP);
      
      mySwitch.resetAvailable();
    }
  }

}
```



# 4. Hardware (ESP_RFgateway & ESP_RFextender)

La câblage de la passerelle et des répéteurs est identique et simple :


| Broche SYN480R    | Broche ESP8266  | Description                     |
|-------------------|-----------------|---------------------------------|
| VCC               | 3.3V            | Alimentation (3.3V obligatoire)|
| GND               | GND             | Masse                           |
| DATA OUT          | GPIO4 (D2)      | Sortie des données RF           |
| ANT               | -               | Antenne (17,3 cm)               |


![Câblage](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/Cablage.PNG)

Pour améliorer la réception, vous pouvez ajouter une antenne de 17,3 cm (quart de longueur d'onde) à l'aide d'un câble silicone 26 awg à passer dans un morceau de gaine thermo pour la rigidifier (pas besoin de chauffer).
J'ai utilisé le connecteur rapide de l'ESP, mais vous pouvez évidemment y souder directement les fils.

Si le boitier vous intéresse, les STL sont disponibles sur le dépôt (PLA/PLA+ ; buse 0,4mm ; couches 0,28 mm).



# 5. Intégration à Home Assistant


## 5.1 Réception des messages MQTT

Le code suivant est à ajouter dans le fichier configuration.yaml de Home Assistant :

```yaml
mqtt:
  sensor:
#
# PASSERELLE RF POUR INTERRUPTEURS
#
    - name: "mqtt_rfgateway_code"
      unique_id: "mqtt_rfgateway_code"
      state_topic: "rfgateway/code"
```
N'oubliez pas de redémarrer votre instance.

Ajoutez ensuite temporairement une carte entité sur votre dashboard afin de visualiser facilement les code reçus (utile pour vérifier le bon fonctionnement du dispositif et noter les codes pour les associer à des actions par la suite) :

```yaml
type: entities
entities:
  - sensor.mqtt_rfgateway_code
```

![HA entity card](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/HA_entity_card.PNG)

Et voilà ! Vous devriez normalement maintenant recevoir les codes de vos interrupeurs RF dans Home Assistant !


## 5.2 Méthodologie d'association des interrupteurs

Purement informative, cette partie a pour but de vous proposer une méthodologie d'association des interrupteurs pour vos actions dans Home Assistant.

Dans un premier temps, il va falloir identifier vos interrupteurs et leurs voies, et donc leur donner des noms :

![Numérotation des voies](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/Num%C3%A9rotation_voies.PNG)

Ensuite, je vous conseille de prédéfinir les actions souhaitées pour chaque voie et de les noter dans un tableau à conserver (croyez moi, vous serez contents de l'avoir si vous devez vous replonger dedans quelques mois plus tard pour en ajouter/modifier... !)

![Tableau associations](https://github.com/Quentin-33/esp8266-simple-rf-to-mqtt-gateway/blob/main/IMAGES/Tableau_associations.PNG)

Enfin, créez vos automatisations qui, dès la réception d'un message MQTT sur le topic "rfgateway/code", exécuteront vos actions en fonction du code RF reçu.
Pour ma part, j'ai choisi de faire une automatisation par type d'action afin de ne pas surgcharger le listing dans Home Assistant. Mais vous pouvez très bien créer une automatisation par code RF !


## 5.3 Mes exemples d'automatisations

### 5.3.1 Contrôle des lumières par RF :

Note : J'ai prévu des entrées pour mes interrupteurs en stock non utilisés pour pouvoir ajouter des commande facilement par la suite, d'où certaines séquences avec des entity_id nulles.

```yaml
alias: Contrôle des interrupteurs RF > MQTT V3
description: ""
triggers:
  - topic: rfgateway/code
    trigger: mqtt
conditions: []
actions:
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '864674' }}"
            alias: 1C.1
        sequence:
          - entity_id: null
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '864676' }}"
            alias: 1C.2
        sequence:
          - entity_id: null
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '864673' }}"
            alias: 1C.3
        sequence:
          - data: {}
            target:
              entity_id: null
            action: scene.turn_on
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '8576162' }}"
            alias: 2B.1
        sequence:
          - entity_id: switch.salon_plafonnier_zb
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '8576161' }}"
            alias: 2B.2
        sequence:
          - entity_id: null
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '11089876' }}"
            alias: 3C.1
        sequence:
          - entity_id: switch.salon_plafonnier_zb
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '11089873' }}"
            alias: 3C.2
        sequence:
          - entity_id: switch.sonoff_1_lt, switch.sonoff_1
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '11089874' }}"
            alias: 3C.3
        sequence:
          - entity_id: switch.sonoff_2_lt, switch.sonoff_2
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '867620' }}"
            alias: 5A.1
        sequence:
          - data:
              brightness: 255
            target:
              entity_id: light.chambre_lt, light.ampoule_chambre
            action: light.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '6221682' }}"
            alias: 6C.1
        sequence:
          - data: {}
            target:
              entity_id: light.chambre_lt, light.ampoule_chambre
            action: light.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '6221684' }}"
            alias: 6C.2
        sequence:
          - data: {}
            target:
              entity_id: scene.chambre_25
            action: scene.turn_on
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '6221681' }}"
            alias: 6C.3
        sequence:
          - data: {}
            target:
              entity_id: scene.chambre_100
            action: scene.turn_on
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '12863812' }}"
            alias: 7A.1
        sequence:
          - entity_id: null
            action: switch.toggle
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '5063816' }}"
            alias: 8B.1
        sequence:
          - data: {}
            target:
              entity_id: scene.all_light_off
            action: scene.turn_on
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '9167912' }}"
            alias: 11B.1
        sequence:
          - data: {}
            target:
              entity_id: scene.etage_off
            action: scene.turn_on
      - conditions:
          - condition: template
            value_template: "{{ trigger.payload == '9167906' }}"
            alias: 11B.2
        sequence:
          - data: {}
            target:
              entity_id: scene.rdc_off
            action: scene.turn_on
mode: single

```

### 5.3.2 Contrôle des lumières par RF :

Note : un appui sur l'interrupteur permet à la fois de lancer le nettoyage de la cuisine si le robot n'est pas déjà en fonctionnement, ou de le faire retourner à sa base pour annuler/écourter la tâche.

```yaml
alias: ASPIRATEUR - Nettoyage cuisine bouton RF
description: ""
mode: single
triggers:
  - topic: rfgateway/code
    payload: "12863812"
    trigger: mqtt
conditions: []
actions:
  - if:
      - condition: state
        entity_id: vacuum.dreame_bot_d9_max
        state: docked
    then:
      - data: {}
        target:
          entity_id: script.aspirateur_cuisine_auto
        action: script.turn_on
  - if:
      - condition: state
        entity_id: script.aspirateur_cuisine_auto
        state: "on"
    then:
      - data: {}
        target:
          entity_id: vacuum.dreame_bot_d9_max
        action: vacuum.return_to_base
```
