# Technical Design

Dette er en beskrivelse af hvordan løsningen skal implementeres. 

## Beskrivelse af Data flow
At oprette en ny vare eller oprette en ny salgsordre er *ikke* en hændelse, der trigger en hændelse mod lagerhotellet. 

Brugeren skal specifikt indikere at varen er klar til at blive oprettet i lagerhotellet. 
Ligeledes skal brugeren indikere at ordren er klar til levering før en besked bliver sendt til lagerhotellet.

Kunder oprettes i lagerhotellet samtidigt med oprettelse af ordren. 

## Event Log
Først når man er klar til at oprette en vare eller til at levere salgsordre, oprettes en Event i Eventloggen med status klar `Ready`.

## Event Handler
En event-handler kan køre automatisk eller fyres af manuelt. 
Event-handleren leder efter events som er klar til at blive behandlet. 
I opsætningen skal man kunne angive med hvilken frekvens event-handleren skal køre. 

En event skal oversættes til en XML body, som Event-handleren sender i et HTTP POST request til lagerhotellet og modtager et svar. 

*Formatet af hvert af de seks reqeusts er beskrevet i detaljer med eksempler i afsnittet HTTP requests herunder.*

### Dobbelt hændelser
Det kan forekomme at man opdaterer en vare eller en salgsordre flere gange inden Event-handleren kommer til at behandle hændelsen. 
Det kan ske hvis Event Handleren har været stoppet i en periode. 

Det er vigtigt, at ved behandlingen af en event at det sikres at man behandler den seneste og at alle andre tidligere 
events vedrørende den samme vare eller ordre, får en status `Udløbet`, der gør at de bliver ignorert fremover. 

Alternativt, kan man ved oprettelsen af hændelsen sikre sig at en eksisterende event bliver opdateret i stedet for at oprettet en ny.


### Preconditions
Det er vigtigt først og fremmest at forstå hvorfor et request kan fejle og fra starten forsøge at undgå disse situationer
før requests sendes. Det vil sige at Event Handleren skal kunne afvise en event, hvis denne ikke indeholder tilstrækkelige 
oplysninger for at requestet kan sendes med forventet succes. Dette kaldes for **preconditions**. 

EventLoggen's status felt skal altså have en option `Afvist` og med en begrundelse for hvorfor. 

Et eksempel på en precondition er at man ikke skal sælge en vare, der ikke først er oprettet i lagerhotellet. 
Eller at varen ikke er på lager i tilstrækkeligt antal for at kunne behandle ordren. 

Det er vigtig at vi kortlægger og behandler disse tilfælde i dokumentet Functional Specification. 

## Respons Handler
Respons-handleren behandler det svar, der kommer tilbage fra lagerhotellet. 

Svaret er i XML format, som beskrevet under HTTP requests herunder. 
Hvert request giver anledning til forskellige mulige svar svarende til requestet, herunder også fejl. 

Feltet `getResult` returnerer enten `Failed` eller `Succes`. 

### Succes
Når requestet lykkes har Respons Handleren en række opgaver. 

* Parse XML svaret.
* Hvis svaret indeholder data, så skal disse gemmes i tabeller tilhørende EventLoggen, behandles og evt. 
opdatere de tilsvarende tabeller i BC. 
* Eksempel: En vare oprettes med succes i Lagerhotellet. Dette kunne skrives tilbage til varekortet i BC.
* Eksempel: Lagerhotellet har expedieret en ordre. Dette kunne skrives tilbage til ordren i BC. 
* Og til sidst opdatere Eventloggen med Status `Succes`, hvis de interne operationer i BC også kunne udføres uden problemer. 
* Hvis svaret indeholder en `ReturnMessage` skrives denne også i EventLoggen. 
* I sidste instans bør systemet også kunne håndtere, at der går noget galt i opdateringen ved behandlingen af svaret af hensyn til 
efterfølgende support og vedligeholdes af systemt. For eksempel, hvis man skulle havne i en uventet situation. 

### Failed
Hvis requestet fejler, er det vigtigt at Respons Handleren kan opdatere EventLoggen med oplysninger om hvorfor noget gik galt. 
Hvis requestet ellers har den rette syntax, burde forklaringen komme tilbage i feltet `returnMessage`. 

Skulle lagerhotellets interface ikke svare tilbage, er det tilsvarende op til Respons-Handleren at håndtere denne situation. 

For eksempel på denne måde:

* Sende en e-mail.
* Suspendere Event Handleren i en periode. 
* Sikre at Eventen i EventLoggen bliver behandlet når Event Handleren igen bliver aktiveret. 

*Det skal beskrives i detaljer, hvad der skal ske for hvert af de seks requests og for hvert af de mulige reponses*

### Support og Log
EventLoggens rolle er dels at sikre en historik over de hændelser som gik godt og dels at hjælpe med håndteringen af de hændelser 
der ikke blev behandlet. Den er altså en vigtig støtte til dem der skal supportere systemet i det daglige og i at sikre at 
løsningen er robust. 

## Configuration and Setup

Opsætningsoplysninger kunne typisk være

* Frekvens for kørsel af Event Handler
* Emails på personer der skal informeres hvis noget går galt
* Sti til lagerhotellets webservice
* ...

### Test Setup
* _configuration: Ack#2009
* _company: test
* _encryptionkey: sd%#gg9HwT2
* _shopId: Test

### Production Setup
* _configuration: ????
* _company: ????
* _encryptionkey: ????
* _shopId: ????

## Fremtidige opgraderinger af Webservicen
APIer, alså interfaces, til systemer som Ackro's webservice udvikler sig over tid. 

Men det gøres altid på en bestemt måde. 

Hvis udbyderen af webservicen ønsker at ændre sit interface, for eksempel ved at tilføje et nyt felt til en eksisterende 
operation, gøres dette ved at tilbyde en ny opration. I Ackro's tilfælde kan man se at funktionen til at oprette
ordrehovder har ændres sig over tid. Den seneste operation hedder nu `setSalesOperations_V4`. 
Det er altså den fjerde version. Tilsvarende for operationen `getPackingSlipInfoExtended_V3`.

