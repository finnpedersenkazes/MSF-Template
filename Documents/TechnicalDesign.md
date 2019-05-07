# Technical Design

*Describe how you intend to implement the solution.*

## Beskrivelse af Data flow

At oprette en ny vare eller opretten en ny salgsordre er *ikke* en hændelse der trigger en hændelse mod lagerhotellet. 

Brugeren skal specifikt indikere at varen er klar til at blive oprettet i lagerhotellet. Ligeledes skal brugeren indikere at ordren er klar til levering før en besked bliver sendt til lagerhotellet. 

## Event Log
Når man er klar, vare eller salgsordre, oprettes en Event i Eventloggen med status klar (Ready).

## Event Handler
En event-handler kan køre automatisk eller fyres af manuelt. 
Event-handleren leder efter events som er klar til at blive behandlet. 

Event-handleren sender et HTTP POST request til lagerhotellet og modtager et svar. 

*Formatet af hvert af de seks reqeusts skal beskrives i detaljer.*

## Respons Handler
Respons-handleren behandler det svar der kommer tilbage fra lagerhotellet. 

*Det skal beskrives i detaljer, hvad der skal ske for hvert af de seks requests og for hvert af de mulige reponses*

## Configuration and Setup
*What behaviour has to be configurable?* 

*What options have to me move to a setup table?*

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

