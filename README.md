## MQTT (Message Queue Telemetry Transport)
***

> [⇧ **Home**](https://github.com/iotkitv3/intro)

![](https://raw.githubusercontent.com/iotkitv3/intro/main/images/MQTTPubSub.png)

Quelle: Publish/Subscribe-Architektur von MQTT. © HiveMQ.com
- - -

[Message Queue Telemetry Transport (MQTT)](http://de.wikipedia.org/wiki/MQ_Telemetry_Transport) ist ein offenes Nachrichten-Protokoll für Machine-to-Machine-Kommunikation (M2M), das die Übertragung von Messdaten-Daten in Form von Nachrichten zwischen Geräten ermöglicht, trotz hoher Verzögerungen oder beschränkten Netzwerken. Entsprechende Geräte reichen von Sensoren und Aktoren, Mobiltelefonen, Eingebetteten Systemen in Fahrzeugen oder Laptops bis zu voll entwickelten Rechnern. **MQTT basiert auf TCP-Sockets.**

MQTT implementiert das [**Publish/Subscribe-Pattern**](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern). Es ersetzt die Punkt-zu-Punkt-Verbindungen durch einen zentralen Server (Broker), zu dem sich Datenproduzenten und -nutzer gleichermaßen verbinden können. Das Senden (publish) und Empfangen (subscribe) von Nachrichten funktioniert über sogenannte Topics. Ein **Topic** ist ein String, der eine Art Betreff der Nachricht darstellt, aber ähnlich einer Web Adresse aufgebaut ist.

Im obigen Beispiel funktioniert die komplette Kommunikation rein über Topics, und der Sensor (links) und die Endgeräte (rechts) wissen nichts über die Existenz des jeweils anderen.

### Beispiel Topics

	Zuhause/Wohnzimmer/Temperatur
	Zuhause/Wohnzimmer/Luftfeuchtigkeit
	Zuhause/Schlafzimmer/Temperatur
	Zuhause/Schlafzimmer/Luftfeuchtigkeit						

Topics haben noch ein weiteres wichtiges Konzept - Wildcards. In oben stehenden Codebeispiel sind vier Topics aufgelistet, und je ein Sensor sendet eine neue Nachricht auf den jeweiligen Topic, sobald sich ein Wert geändert hat. Man kann nun je nach Anwendungsfall Wildcards benutzen, um mehrere Topics zu abonnieren.

### Beispiel Wildcards 

	Zuhause/+/Temperatur
	Zuhause/Wohnzimmer/#
	#

Oben sind alle möglichen Wildcard-Operatoren aufgelistet. Im ersten Fall bekommt die mobile Anwendung nur alle Nachrichten über neue Temperaturwerte, im zweiten Fall nur alle Werte aus dem Wohnzimmer und im dritten Fall alle Werte. Dabei lässt sich der +-Operator immer nur für eine Hierarchiestufe einsetzen und der #-Operator für beliebig viele Hierarchiestufen mit der Bedingung, dass dieser am Ende stehen muss.

### Mechanismen zur Qualitätskontrolle 

Ein weiteres wichtiges Konzept sind die drei Servicequalitäten bei der Datenübertragung 0, 1 und 2. Die Zusicherung variiert von keiner Garantie (Level 0) über die, dass die Nachricht mindestens einmal ankommt (Level 1), bis hin zur Garantie, dass die Nachricht genau einmal ankommt (Level 2).

## MQTT Publish Beispiel
***

Mittels Publish (unten) kann eine Meldung zum MQTT Broker bzw. Topic gesendet werden.

Ein anderes Board oder der Mosquitto Client mosquitto_sub kann dieses Topic Abonnieren (subscribe).

### Client

Mittels der Client Utilities von [Mosquitto](https://projects.eclipse.org/projects/technology.mosquitto) können Werte abgefragt oder gesendet werden.

Beispiel: Abfragen der Werte von Temperatur und Luftfeuchtigkeit (Ausgabe: I2C Id, Temperatur, Luftfeuchtigkeit, Motorlaufgeschwindigkeit)

    mosquitto_sub -h cloud.tbz.ch -t iotkit/sensor/#
    0xBC,22.90,36.9,low
    0xBC,28.00,36.7,middle
    0xBC,32.90,36.7,high

Beispiel: Abfragen ob jemand einen Magneten an den Hall Sensor gehalten hat
    
    mosquitto_sub -h cloud.tbz.ch -t iotkit/alert/#
    
Beispiel: Setzen der Servo Position (0.0 - 1.0) 

    mosquitto_pub -h cloud.tbz.ch -t "iotkit/actors/servo2" -m "0.1" -q 0    

**Hinweis**: Der `broker.mqttdashboard.com` lässt leider nur 1ne Connection zu. Deshalb sollte dieser im Beispiel geändert werden (Eintrag `hostname` in `MQTTPublish/src/main.cpp`). Eine Liste von Öffentlichen MQTT gibt es [hier](https://github.com/mqtt/mqtt.github.io/wiki/public_brokers).

### Beispiel(e)

* [Sensordaten mittels MQTT publizieren](#mqttpublish)
* [MQTT Workflow mit Node-RED](#node-red-mqtt-workflow)

#### MQTTPublish
***

> [⇧ **Nach oben**](#beispiele)

Das Beispiel [MQTTPublish](main.cpp) sendet Sensordaten an einen MQTT Broker und wartet auf dem Topic `iotkit/actors/servo2` auf Anforderungen für einen Servo (siehe `mosquitto_pub` Beispiel oben).

Sollte das Beispiel nicht sauber funktionieren (Subscribe Topic), kann die alte Variante ohne Subscribe verwendet werden.

<details><summary>main.cpp</summary>  


    /** MQTT Publish von Sensordaten */
    #include "mbed.h"
    #include "OLEDDisplay.h"
    #include "Motor.h"
    
    #if MBED_CONF_IOTKIT_HTS221_SENSOR == true
    #include "HTS221Sensor.h"
    #endif
    #if MBED_CONF_IOTKIT_BMP180_SENSOR == true
    #include "BMP180Wrapper.h"
    #endif
    
    #ifdef TARGET_K64F
    #include "QEI.h"
    #include "MFRC522.h"
    
    // NFC/RFID Reader (SPI)
    MFRC522    rfidReader( MBED_CONF_IOTKIT_RFID_MOSI, MBED_CONF_IOTKIT_RFID_MISO, MBED_CONF_IOTKIT_RFID_SCLK, MBED_CONF_IOTKIT_RFID_SS, MBED_CONF_IOTKIT_RFID_RST ); 
    //Use X2 encoding by default.
    QEI wheel (MBED_CONF_IOTKIT_BUTTON2, MBED_CONF_IOTKIT_BUTTON3, NC, 624);
    #endif
    
    #include <MQTTClientMbedOs.h>
    #include <MQTTNetwork.h>
    #include <MQTTClient.h>
    #include <MQTTmbed.h> // Countdown
    
    // Sensoren wo Daten fuer Topics produzieren
    static DevI2C devI2c( MBED_CONF_IOTKIT_I2C_SDA, MBED_CONF_IOTKIT_I2C_SCL );
    #if MBED_CONF_IOTKIT_HTS221_SENSOR == true
    static HTS221Sensor hum_temp(&devI2c);
    #endif
    #if MBED_CONF_IOTKIT_BMP180_SENSOR == true
    static BMP180Wrapper hum_temp( &devI2c );
    #endif
    AnalogIn hallSensor( MBED_CONF_IOTKIT_HALL_SENSOR );
    DigitalIn button( MBED_CONF_IOTKIT_BUTTON1 );
    
    // Topic's
    char* topicTEMP = (char*) "iotkit/sensor";
    char* topicALERT = (char*) "iotkit/alert";
    char* topicBUTTON = (char*) "iotkit/button";
    char* topicENCODER = (char*) "iotkit/encoder";
    char* topicRFID = (char*) "iotkit/rfid";
    // MQTT Brocker
    char* hostname = (char*) "cloud.tbz.ch";
    int port = 1883;
    // MQTT Message
    MQTT::Message message;
    // I/O Buffer
    char buf[100];
    
    // Klassifikation 
    char cls[3][10] = { "low", "middle", "high" };
    int type = 0;
    
    // UI
    OLEDDisplay oled( MBED_CONF_IOTKIT_OLED_RST, MBED_CONF_IOTKIT_OLED_SDA, MBED_CONF_IOTKIT_OLED_SCL );
    DigitalOut led1( MBED_CONF_IOTKIT_LED1 );
    DigitalOut alert( MBED_CONF_IOTKIT_LED3 );
    
    // Aktore(n)
    Motor m1( MBED_CONF_IOTKIT_MOTOR2_PWM, MBED_CONF_IOTKIT_MOTOR2_FWD, MBED_CONF_IOTKIT_MOTOR2_REV ); // PWM, Vorwaerts, Rueckwarts
    PwmOut speaker( MBED_CONF_IOTKIT_BUZZER );
    
    /** Hilfsfunktion zum Publizieren auf MQTT Broker */
    void publish( MQTTNetwork &mqttNetwork, MQTT::Client<MQTTNetwork, Countdown> &client, char* topic )
    {
        led1 = 1;
        printf("Connecting to %s:%d\r\n", hostname, port);
        
        int rc = mqttNetwork.connect(hostname, port);
        if (rc != 0)
            printf("rc from TCP connect is %d\r\n", rc);
    
        MQTTPacket_connectData data = MQTTPacket_connectData_initializer;
        data.MQTTVersion = 3;
        data.clientID.cstring = (char*) "mbed-sample";
        data.username.cstring = (char*) "testuser";
        data.password.cstring = (char*) "testpassword";
        if ((rc = client.connect(data)) != 0)
            printf("rc from MQTT connect is %d\r\n", rc);
    
        MQTT::Message message;    
        
        oled.cursor( 2, 0 );
        oled.printf( "Topi: %s\n", topic );
        oled.cursor( 3, 0 );    
        oled.printf( "Push: %s\n", buf );
        message.qos = MQTT::QOS0;
        message.retained = false;
        message.dup = false;
        message.payload = (void*) buf;
        message.payloadlen = strlen(buf)+1;
        client.publish( topic, message);  
        
        // Verbindung beenden, ansonsten ist nach 4x Schluss
        if ((rc = client.disconnect()) != 0)
            printf("rc from disconnect was %d\r\n", rc);
    
        mqttNetwork.disconnect();
        led1 = 0;
    }
    
    /** Hauptprogramm */
    int main()
    {
        uint8_t id;
        float temp, hum;
        int encoder;
        alert = 0;
        
        oled.clear();
        oled.printf( "MQTTPublish\r\n" );
        oled.printf( "host: %s:%s\r\n", hostname, port );
    
        printf("\nConnecting to %s...\n", MBED_CONF_APP_WIFI_SSID);
        oled.printf( "SSID: %s\r\n", MBED_CONF_APP_WIFI_SSID );
        
        // Connect to the network with the default networking interface
        // if you use WiFi: see mbed_app.json for the credentials
        WiFiInterface *wifi = WiFiInterface::get_default_instance();
        if ( !wifi ) 
        {
            printf("ERROR: No WiFiInterface found.\n");
            return -1;
        }
        printf("\nConnecting to %s...\n", MBED_CONF_APP_WIFI_SSID);
        int ret = wifi->connect( MBED_CONF_APP_WIFI_SSID, MBED_CONF_APP_WIFI_PASSWORD, NSAPI_SECURITY_WPA_WPA2 );
        if ( ret != 0 ) 
        {
            printf("\nConnection error: %d\n", ret);
            return -1;
        }    
    
        // TCP/IP und MQTT initialisieren (muss in main erfolgen)
        MQTTNetwork mqttNetwork( wifi );
        MQTT::Client<MQTTNetwork, Countdown> client(mqttNetwork);
        
        /* Init all sensors with default params */
        hum_temp.init(NULL);
        hum_temp.enable(); 
    
    #ifdef TARGET_K64F
        // RFID Reader initialisieren
        rfidReader.PCD_Init();  
    #endif
        
        while   ( 1 ) 
        {
            // Temperator und Luftfeuchtigkeit
            hum_temp.read_id(&id);
            hum_temp.get_temperature(&temp);
            hum_temp.get_humidity(&hum);    
            if  ( type == 0 )
            {
                temp -= 5.0f;
                m1.speed( 0.0f );
            }
            else if  ( type == 2 )
            {
                temp += 5.0f;
                m1.speed( 1.0f );
            }
            else
            {
                m1.speed( 0.75f );
            }
            sprintf( buf, "0x%X,%2.2f,%2.1f,%s", id, temp, hum, cls[type] ); 
            type++;
            if  ( type > 2 )
                type = 0;       
            publish( mqttNetwork, client, topicTEMP );
            
            // alert Tuer offen 
            printf( "Hall %4.4f, alert %d\n", hallSensor.read(), alert.read() );
            if  ( hallSensor.read() > 0.6f )
            {
                // nur einmal Melden!, bis Reset
                if  ( alert.read() == 0 )
                {
                    sprintf( buf, "alert: hall" );
                    message.payload = (void*) buf;
                    message.payloadlen = strlen(buf)+1;
                    publish( mqttNetwork, client, topicALERT );
                    alert = 1;
                }
                speaker.period( 1.0 / 3969.0 );      // 3969 = Tonfrequenz in Hz
                speaker = 0.5f;
                thread_sleep_for( 500 );
                speaker.period( 1.0 / 2800.0 );
                thread_sleep_for( 500 );
            }
            else
            {
                alert = 0;
                speaker = 0.0f;
            }
    
            // Button (nur wenn gedrueckt)
            if  ( button == 0 )
            {
                sprintf( buf, "ON" );
                publish( mqttNetwork, client, topicBUTTON );
            }
    
    #ifdef TARGET_K64F
    
            // Encoder
            encoder = wheel.getPulses();
            sprintf( buf, "%d", encoder );
            publish( mqttNetwork, client, topicENCODER );
            
            // RFID Reader
            if ( rfidReader.PICC_IsNewCardPresent())
                if ( rfidReader.PICC_ReadCardSerial()) 
                {
                    // Print Card UID (2-stellig mit Vornullen, Hexadecimal)
                    printf("Card UID: ");
                    for ( int i = 0; i < rfidReader.uid.size; i++ )
                        printf("%02X:", rfidReader.uid.uidByte[i]);
                    printf("\n");
                    
                    // Print Card type
                    int piccType = rfidReader.PICC_GetType(rfidReader.uid.sak);
                    printf("PICC Type: %s \n", rfidReader.PICC_GetTypeName(piccType) );
                    
                    sprintf( buf, "%02X:%02X:%02X:%02X:", rfidReader.uid.uidByte[0], rfidReader.uid.uidByte[1], rfidReader.uid.uidByte[2], rfidReader.uid.uidByte[3] );
                    publish( mqttNetwork, client, topicRFID );                
                    
                }        
    #endif        
    
            thread_sleep_for    ( 500 );
        }
    }
    
</p></details>

**Links** 

*   [Ausführlicher Artikel auf heise.de](http://www.heise.de/developer/artikel/MQTT-Protokoll-fuer-das-Internet-der-Dinge-2168152.html)
*   [Offizelle MQTT Library von mbed](https://github.com/ARMmbed/mbed-mqtt)
*   [MQTT Client Library Encyclopedia - Paho Embedded](https://www.hivemq.com/blog/mqtt-client-library-encyclopedia-paho-embedded/) - gute Einführung in mbed Library
*   [MQTT JavaScript Client Library für node.js und Browser](https://github.com/mqttjs/MQTT.js)
*   [Eclipse Paho, Client Libraries für Verschiedene Sprachen](http://www.eclipse.org/paho/)
*   [Practical MQTT with Paho](http://www.infoq.com/articles/practical-mqtt-with-paho)
*   [Paho UI Utilities für MQTT](https://wiki.eclipse.org/Paho/GUI_Utility)
*   [MQTT Toolbox - MQTT Client Chrome App](https://www.hivemq.com/blog/mqtt-toolbox-mqtt-client-chrome-app/) - einfacher MQTT Client um Meldungen zu abonnieren (subsribe).

#### Node-RED MQTT Workflow
***

> [⇧ **Nach oben**](#beispiele)

![](https://raw.githubusercontent.com/iotkitv3/intro/main/images/NodeREDMQTT.png)

- - -

* Benötigte Software installieren (z.B. auf einem Raspberry Pi oder einer Linux VM)
    * [Mosquitto](https://mosquitto.org/) - MQTT Broker.
    * [Node-RED](https://nodered.org/) - Workflow Engine.
    * [ngrok](https://ngrok.com/) für eine Public URL. Wenn z.B. der Raspberry Pi hinter einer Firewall ist.
* In Node-RED
    * `mqtt` Input Node auf Flow 1 platzieren, mit Mosquitto Server verbinden, als Topic `iotkit/#` und bei Output `a String` eintragen.
    * `debug` Output Node auf Flow 1 platzieren und mit Input Node verbinden.
    * Programm mittels `Deploy` veröffentlichen.
* mbed Teil
    * [MQTTPublish](#mqttpublish) Beispiel in mbed Compiler importieren und ca. auf Zeile 21 den `hostname` mit der IP-Adresse auswechseln wo der Mosquitto Server läuft. 
    * Programm Compilieren und auf Board laden.
    
**Daten an IoTKitV3 senden:**

* In Node-RED
    * `inject` Node (2x mal) auf Flow 1 platzieren. Nutzdaten "String" und zwei unterschiedliche Werte, z.B. 0.2 und 0.9 eingeben. Als Topic `iotkit/actors/servo2` eintragen
    * `mqtt out` Node auf Flow 1 platzieren. Gleicher MQTT Server, wie oben, eintragen. Die restlichen Felder auf Standardwerte belassen.

Servo an IoTKitV3 (oranges Kabel auf `S`) an Servo 2 Anschluss (in der Regel `D9`) anschliessen.

Durch abwechselendes Drücken auf die zwei `inject` Nodes wird der Servo hin und her bewegt.

**Links**
 
 * [Home Page](https://nodered.org/)
 * [Node-RED Einführung](https://www.youtube.com/watch?v=f5o4tIz2Zzc)