Disse ændringer vil også ske i fremtiden. Det gode er at når Ackro introduceret en ny version af setSalesOperations, 
så virker den gamle stadigvæk. Det vil sige at vi som forbrugere af Ackro's webservice har lidt tid til at implementere
den nye version.

Det er altså vigtigt at forstå at denne implementering tager udgangspunkt i de operationer som Ackro tilbyder lige nu 
og at den vil virke så længe Ackro understøtter disse interfaces. Skulle Ackro ønske ikke længere at tilbyde et 
gammelt interface, vil det være nødvendigt at upgradere løsningen til det nye interface. 



### Test mode and Production Mode
Setup in Test and Setup in Production. 

- Navn på produktions miliø
- Automatisk sikring at test miliø anvender test setup
- Produktionssetup
- Testsetup
- Indtænke Golive. 
- Sikre at backup miliø aldrig køre på produktionsdata. 

# HTTP Requests

## Header
Det er vigtig, især ved requests der opretter data at formatet er UTF-8. Dette gøres ved at tilføje følgende ligne til Headeren. 

**Content-Type: text/xml; charset=utf-8**

Dette sikre at varenavne, kontaktpersoner, adresser mm. der indeholder special tegn som æ, ø og å eller accenter som é eller ô 
bliver oprettet korrekt i Ackro's system. 

## getInventSum

`getInventSum` operationen returnere en liste med alle vare i lagerhotellet og kan anvendes til følgende formål

* Undersøge om en given vare oprettet i lagerhotellet
* Sammenligne om oplysningerne i lagerhotellet stemmer overens med de tilsvarende oplysninger i BC
* Varenavn: `ItemName` 
* Antal på lager: `AvailablePhysicalQty`
* Vares kostpris i Ackro's Axapta: `CostPrice`

Spørgsmålet er hvad skal disse oplysninger bruges til. 
Skal BC opdateres med de fysiske antal på laget? 
Skal varenavnet opdateres på lagerhotellet? 


### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <getInventSum xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_calcDate>2019-05-07T15:25:38+02:00</_calcDate>
    </getInventSum>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <getInventSumResponse xmlns="http://Ackro.dk/Services/2010">
            <InventSumRequest xmlns="http://www.dynacon.dk/2009/services/invent">
                <InventSum xmlns="">
                    <ItemId>00000001</ItemId>
                    <ItemName>Standard vare</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>-19</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>915</ItemId>
                    <ItemName>Test Testobject</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>-150</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>916</ItemId>
                    <ItemName>Test US 1</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>-1</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>917</ItemId>
                    <ItemName>Test Chew v 10</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>-10</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>922</ItemId>
                    <ItemName>Test Chew</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>928</ItemId>
                    <ItemName>Test Chew Copy</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>Handling</ItemId>
                    <ItemName>Handling</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>Shipping</ItemId>
                    <ItemName>Shipping</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>Test Testobject</ItemId>
                    <ItemName>Test Testobject</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>V2-0056</ItemId>
                    <ItemName>Thunder Frosted, 19.8g, Extra Strong Portion B</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
                <InventSum xmlns="">
                    <ItemId>V2-0103</ItemId>
                    <ItemName>Offroad Mel Oh!, 10g, Mini Portion</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
            </InventSumRequest>
        </getInventSumResponse>
    </soap:Body>
</soap:Envelope>

````

## getInventSumDetails

`getInventSumDetails` giver lidt flere oplysninger end `getInventSum` om lager beholdningen. 

For en given data får man følgende oplysninger: 

* `QtyAvailablePhysical`: 10 
* `QtySalesOrder`: -4
* `QtyPurchOrder`: 0
* `QtyAvailable`: 6

`QtyAvailable` er givet ved denne formel:

````
QtyAvailable = QtyAvailablePhysical + QtySalesOrder + QtyPurchOrder
````

### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <getInventSumDetails xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_calcDate>2019-05-07T15:25:38+02:00</_calcDate>
    </getInventSumDetails>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <getInventSumDetailsResponse xmlns="http://Ackro.dk/Services/2010">
            <getInventSumDetailsResult>
                <InventSumDetailsSum>
                    <InventSumDetails>
                        <ItemId>00000001</ItemId>
                        <ItemName>Standard vare</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>-9</QtyAvailablePhysical>
                        <QtySalesOrder>-10</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>-19</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>915</ItemId>
                        <ItemName>Test Testobject</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>-150</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>-150</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>916</ItemId>
                        <ItemName>Test US 1</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>-1</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>-1</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>917</ItemId>
                        <ItemName>Test Chew v 10</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>-10</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>-10</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>922</ItemId>
                        <ItemName>Test Chew</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>928</ItemId>
                        <ItemName>Test Chew Copy</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>Handling</ItemId>
                        <ItemName>Handling</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>Shipping</ItemId>
                        <ItemName>Shipping</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>Test Testobject</ItemId>
                        <ItemName>Test Testobject</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>V2-0056</ItemId>
                        <ItemName>Thunder Frosted, 19.8g, Extra Strong Portion B</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                    <InventSumDetails>
                        <ItemId>V2-0103</ItemId>
                        <ItemName>Offroad Mel Oh!, 10g, Mini Portion</ItemName>
                        <PhysicalDate>2019-05-07T00:00:00</PhysicalDate>
                        <QtyAvailablePhysical>0</QtyAvailablePhysical>
                        <QtySalesOrder>0</QtySalesOrder>
                        <QtyPurchOrder>0</QtyPurchOrder>
                        <QtyAvailable>0</QtyAvailable>
                        <CostPrice>0</CostPrice>
                    </InventSumDetails>
                </InventSumDetailsSum>
            </getInventSumDetailsResult>
        </getInventSumDetailsResponse>
    </soap:Body>
</soap:Envelope>
````

## setInventOperations
Denne `setInventOperations` operationen skal kaldes før `setSalesOperations` og `setSalesLineOperations`.
Det er klart at en vare skal findes før den kan sælges. 
Denne operation anvendes ligeledes hvis vares navn eller stregkode skal opdateres. 

### operationsType
* **insert <--** 
* update
* delete

Operationerne `update` og `delete` er også blevet testet. De giver tilsvarende responses. 

### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <setInventOperations xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_encryptionkey>sd%#gg9HwT2</_encryptionkey>
      <_shopId>Test</_shopId>
      <_itemId>LT001</_itemId>
      <_itemGroup>vrg01</_itemGroup>
      <_itemName>LT æøå ÆØÅ déjà aujourd'hui à côté</_itemName>
      <_itemBarCodeType></_itemBarCodeType>
      <_itemBarcode>7319009680131</_itemBarcode>
      <_operationsType>insert</_operationsType>
    </setInventOperations>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <setInventOperationsResponse xmlns="http://Ackro.dk/Services/2010">
            <setInventOperationsResult>
                <ItemOperations>
                    <ItemOperations>
                        <operationType>insert</operationType>
                        <ReturnMessage>OK</ReturnMessage>
                        <getResult>success</getResult>
                    </ItemOperations>
                </ItemOperations>
            </setInventOperationsResult>
        </setInventOperationsResponse>
    </soap:Body>
</soap:Envelope>
````

## setSalesOperations_V4

`setSalesOperations_V4` anvendes til at oprette Ordrehovedet.
Det er også muligt at opdatere ordrehovedet, men det er vigtigt at klarlægge under hvilke omstændigheder det er muligt. 


### operationsType
* **insert <--** 
* update
* delete

Operationerne `update` og `delete` er ikke blevet testet. 

### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <setSalesOperations_V4 xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_encryptionkey>sd%#gg9HwT2</_encryptionkey>
      <_operationsType>insert</_operationsType>
      <_shopId>Test</_shopId>
      <_supplierOrderNo>DA001</_supplierOrderNo>
      <_currency>dkk</_currency>
      <_languageId>da</_languageId>
      <_custAccount>LTCU001</_custAccount>
      <_eMail>info@me.com</_eMail>
      <_phone>+4512345678</_phone>
      <_phoneMobile>+4512345678</_phoneMobile>
      <_custName>Customer Name</_custName>
      <_custStreetLine1>Customer Street 1</_custStreetLine1>
      <_custStreetLine2>Customer Street 2</_custStreetLine2>
      <_custZipCode>2830</_custZipCode>
      <_custCity>Virum</_custCity>
      <_custCountry>Danmark</_custCountry>
      <_custContact>Mr. Søren Ære Østergård</_custContact>
      <_custVatNum>DK0123456789</_custVatNum>
      <_dlvName>Delivery Name</_dlvName>
      <_dlvStreetLine_1>Delivery Street 1</_dlvStreetLine_1>
      <_dlvStreetLine_2>Delivery Street 2</_dlvStreetLine_2>
      <_dlvZipCode>2830</_dlvZipCode>
      <_dlvCity>Virum</_dlvCity>
      <_dlvCountry>Denmark</_dlvCountry>
      <_dlvContact>H.C. Andersen</_dlvContact>
      <_dlvModeId></_dlvModeId>
      <_servicePointId></_servicePointId>
      <_receiptDateRequested>2019-05-13</_receiptDateRequested>
      <_notePick>Pluknotat</_notePick>
      <_noteDelivery>Leveringsnotat</_noteDelivery>
    </setSalesOperations_V4>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <setSalesOperations_V4Response xmlns="http://Ackro.dk/Services/2010">
            <setSalesOperations_V4Result>
                <SalesOperations>
                    <SalesOperations>
                        <operationType>insert</operationType>
                        <getResult>success</getResult>
                        <returnMessage>Ackro Salgsordre: SO00000225, oprettet</returnMessage>
                    </SalesOperations>
                </SalesOperations>
            </setSalesOperations_V4Result>
        </setSalesOperations_V4Response>
    </soap:Body>
</soap:Envelope>
````

## setSalesLineOperations
Da der ikke er noget linienummer, må man formode at `setSalesLineOperations` kun kan opretten én linie pr. varenummer. 
Det er også en precondition som vi skal være opmærksomme på.

Som for Ordrehovedet, skal vi også være opmærksomme på hvornår det er muligt at rette i en ordrelinje. 

### operationsType
* **insert <--** 
* update
* delete

Operationerne `update` og `delete` er ikke blevet testet. 

### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <setSalesLineOperations xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_encryptionkey>sd%#gg9HwT2</_encryptionkey>
      <_operationsType>insert</_operationsType>
      <_shopId>Test</_shopId>
      <_supplierOrderNo>DA001</_supplierOrderNo>
      <_itemId>LT001</_itemId>
      <_itemName></_itemName>
      <_salesQty>1</_salesQty>
      <_salesPrice>1234.56</_salesPrice>
    </setSalesLineOperations>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <setSalesLineOperationsResponse xmlns="http://Ackro.dk/Services/2010">
            <setSalesLineOperationsResult>
                <SalesOperations>
                    <SalesOperations>
                        <operationType>insert</operationType>
                        <getResult>success</getResult>
                        <returnMessage>Salgsordre: DA001, opdateret</returnMessage>
                    </SalesOperations>
                </SalesOperations>
            </setSalesLineOperationsResult>
        </setSalesLineOperationsResponse>
    </soap:Body>
</soap:Envelope>
````

## getPackingSlipInfoExtended_V3
Lagerhotellet kan ikke kalde tilbage og fortælle os at en ordre har skiftet status. 
Det skal vi selv sørge for at holde øje med. 

Det er derfor vigtig, at vi reglmæssigt forespørger på de åbne ordre, hvad deres status er. 

Ordrestatus kan være: Modtaget, Afsendt, eller ukendt. 

Hvis ordren er afsendt fra lagerhotellet, skal vi have opdateret Ordren i BC. 

`getPackingSlipInfoExtended_V3` skal kaldes for hver enkelt ordre og formodentlig mindst en gang om dagen.  


### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <getPackingSlipInfoExtended_V3 xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_encryptionkey>sd%#gg9HwT2</_encryptionkey>
      <_shopId>Test</_shopId>
      <_supplierOrderNo>DA001</_supplierOrderNo>
    </getPackingSlipInfoExtended_V3>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <getPackingSlipInfoExtended_V3Response xmlns="http://Ackro.dk/Services/2010">
            <getPackingSlipInfoExtended_V3Result>
                <PackingSlipInfo>
                    <PackingSlipInfo_V3>
                        <returnMessage>Ordre DA001 modtaget hos Ackro</returnMessage>
                        <getResult>success</getResult>
                        <getSalesOrderStatus>none</getSalesOrderStatus>
                        <getPackingSlipId />
                        <getPackingSlipDate>1900-01-01T00:00:00</getPackingSlipDate>
                        <getTrackNTraceId />
                        <getItemId>LT001</getItemId>
                        <getDlvQty>1</getDlvQty>
                        <getSerialNumber />
                        <getBatchNumber />
                        <getTrackNTraceList />
                    </PackingSlipInfo_V3>
                </PackingSlipInfo>
            </getPackingSlipInfoExtended_V3Result>
        </getPackingSlipInfoExtended_V3Response>
    </soap:Body>
</soap:Envelope>
````


## Kilder
Hermed oplysningerne til Ackro’s TEST regnskab.

WSDL: http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?wsdl

URL: http://mail.ackro.dk/AcKroInvent/InventItemService.asmx
 

Mht. eksempler på SOAP kaldet, fremgår de egentlig af de enkelte operationer på servicen, så det er nok nemmest, at du ser eksemplerne der:

http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?op=setInventOperations
http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?op=setSalesOperations_V4
http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?op=setSalesLineOperations
http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?op=getPackingSlipInfoExtended
http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?op=getInventSum
http://mail.ackro.dk/AcKroInvent/InventItemService.asmx?op=getInventSumDetails
 

Hvis man ikke ønsker at udfylde et felt, så inkluderes feltet som et tomt element, eks. <_itemBarcode/>. 
Ellers vil forespørgslen ikke overholde skemaet, og webserveren vil give en fejl.

 
 
# C/AL Code Patterns

## OBJECT Table 407 Graph Mail Setup

````
    LOCAL PROCEDURE SendWebRequest@5(Payload@1000 : Text;Token@1009 : Text) : Boolean;
    VAR
      TempBlob@1001 : TEMPORARY Record 99008535;
      HttpWebRequestMgt@1002 : Codeunit 1297;
      GraphMail@1006 : Codeunit 405;
      HttpStatusCode@1005 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode";
      ResponseHeaders@1004 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
      ResponseInStream@1003 : InStream;
    BEGIN
      TempBlob.INIT;
      TempBlob.Blob.CREATEINSTREAM(ResponseInStream);

      HttpWebRequestMgt.Initialize(STRSUBSTNO('%1/v1.0/me/sendMail',GraphMail.GetGraphDomain));
      HttpWebRequestMgt.SetMethod('POST');
      HttpWebRequestMgt.SetContentType('application/json');
      HttpWebRequestMgt.SetReturnType('application/json');
      HttpWebRequestMgt.AddHeader('Authorization',STRSUBSTNO('Bearer %1',Token));
      HttpWebRequestMgt.AddBodyAsText(Payload);

      IF NOT HttpWebRequestMgt.GetResponse(ResponseInStream,HttpStatusCode,ResponseHeaders) THEN BEGIN
        HttpWebRequestMgt.ProcessFaultResponse('');
        EXIT(FALSE);
      END;

      EXIT(TRUE);
    END;
````

## OBJECT Codeunit 1237 Get Json Structure

````
OBJECT Codeunit 1237 Get Json Structure
{
  OBJECT-PROPERTIES
  {
    Date=24-03-19;
    Time=12:00:00;
    Version List=NAVW114.00;
  }
  PROPERTIES
  {
    OnRun=BEGIN
          END;

  }
  CODE
  {
    VAR
      HttpWebRequestMgt@1004 : Codeunit 1297;
      JsonConvert@1000 : DotNet "'Newtonsoft.Json'.Newtonsoft.Json.JsonConvert";
      GLBHttpStatusCode@1003 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode";
      GLBResponseHeaders@1002 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
      FileContent@1001 : Text;
      InvalidResponseErr@1005 : TextConst 'DAN=Svaret var ugyldigt.;ENU=The response was not valid.';

    [Internal]
    PROCEDURE GenerateStructure@2(Path@1000 : Text;VAR XMLBuffer@1001 : Record 1235);
    VAR
      TempBlob@1010 : Record 99008535;
      ResponseTempBlob@1003 : Record 99008535;
      XMLBufferWriter@1002 : Codeunit 1235;
      JsonInStream@1007 : InStream;
      XMLOutStream@1009 : OutStream;
      File@1006 : File;
    BEGIN
      IF File.OPEN(Path) THEN
        File.CREATEINSTREAM(JsonInStream)
      ELSE BEGIN
        CLEAR(ResponseTempBlob);
        ResponseTempBlob.INIT;
        ResponseTempBlob.Blob.CREATEINSTREAM(JsonInStream);
        CLEAR(HttpWebRequestMgt);
        HttpWebRequestMgt.Initialize(Path);
        HttpWebRequestMgt.SetMethod('POST');
        HttpWebRequestMgt.SetReturnType('application/json');
        HttpWebRequestMgt.SetContentType('application/x-www-form-urlencoded');
        HttpWebRequestMgt.AddHeader('Accept-Encoding','utf-8');
        HttpWebRequestMgt.GetResponse(JsonInStream,GLBHttpStatusCode,GLBResponseHeaders);
      END;

      TempBlob.INIT;
      TempBlob.Blob.CREATEOUTSTREAM(XMLOutStream);
      IF NOT JsonToXML(JsonInStream,XMLOutStream) THEN
        IF NOT JsonToXMLCreateDefaultRoot(JsonInStream,XMLOutStream) THEN
          ERROR(InvalidResponseErr);

      XMLBufferWriter.GenerateStructure(XMLBuffer,XMLOutStream);
    END;

    [TryFunction]
    [External]
    PROCEDURE JsonToXML@1(JsonInStream@1000 : InStream;VAR XMLOutStream@1001 : OutStream);
    VAR
      XmlDocument@1003 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlDocument";
      NewContent@1002 : Text;
    BEGIN
      WHILE JsonInStream.READ(NewContent) > 0 DO
        FileContent += NewContent;

      XmlDocument := JsonConvert.DeserializeXmlNode(FileContent);
      XmlDocument.Save(XMLOutStream);
    END;

    [TryFunction]
    [External]
    PROCEDURE JsonToXMLCreateDefaultRoot@3(JsonInStream@1005 : InStream;VAR XMLOutStream@1000 : OutStream);
    VAR
      XmlDocument@1001 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlDocument";
      NewContent@1002 : Text;
    BEGIN
      WHILE JsonInStream.READ(NewContent) > 0 DO
        FileContent += NewContent;

      FileContent := '{"root":' + FileContent + '}';

      XmlDocument := JsonConvert.DeserializeXmlNode(FileContent,'root');
      XmlDocument.Save(XMLOutStream);
    END;

    BEGIN
    END.
  }
}
````


