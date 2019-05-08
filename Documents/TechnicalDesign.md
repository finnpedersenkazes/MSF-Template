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
Det kan forekomme at opdatere en vare eller en salgsordre flere gange inden Event Handlere kommer til at behandle hændelse. 
Det kan se hvis Event Handleren har været stoppet i en periode. 

Det er vigtigt at ved behandlingen af en event at det sikres at man behandler den seneste og at alle andre tidligere 
events vedrørende den samme vare eller ordre, får en status `Udløbet` der gør at de bliver ignorert fremover. 


### Preconditions
Det er vigtigt først og fremmest at forstå hvorfor et request kan fejle og fra starten forsøge at undgå disse situationer
før requests sendes. Det vil sige at Event Handleren skal kunne afvise en event, hvis denne ikke indeholder tilstrækkelige 
oplysninger for at requestet kan sendes med forventet succes. Dette kaldes for **preconditions**. 

EventLoggen's status felt skal altså have en option `Afvist`.

Et eksempel på en precondition er at man ikke skal sælge en vare, der ikke først er oprettet i lagerhotellet. 

## Respons Handler
Respons-handleren behandler det svar der kommer tilbage fra lagerhotellet. 

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
Hvis requestet eller har den rette syntax, burde forklaringen komme tilbage i feltet `returnMessage`. 

Skulle lagerhotellets interface ikke svare tilbage, er det tilsvarende op til Respons Handlere at sikre at håndtere denne situation. 

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
*What behaviour has to be configurable?* 
*What options have to be moved to a setup table?*

Opsætningsoplysninger kunne typisk være

* Frekvens for kørsel af Event Handler
* Emails på personer der skal informeres hvis noget går galt
* Sti til lagerhotellets webservice
* ...


## Fremtidige opgraderinger
APIer, alså interfaces, til systemer som Ackro's webservice udvikler sig over tid. 

Men det gøres altid på en bestemt måde. 

Hvis udbyderen af webservicen ønsker at ændre sit interface, for eksempel ved at tilføje et nyt felt til en eksisterende 
operation, gøres dette ved at tilbyde en ny opration. I Ackro's tilfælde kan man se at funktionen til at oprette
ordrehovder har ændres sig over tid. Den seneste operation hedder nu `setSalesOperations_V4`. 
Det er altså den fjerde version. 

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

## Test Setup
* _configuration: Ack#2009
* _company: test
* _encryptionkey: sd%#gg9HwT2
* _shopId: Test

## getInventSum

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
### Hvordan blev varen oprettet?
Tilsyneladene er der umiddelbart et problem med special karakterer. 

Vi må se på hvordan varen faktisk blev oprettet. For at se om det bare er eksporten af varenavnet der har et problem eller om det er ved oprettelsen. 


````
                <InventSum xmlns="">
                    <ItemId>LT001</ItemId>
                    <ItemName>LT ?????? ?????? d??j?? aujourd'hui ?? c??t??</ItemName>
                    <Date>2019-05-07</Date>
                    <AvailablePhysicalQty>0</AvailablePhysicalQty>
                    <CostPrice>0</CostPrice>
                </InventSum>
````

## setSalesOperations_V4

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

## getPackingSlipInfoExtended


### Request Body
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <getPackingSlipInfoExtended xmlns="http://Ackro.dk/Services/2010">
      <_configuration>Ack#2009</_configuration>
      <_company>test</_company>
      <_encryptionkey>sd%#gg9HwT2</_encryptionkey>
      <_shopId>Test</_shopId>
      <_supplierOrderNo>DA001</_supplierOrderNo>
    </getPackingSlipInfoExtended>
  </soap:Body>
</soap:Envelope>
````

### Response
````
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
    <soap:Body>
        <getPackingSlipInfoExtendedResponse xmlns="http://Ackro.dk/Services/2010">
            <getPackingSlipInfoExtendedResult>
                <PackingSlipInfo>
                    <PackingSlipInfo>
                        <returnMessage>Ordre DA001 modtaget hos Ackro</returnMessage>
                        <getResult>success</getResult>
                        <getSalesOrderStatus>none</getSalesOrderStatus>
                        <getPackingSlipId />
                        <getPackingSlipDate>1900-01-01T00:00:00</getPackingSlipDate>
                        <getTrackNTraceId />
                        <getItemId>LT001</getItemId>
                        <getDlvQty>1</getDlvQty>
                        <getSerialNumber />
                    </PackingSlipInfo>
                </PackingSlipInfo>
            </getPackingSlipInfoExtendedResult>
        </getPackingSlipInfoExtendedResponse>
    </soap:Body>
</soap:Envelope>
````
