# Utveksling av SUTI-meldinger via RabbitMQ
## Introduksjon
Følgende dokument har som formål å dokumentere transportørers bruk av RabbitMQ i SUTI-kommunikasjon mot NISSY og CTRL.

Meldingsflyten er tilsvarende slik det tidligere har fungert med SonicMQ. SUTI-meldinger fra NISSY/CTRL sendes til transportør-spesifikke utgående køer, og meldinger fra transportører til NISSY/CTRL sendes til én felles inngående kø.

RabbitMQ er et åpent kildekode-prosjekt som er basert på AMQP-protokollen. Både biblioteker og dokumentasjon er åpent tilgjengelig på prosjektets nettside (https://www.rabbitmq.com/).
Dette dokumentet er derfor  begrenset til å inneholde informasjon om kønavn og egenskaper som skal til for å koble seg til, hente ut, og sende inn meldinger til Pasientreisers RabbitMQ-tjeneste.

Kodeeksemplene i dokumentet er skrevet i Python og benytter biblioteket pika (https://pika.readthedocs.io/en/stable/). Eksemplene er kun ment som et hjelpemiddel for å komme i gang med utvikling mot RabbitMQ, og er ikke egnet til å kjøres i produksjon.


## Tilkoblingsinformasjon
Brukernavn og passord oversendes personen som er oppgitt som teknisk ansvarlig i bestillingsskjemaet. Systemets eksterne IP-adresser må være forhåndsgodkjent av Norsk Helsenett for at det kan etableres kontakt mot tjenesten.

| **Egenskap** |   **Verdi**   |
|-----|-----|
|   **Protokoll**  |  AMQPS   |
| **Host** | rabbitmq.nhn.no |
| **Port** | 5671 |
| **SSL** | TLS v1.2 |
| **VHost** | nissy / ctrl |


```python
def connect():
    credentials = pika.PlainCredentials('pasientreiser0002.918695079', '<passord>')
    context = ssl.SSLContext(ssl.PROTOCOL_TLSv1_2)
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(
            host='suti.test.pasientreiser.nhn.no',
            port='5671',
            virtual_host='nissy',
            credentials=credentials,
            ssl_options=pika.SSLOptions(context),
            client_properties={
                'connection_name': 'Pasientreisers taksametersystem',
        })
    )
```


## Transportørkøer
Inndelingen av køer baserer seg på at hver systemleverandør skal ha én kø per transportør, per system. Navnestandarden baserer seg på følgende format.

| System | Prefiks        | Del 1                                       | Del 2                                 | Eksempel                              |
|--------|----------------|---------------------------------------------|---------------------------------------|---------------------------------------|
| NISSY  | nissy.out.rmr. | (taksameterleverandør)(4-sifret løpenummer) | .(transportørens organisasjonsnummer) | nissy.out.rmr.tds0001.980650170       |
| CTRL   | ctrl.out.rmr.  | (systemleverandør)(4-sifret løpenummer)     | .(transportørens organisasjonsnummer) | ctrl.out.rmr.taxifinans0001.993217654 |



# Publisering av meldinger
Meldinger som sendes til NISSY eller CTRL er satt opp med en utvekslingstype som heter "Direct Exchange". Dette betyr at meldinger rutes til riktig kø basert på en "message routing key".

| **Egenskap** | **Verdi NISSY**         | **Verdi CTRL**      |
|--------------|-------------------------|---------------------|
| Exchange     | ""                      | ""                  |
| Routing key  | nissy.in.rmr.sutiapprec | ctrl.in.rmr.oppgjor |
| Body         | (SUTI-melding)          | (SUTI-melding)      |

For å verifisere at meldinger har blitt publisert på riktig kø, er det anbefalt at klienter benytter “Publisher confirms". På nettsiden til RabbitMQ er konseptet godt forklart (https://www.rabbitmq.com/confirms.html).

```python
def publish(connection):
    channel = connection.channel()
    channel.confirm_delivery()
    
    try: 
        channel.basic_publish(exchange='',
          routing_key='nissy.in.rmr.sutiapprec',
          body=suti_msg_2001, # UTF-8
          properties=pika.BasicProperties(content_type='text/plain', delivery_mode=pika.DeliveryMode.Transient))
          print("Melding publisert på køen")
          
    except pika.exceptions.UnroutableError:
        print("Melding kunne ikke publiseres på køen")
```

# Konsumering av meldinger
Meldinger som sendes fra NISSY eller Ctrl blir lagt på transportørens kø.
For å unngå tap av data ved driftsstans er det ved uthenting av meldinger fra kø anbefalt å bekrefte mottak først etter at meldingen er mottatt og prosessert riktig. På nettsiden til RabbitMQ er konseptet godt forklart (https://www.rabbitmq.com/confirms.html#consumer-acknowledgements), og det finnes kodeeksempler for en rekke forskjellige språk i den offisielle veiledingen under kapittel "2 - Work queues - Message acknowledgement" (https://www.rabbitmq.com/getstarted.html).

```python
def consume(channel):

    def callback(channel, method, properties, body):
        print(body.decode("utf-8"))
        channel.basic_ack(delivery_tag = method.delivery_tag) # Bekreft mottak
        
    
    channel.basic_consume(queue='nissy.out.rmr.pastrans0002.918695079',
                          auto_ack=False,
                          on_message_callback=callback)
    channel.start_consuming()
```

# SUTI Metadata
Ved overgang til ny kø må man oppdatere “SUTI link” i meldingens metadata til å inneholde den nest siste delen av det nye kønavnet, altså <leverandør><løpenummer>. Eksempel: Dersom kønavn er "nissy.out.rmr.tds0001.980650170" vil SUTI-link være "tds0001".

```xml
# NISSY
<orgSender name="Tds">
  <idOrg src="SUTI:idLink" id="tds0001" />
</orgSender>
<orgReceiver name="Pasientreiser">
  <idOrg src="SUTI:idLink" id="pasientreiser_nissy" />
</orgReceiver>

# CTRL
<orgSender name="Tds">
  <idOrg src="SUTI:idLink" id="tds0001" />
</orgSender>
<orgReceiver name="Pasientreiser">
  <idOrg src="SUTI:idLink" id="oppgjor" />
</orgReceiver>
```

I tillegg er det for CTRL viktig at organisasjonsnummeret til transportøren i kønavnet samsvarer med det som er oppgitt i SUTI-meldingens `orgProvider.idOrg.id`-element. Dette grunnet at Ctrl benytter dette elementet til å utlede hvilken transportørkø det skal sendes svar til.
```xml
<orgProvider orgName="Oslo Taxi AS">
<idOrg src="NO:idOrg" id="972413615" />
</orgProvider>
```