## OBJECT Codeunit 1281 Update Currency Exchange Rates

````
    LOCAL PROCEDURE ExecuteWebServiceRequest@1(CurrExchRateUpdateSetup@1001 : Record 1650;VAR ResponseInStream@1003 : InStream);
    VAR
      HttpStatusCode@1000 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode";
      ResponseHeaders@1004 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
      URL@1002 : Text;
    BEGIN
      CurrExchRateUpdateSetup.GetWebServiceURL(URL);
      HttpWebRequestMgt.Initialize(URL);
      HttpWebRequestMgt.SetReturnType('application/xml,text/xml');

      IF NOT GUIALLOWED THEN
        HttpWebRequestMgt.DisableUI;

      HttpWebRequestMgt.SetTraceLogEnabled(CurrExchRateUpdateSetup."Log Web Requests");

      IF NOT HttpWebRequestMgt.GetResponse(ResponseInStream,HttpStatusCode,ResponseHeaders) THEN
        ShowHttpError(CurrExchRateUpdateSetup,URL);
    END;
````

## OBJECT Codeunit 1290 SOAP Web Service Request Mgt.

````
OBJECT Codeunit 1290 SOAP Web Service Request Mgt.
{
  OBJECT-PROPERTIES
  {
    Date=24-03-19;
    Time=12:00:00;
    Version List=NAVW114.00;
  }
  PROPERTIES
  {
    OnRun=BEGIN
          END;

  }
  CODE
  {
    VAR
      BodyPathTxt@1001 : TextConst '@@@={Locked};DAN=/soap:Envelope/soap:Body;ENU=/soap:Envelope/soap:Body';
      ContentTypeTxt@1000 : TextConst '@@@={Locked};DAN="multipart/form-data; charset=utf-8";ENU="multipart/form-data; charset=utf-8"';
      FaultStringXmlPathTxt@1012 : TextConst '@@@={Locked};DAN=/soap:Envelope/soap:Body/soap:Fault/faultstring;ENU=/soap:Envelope/soap:Body/soap:Fault/faultstring';
      NoRequestBodyErr@1015 : TextConst 'DAN=Anmodningsindholdet er ikke angivet.;ENU=The request body is not set.';
      NoServiceAddressErr@1017 : TextConst 'DAN=Webtjeneste-URI''en er ikke angivet.;ENU=The web service URI is not set.';
      ExpectedResponseNotReceivedErr@1009 : TextConst 'DAN=De forventede data blev ikke modtaget fra webtjenesten.;ENU=The expected data was not received from the web service.';
      SchemaNamespaceTxt@1007 : TextConst '@@@={Locked};DAN=http://www.w3.org/2001/XMLSchema;ENU=http://www.w3.org/2001/XMLSchema';
      SchemaInstanceNamespaceTxt@1006 : TextConst '@@@={Locked};DAN=http://www.w3.org/2001/XMLSchema-instance;ENU=http://www.w3.org/2001/XMLSchema-instance';
      SecurityUtilityNamespaceTxt@1003 : TextConst '@@@={Locked};DAN=http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd;ENU=http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd';
      SecurityExtensionNamespaceTxt@1004 : TextConst '@@@={Locked};DAN=http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd;ENU=http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd';
      SoapNamespaceTxt@1002 : TextConst '@@@={Locked};DAN=http://schemas.xmlsoap.org/soap/envelope/;ENU=http://schemas.xmlsoap.org/soap/envelope/';
      UsernameTokenNamepsaceTxt@1005 : TextConst '@@@={Locked};DAN=http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordText;ENU=http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordText';
      TempDebugLogTempBlob@1010 : TEMPORARY Record 99008535;
      ResponseBodyTempBlob@1020 : Record 99008535;
      ResponseInStreamTempBlob@1019 : Record 99008535;
      Trace@1016 : Codeunit 1292;
      GlobalRequestBodyInStream@1022 : InStream;
      HttpWebResponse@1021 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpWebResponse";
      GlobalPassword@1013 : Text;
      GlobalURL@1014 : Text;
      GlobalUsername@1008 : Text;
      TraceLogEnabled@1011 : Boolean;
      GlobalTimeout@1024 : Integer;
      InternalErr@1028 : TextConst 'DAN=Fjerntjenesten har returneret f�lgende fejlmeddelelse:\\;ENU=The remote service has returned the following error message:\\';
      GlobalContentType@1026 : Text;
      GlobalSkipCheckHttps@1018 : Boolean;
      GlobalProgressDialogEnabled@1023 : Boolean;
      InvalidTokenFormatErr@1025 : TextConst 'DAN=Tokenet skal v�re i JWS- eller JWE-kompakt serialiseringsformat.;ENU=The token must be in JWS or JWE Compact Serialization Format.';

    [TryFunction]
    [Internal]
    PROCEDURE SendRequestToWebService@17();
    VAR
      WebRequestHelper@1000 : Codeunit 1299;
      HttpWebRequest@1007 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpWebRequest";
      HttpStatusCode@1002 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode";
      ResponseHeaders@1001 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
      ResponseInStream@1006 : InStream;
    BEGIN
      CheckGlobals;
      BuildWebRequest(GlobalURL,HttpWebRequest);
      ResponseInStreamTempBlob.INIT;
      ResponseInStreamTempBlob.Blob.CREATEINSTREAM(ResponseInStream);
      CreateSoapRequest(HttpWebRequest.GetRequestStream,GlobalRequestBodyInStream,GlobalUsername,GlobalPassword);
      WebRequestHelper.GetWebResponse(HttpWebRequest,HttpWebResponse,ResponseInStream,
        HttpStatusCode,ResponseHeaders,GlobalProgressDialogEnabled);
      ExtractContentFromResponse(ResponseInStream,ResponseBodyTempBlob);
    END;

    LOCAL PROCEDURE BuildWebRequest@3(ServiceUrl@1000 : Text;VAR HttpWebRequest@1002 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpWebRequest");
    VAR
      DecompressionMethods@1003 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.DecompressionMethods";
    BEGIN
      HttpWebRequest := HttpWebRequest.Create(ServiceUrl);
      HttpWebRequest.Method := 'POST';
      HttpWebRequest.KeepAlive := TRUE;
      HttpWebRequest.AllowAutoRedirect := TRUE;
      HttpWebRequest.UseDefaultCredentials := TRUE;
      IF GlobalContentType = '' THEN
        GlobalContentType := ContentTypeTxt;
      HttpWebRequest.ContentType := GlobalContentType;
      IF GlobalTimeout <= 0 THEN
        GlobalTimeout := 600000;
      HttpWebRequest.Timeout := GlobalTimeout;
      HttpWebRequest.AutomaticDecompression := DecompressionMethods.GZip;
    END;

    LOCAL PROCEDURE CreateSoapRequest@2(RequestOutStream@1000 : OutStream;BodyContentInStream@1004 : InStream;Username@1003 : Text;Password@1005 : Text);
    VAR
      XmlDoc@1007 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlDocument";
      BodyXmlNode@1016 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
    BEGIN
      CreateEnvelope(XmlDoc,BodyXmlNode,Username,Password);
      AddBodyToEnvelope(BodyXmlNode,BodyContentInStream);
      XmlDoc.Save(RequestOutStream);
      TraceLogXmlDocToTempFile(XmlDoc,'FullRequest');
    END;

    LOCAL PROCEDURE CreateEnvelope@11(VAR XmlDoc@1011 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlDocument";VAR BodyXmlNode@1001 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";Username@1009 : Text;Password@1010 : Text);
    VAR
      XMLDOMMgt@1000 : Codeunit 6224;
      EnvelopeXmlNode@1007 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      HeaderXmlNode@1006 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      SecurityXmlNode@1005 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      UsernameTokenXmlNode@1004 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      TempXmlNode@1003 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      PasswordXmlNode@1002 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
    BEGIN
      XmlDoc := XmlDoc.XmlDocument;
      WITH XMLDOMMgt DO BEGIN
        AddRootElementWithPrefix(XmlDoc,'Envelope','s',SoapNamespaceTxt,EnvelopeXmlNode);
        AddAttribute(EnvelopeXmlNode,'xmlns:u',SecurityUtilityNamespaceTxt);

        AddElementWithPrefix(EnvelopeXmlNode,'Header','','s',SoapNamespaceTxt,HeaderXmlNode);

        IF (Username <> '') OR (Password <> '') THEN BEGIN
          AddElementWithPrefix(HeaderXmlNode,'Security','','o',SecurityExtensionNamespaceTxt,SecurityXmlNode);
          AddAttributeWithPrefix(SecurityXmlNode,'mustUnderstand','s',SoapNamespaceTxt,'1');

          AddElementWithPrefix(SecurityXmlNode,'UsernameToken','','o',SecurityExtensionNamespaceTxt,UsernameTokenXmlNode);
          AddAttributeWithPrefix(UsernameTokenXmlNode,'Id','u',SecurityUtilityNamespaceTxt,CreateUUID);

          AddElementWithPrefix(UsernameTokenXmlNode,'Username',Username,'o',SecurityExtensionNamespaceTxt,TempXmlNode);
          AddElementWithPrefix(UsernameTokenXmlNode,'Password',Password,'o',SecurityExtensionNamespaceTxt,PasswordXmlNode);
          AddAttribute(PasswordXmlNode,'Type',UsernameTokenNamepsaceTxt);
        END;

        AddElementWithPrefix(EnvelopeXmlNode,'Body','','s',SoapNamespaceTxt,BodyXmlNode);
        AddAttribute(BodyXmlNode,'xmlns:xsi',SchemaInstanceNamespaceTxt);
        AddAttribute(BodyXmlNode,'xmlns:xsd',SchemaNamespaceTxt);
      END;
    END;

    LOCAL PROCEDURE CreateUUID@9() : Text;
    BEGIN
      EXIT('uuid-' + DELCHR(LOWERCASE(FORMAT(CREATEGUID)),'=','{}'));
    END;

    LOCAL PROCEDURE AddBodyToEnvelope@12(VAR BodyXmlNode@1005 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";BodyInStream@1000 : InStream);
    VAR
      XMLDOMManagement@1001 : Codeunit 6224;
      BodyContentXmlDoc@1003 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlDocument";
    BEGIN
      XMLDOMManagement.LoadXMLDocumentFromInStream(BodyInStream,BodyContentXmlDoc);
      TraceLogXmlDocToTempFile(BodyContentXmlDoc,'RequestBodyContent');

      BodyXmlNode.AppendChild(BodyXmlNode.OwnerDocument.ImportNode(BodyContentXmlDoc.DocumentElement,TRUE));
    END;

    LOCAL PROCEDURE ExtractContentFromResponse@4(ResponseInStream@1000 : InStream;VAR BodyTempBlob@1002 : Record 99008535);
    VAR
      XMLDOMMgt@1005 : Codeunit 6224;
      ResponseBodyXMLDoc@1004 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlDocument";
      ResponseBodyXmlNode@1006 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      XmlNode@1008 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      BodyOutStream@1007 : OutStream;
      Found@1001 : Boolean;
    BEGIN
      TraceLogStreamToTempFile(ResponseInStream,'FullResponse',TempDebugLogTempBlob);
      XMLDOMMgt.LoadXMLNodeFromInStream(ResponseInStream,XmlNode);

      Found := XMLDOMMgt.FindNodeWithNamespace(XmlNode,BodyPathTxt,'soap',SoapNamespaceTxt,ResponseBodyXmlNode);
      IF NOT Found THEN
        ERROR(ExpectedResponseNotReceivedErr);

      ResponseBodyXMLDoc := ResponseBodyXMLDoc.XmlDocument;
      ResponseBodyXMLDoc.AppendChild(ResponseBodyXMLDoc.ImportNode(ResponseBodyXmlNode.FirstChild,TRUE));

      BodyTempBlob.Blob.CREATEOUTSTREAM(BodyOutStream);
      ResponseBodyXMLDoc.Save(BodyOutStream);
      TraceLogXmlDocToTempFile(ResponseBodyXMLDoc,'ResponseBodyContent');
    END;

    PROCEDURE GetResponseContent@22(VAR ResponseBodyInStream@1000 : InStream);
    BEGIN
      ResponseBodyTempBlob.Blob.CREATEINSTREAM(ResponseBodyInStream);
    END;

    [Internal]
    PROCEDURE ProcessFaultResponse@15(SupportInfo@1001 : Text);
    VAR
      WebRequestHelper@1002 : Codeunit 1299;
      XMLDOMMgt@1006 : Codeunit 6224;
      WebException@1005 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.WebException";
      XmlNode@1004 : DotNet "'System.Xml, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Xml.XmlNode";
      ResponseInputStream@1000 : InStream;
      ErrorText@1009 : Text;
      ServiceURL@1010 : Text;
    BEGIN
      ErrorText := WebRequestHelper.GetWebResponseError(WebException,ServiceURL);

      IF ErrorText <> '' THEN
        ERROR(ErrorText);

      ResponseInputStream := WebException.Response.GetResponseStream;
      IF TraceLogEnabled THEN
        Trace.LogStreamToTempFile(ResponseInputStream,'WebExceptionResponse',TempDebugLogTempBlob);

      XMLDOMMgt.LoadXMLNodeFromInStream(ResponseInputStream,XmlNode);

      ErrorText := XMLDOMMgt.FindNodeTextWithNamespace(XmlNode,FaultStringXmlPathTxt,'soap',SoapNamespaceTxt);
      IF ErrorText = '' THEN
        ErrorText := WebException.Message;
      ErrorText := InternalErr + ErrorText + ServiceURL;

      IF SupportInfo <> '' THEN
        ErrorText += '\\' + SupportInfo;

      ERROR(ErrorText);
    END;

    [External]
    PROCEDURE SetGlobals@10(RequestBodyInStream@1000 : InStream;URL@1001 : Text;Username@1002 : Text;Password@1003 : Text);
    BEGIN
      GlobalRequestBodyInStream := RequestBodyInStream;

      GlobalSkipCheckHttps := FALSE;

      GlobalURL := URL;
      GlobalUsername := Username;
      GlobalPassword := Password;

      GlobalProgressDialogEnabled := TRUE;

      TraceLogEnabled := FALSE;
    END;

    [External]
    PROCEDURE SetTimeout@7(NewTimeout@1000 : Integer);
    BEGIN
      GlobalTimeout := NewTimeout;
    END;

````

## OBJECT Codeunit 1297 Http Web Request Mgt.
I guess this is my toolbox. 

## OBJECT Codeunit 1298 OAuth Management

## OBJECT Codeunit 1299 Web Request Helper
Used in Codeunit 1297.


## Some XML handling

````
      IF NOT HttpWebRequestMgt.TryLoadXMLResponse(GLBResponseInStream,XmlDoc) THEN BEGIN
        LogActivityFailed(DocRecordID,GetDocErrorTxt,'');
        EXIT(FALSE);
      END;

      Errors := XMLDOMMgt.FindNodeTextWithNamespace(XmlDoc.DocumentElement,GetErrorXPath,
          GetPrefix,GetApiNamespace);
````

## OBJECT Codeunit 1410 Doc. Exch. Service Mgt.

## OBJECT Codeunit 1432 Net Promoter Score Mgt.

````
    [TryFunction]
    [External]
    PROCEDURE ExecuteWebRequest@3(Url@1006 : Text;VAR Response@1004 : Text);
    VAR
      HttpWebRequestMgt@1002 : Codeunit 1297;
      HttpStatusCode@1001 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode";
      Headers@1000 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
      ErrorMessage@1007 : Text;
      ErrorDetails@1009 : Text;
    BEGIN
      HttpWebRequestMgt.Initialize(Url);
      HttpWebRequestMgt.DisableUI;
      HttpWebRequestMgt.SetReturnType('application/json');
      HttpWebRequestMgt.AddHeader('Accept-Encoding','utf-8');
      HttpWebRequestMgt.SetMethod('GET');
      HttpWebRequestMgt.SetTimeout(TimeoutInMilliseconds);
      IF NOT HttpWebRequestMgt.SendRequestAndReadTextResponse(Response,ErrorMessage,ErrorDetails,HttpStatusCode,Headers) THEN BEGIN
        IF ISNULL(HttpStatusCode) THEN BEGIN
          SENDTRACETAG('0000836',NpsCategoryTxt,VERBOSITY::Warning,RequestFailedErr,DATACLASSIFICATION::SystemMetadata);
          ERROR(ErrorMessage)
        END;

        IF (HttpStatusCode >= 400) AND (HttpStatusCode <= 499) THEN
          SENDTRACETAG('0000837',NpsCategoryTxt,
            VERBOSITY::Error,STRSUBSTNO(RequestFailedWithStatusCodeErr,HttpStatusCode),DATACLASSIFICATION::SystemMetadata)
        ELSE
          SENDTRACETAG('000022Q',NpsCategoryTxt,
            VERBOSITY::Warning,STRSUBSTNO(RequestFailedWithStatusCodeErr,HttpStatusCode),DATACLASSIFICATION::SystemMetadata);
        ERROR(ErrorMessage);
      END;
    END;
````

## OBJECT Codeunit 1545 Workflow Webhook Notification

````
    [TryFunction]
    LOCAL PROCEDURE PostHttpRequest@16(DataID@1002 : GUID;WorkflowStepInstanceID@1001 : GUID;NotificationUrl@1000 : Text;RequestedByUserEmail@1009 : Text);
    VAR
      TypeHelper@1003 : Codeunit 10;
      HttpWebRequest@1007 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpWebRequest";
      HttpWebResponse@1006 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpWebResponse";
      RequestStr@1005 : DotNet "'mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.IO.Stream";
      StreamWriter@1004 : DotNet "'mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.IO.StreamWriter";
      Encoding@1008 : DotNet "'mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Text.Encoding";
    BEGIN
      HttpWebRequest := HttpWebRequest.Create(NotificationUrl);
      HttpWebRequest.Method := 'POST';
      HttpWebRequest.ContentType('application/json');

      RequestStr := HttpWebRequest.GetRequestStream;
      StreamWriter := StreamWriter.StreamWriter(RequestStr,Encoding.ASCII);
      StreamWriter.Write('{"Row Id":"' + TypeHelper.GetGuidAsString(DataID) +
        '","Workflow Step Id":"' + TypeHelper.GetGuidAsString(WorkflowStepInstanceID) +
        '","Requested By User Email":"' + RequestedByUserEmail + '"}');
      StreamWriter.Flush;
      StreamWriter.Close;
      StreamWriter.Dispose;

      HttpWebResponse := HttpWebRequest.GetResponse;
      HttpWebResponse.Close; // close connection
      HttpWebResponse.Dispose; // cleanup of IDisposable
    END;
````

## OBJECT Codeunit 6154 API Webhook Notification Send

````
    [TryFunction]
    LOCAL PROCEDURE SendRequest@14(NotificationUrlNumber@1007 : Integer;NotificationUrl@1015 : Text;NotificationPayload@1004 : Text;VAR ResponseBody@1003 : Text;VAR ErrorMessage@1001 : Text;VAR ErrorDetails@1005 : Text;VAR HttpStatusCode@1002 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode");
    VAR
      HttpWebRequestMgt@1000 : Codeunit 1297;
      ResponseHeaders@1006 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
    BEGIN
      IF NotificationUrl = '' THEN BEGIN
        SENDTRACETAG('00002A1',APIWebhookCategoryLbl,VERBOSITY::Warning,
          STRSUBSTNO(EmptyNotificationUrlErr,NotificationUrlNumber),DATACLASSIFICATION::SystemMetadata);
        ERROR(STRSUBSTNO(EmptyNotificationUrlErr,NotificationUrlNumber));
      END;

      IF NotificationPayload = '' THEN BEGIN
        SENDTRACETAG('00002A2',APIWebhookCategoryLbl,VERBOSITY::Warning,
          STRSUBSTNO(EmptyPayloadPerNotificationUrlErr,NotificationUrlNumber),DATACLASSIFICATION::SystemMetadata);
        ERROR(STRSUBSTNO(EmptyPayloadPerNotificationUrlErr,NotificationUrlNumber));
      END;

      HttpWebRequestMgt.Initialize(NotificationUrl);
      HttpWebRequestMgt.DisableUI;
      HttpWebRequestMgt.SetMethod('POST');
      HttpWebRequestMgt.SetReturnType('application/json');
      HttpWebRequestMgt.SetContentType('application/json');
      HttpWebRequestMgt.SetTimeout(GetSendingNotificationTimeout);
      HttpWebRequestMgt.AddBodyAsText(NotificationPayload);

      IF NOT HttpWebRequestMgt.SendRequestAndReadTextResponse(ResponseBody,ErrorMessage,ErrorDetails,HttpStatusCode,ResponseHeaders) THEN BEGIN
        IF ISNULL(HttpStatusCode) THEN
          SENDTRACETAG('00002A3',APIWebhookCategoryLbl,VERBOSITY::Warning,
            STRSUBSTNO(CannotGetResponseErr,NotificationUrlNumber),DATACLASSIFICATION::SystemMetadata);
        ERROR(STRSUBSTNO(CannotGetResponseErr,NotificationUrlNumber));
      END;
    END;
````

## OBJECT Codeunit 9033 Invite External Accountant

````
    LOCAL PROCEDURE InvokeRequest@24(Url@1007 : Text;Verb@1008 : Text;Body@1010 : Text;AuthResourceUrl@1031 : Text;VAR ResponseContent@1009 : Text) : Boolean;
    VAR
      TempBlob@1021 : Record 99008535;
      AzureADMgt@1006 : Codeunit 6300;
      IdentityManagement@1020 : Codeunit 9801;
      HttpWebRequestMgt@1022 : Codeunit 1297;
      WebRequestHelper@1023 : Codeunit 1299;
      HttpStatusCode@1024 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.HttpStatusCode";
      ResponseHeaders@1025 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Collections.Specialized.NameValueCollection";
      WebException@1026 : DotNet "'System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'.System.Net.WebException";
      InStr@1027 : InStream;
      AccessToken@1005 : Text;
      ServiceUrl@1029 : Text;
      ChunkText@1028 : Text;
      WasSuccessful@1030 : Boolean;
    BEGIN
      AccessToken := AzureADMgt.GetGuestAccessToken(AuthResourceUrl,IdentityManagement.GetAadTenantId);

      IF AccessToken = '' THEN
        ERROR(ErrorAcquiringTokenErr);

      HttpWebRequestMgt.Initialize(Url);
      HttpWebRequestMgt.DisableUI;
      HttpWebRequestMgt.SetReturnType('application/json');
      HttpWebRequestMgt.SetContentType('application/json');
      HttpWebRequestMgt.SetMethod(Verb);
      HttpWebRequestMgt.AddHeader('Authorization','Bearer ' + AccessToken);
      IF Verb <> 'GET' THEN
        HttpWebRequestMgt.AddBodyAsText(Body);

      TempBlob.INIT;
      TempBlob.Blob.CREATEINSTREAM(InStr);
      IF HttpWebRequestMgt.GetResponse(InStr,HttpStatusCode,ResponseHeaders) THEN
        WasSuccessful := TRUE
      ELSE BEGIN
        WebRequestHelper.GetWebResponseError(WebException,ServiceUrl);
        WebException.Response.GetResponseStream.CopyTo(InStr);
        WasSuccessful := FALSE;
      END;

      WHILE NOT InStr.EOS DO BEGIN
        InStr.READTEXT(ChunkText);
        ResponseContent += ChunkText;
      END;

      EXIT(WasSuccessful);
    END;
````

