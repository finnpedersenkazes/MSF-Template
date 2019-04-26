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
