# Vision / Scope

LibraTone integration til Ackro's Webservice

## Vision

LibraTone opgraderer fra NAV 2015 til Business Central on Premise. 

I den forbindelse skifter de lagerhotel til Ackro. http://ackro.dk/ 

Der skal bygges en en løsning til at kommunikere med lagerhotellet. 

Requirements and Constraints
- Integrationen skal bygges således at der er mindst muligt impact på standard Business Central versionen. 
- Robustness. Løsningen skal så vidt muligt kunne køre uden opsyn, men tilbyde mulighed for support. 

Beskrivelse af løsningen findes i dokumentet: Functional Specification.

Design af løsningen findes i dokumentet: Technical Design. 

## Bruger Scenario
En ordre oprettes i Business Central til en kunde. 
Når ordren er godkendt, sendres en besked til lagerhotellet om levering. 

## Scope
Webservicen tilbyder en række andre operationer some **skal** implementeres som en del af dette projekt. 

- Oprettelse af varer
  - setInventOperations
  - operationsType
    - Insert
    - Update
    - Delete
- Oprettelse af ordrehoved
  - setSalesOperations_V4
  - operationsType
    - Insert
    - Update
    - Delete
- Oprettelse af ordrelinje
  - setSalesLineOperations
  - operationsType
    - Insert
    - Update
    - Delete
- Oplyser én beholdning pr. vare, som er den fysiske beholdning minus den åbne ordrebeholdning
  - getInventSum 
- Udvidet version af ’getInventSum’ operationen, har lidt flere informationer.
  - getInventSumDetail
- Forespørger på ordrestatus. Når der er lavet følgeseddel på ordre, vil operationen bla. returnere T&T samt afsendte serienumre.
  - getPackingSlipInfoExtended

### Out of scope
Webservicen tilbyder en række andre operationer some **ikke skal** implementeres som en del af dette projekt. 

- getInventItemTrans
- getInventSumBatch
- getInventTurnOverRate
- getPackingSlipInfo
- setSalesOperations (v1, v2, v3)

### Webservice beskrivelse
Udviklingen tager udgangspunkt i beskrivelse af Ackro's webservice version 1.5 dateret 5-7-2018. 

## Success Criteria

- *What does the goal look like? So we know when we have arrived at the finish line.* 
- *How to prove that we have succeded in achieving the goal?*
