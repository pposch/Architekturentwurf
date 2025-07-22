```mermaid
graph TD
  subgraph Gateway
    APIGW[API Gateway]
  end

  subgraph Frontend
    UI[Hotel Web/App Frontend]
  end

  UI --> APIGW

  subgraph Auth
    AUTH[Auth Service]
  end

  subgraph Microservices
    CUST[Customer Service]
    BOOK[Booking Service]
    ROOM[Room Service]
    PAY[Payment Service]
    INV[Invoice Service]
    NOTIF[Notification Service]
  end

  APIGW --> CUST
  APIGW --> BOOK
  APIGW --> ROOM
  APIGW --> PAY
  APIGW --> INV
  APIGW --> AUTH

  BOOK -->|BookingCreated Event| PAY
  PAY -->|PaymentCompleted Event| INV
  INV -->|InvoiceReady Event| NOTIF
```
1. Customer Service <br>
- Zuständigkeit: <br>
> Kundenprofile, Kontaktdaten, Registrierung, ggf. Login-Verbindung mit Auth. <br>
- Begründung: <br>
> Kundenverwaltung ist ein klar abgegrenzter Fachbereich. <br>
> Wird unabhängig von Buchung oder Zahlung genutzt (z. B. Kundenprofil bearbeiten). <br>
> Ermöglicht Datenschutzkonforme Verarbeitung (z. B. DSGVO-Löschung). <br>

2. Booking Service <br>
- Zuständigkeit: <br>
> Buchung von Zimmern, Reservierung, Check-in/out, Stornierungen. <br>
- Begründung: <br>
> Kerndomäne des Systems (zentrale Geschäftslogik). <br>
> Starke fachliche Trennung von Kunden- und Zahlungsdaten. <br>
> Änderungen (z. B. Umbuchung, Regeln) sollen unabhängig testbar sein. <br>

3. Room Service <br>
- Zuständigkeit: <br>
> Verwaltung von Zimmern, Kategorien, Ausstattung, Verfügbarkeiten. <br>
- Begründung: <br>
> Zimmerdaten sind referenziert, aber unabhängig von Buchungen (z. B. Zimmer-Upgrades). <br>
> Ermöglicht separate Pflege durch Hotel-Backoffice ohne Logik in Booking. <br>
> Kapselt Zimmerverwaltung für Re-use (z. B. in Hotelverwaltungssystemen). <br>

4. Payment Service <br>
- Zuständigkeit: <br>
> Zahlungsabwicklung, Rückerstattung, Zahlungsmethoden (z. B. Kreditkarte, PayPal). <br>
- Begründung: <br>
> Sicherheitstechnisch und regulatorisch (PCI-DSS) kritisch – gehört isoliert. <br>
> Oft mit externen Providern gekoppelt (Stripe, Adyen). <br>
> Muss besonders fehlerresilient und nachvollziehbar (Auditing) sein. <br>

5. Invoice Service <br>
- Zuständigkeit: <br>
> Rechnungserzeugung, steuerliche Berechnung, Rechnungsversand. <br>
- Begründung: <br>
> Unterschiedliche steuerliche Logik (z. B. Länderabhängigkeit). <br>
> PDF-Generierung, Versand und Archivierung sind eigene Verantwortlichkeiten. <br>
> Kann Trigger von Zahlung oder Buchung benötigen – spricht für Event-Handling. <br>

6. Notification Service <br>
- Zuständigkeit: <br>
> E-Mail- und SMS-Versand für Buchungsbestätigungen, Rechnungen etc. <br>
- Begründung: <br>
> Reine Infrastrukturkomponente. <br>
> Ermöglicht lose Kopplung zu anderen Services durch asynchrone Events. <br>
> Kann einfach skalieren und durch externe Provider ersetzt werden (z. B. SendGrid). <br>

8. Auth Service <br>
- Zuständigkeit: <br>
> Authentifizierung (z. B. OAuth2), Token-Ausgabe, Login-Prozesse. <br>
- Begründung: <br>
> Trennung von fachlichen Daten (Customer) und sicherheitsrelevanter Authentifizierung. <br>
> Wiederverwendbar für interne Tools, Admin-Oberfläche etc. <br>
> Bei Bedarf an externe Identity Provider auslagerbar. <br>
- Anmerkungen:  <br>
> Den Auth Service habe ich ausßerhalb des Microservices Block dargestellt da dieser Service eine besondere Rolle als Infrastruktur- und Security-Komponente darstellt und meist an externe Systeme wie Aurth0 oder Azure AD delegiert wird.

<h2>Kommunikation der Services</h2>

```mermaid
graph TD
  APIGW[API Gateway]
  APIGW -->|REST| BOOK[Booking Service]
  APIGW -->|REST| CUST[Customer Service]
  APIGW -->|REST| ROOM[Room Service]
  APIGW -->|REST| AUTH[Auth Service]

  BOOK --> BookingCreated -->|Async Event| PAY[Payment Service]
  PAY --> PaymentCompleted -->|Async Event| INV[Invoice Service]
  INV --> InvoiceReady -->|Async Event| NOTIF[Notification Service]

  BOOK -->|REST| ROOM
  BOOK -->|REST| CUST
```

1. API Gateway → Microservices <br>
- REST (HTTP): <br>
> Externe Clients (Web/Apps) erwarten schnelle, synchrone Antworten (z. B. Buchung abschicken, Rechnungs-PDF anzeigen). <br>
> Einfach, transparent, geeignet für lesende Operationen. <br>

2. Booking Service ↔ Room Service <br>
- REST (synchron): <br>
> Bei einer Buchung muss die aktuelle Verfügbarkeit eines Zimmers geprüft werden (z. B. GET /rooms/{id}/availability). <br>
> Enge zeitliche Kopplung erforderlich – synchroner Zugriff sinnvoll. <br>
> Caching auf Booking-Seite möglich, um REST-Aufrufe zu minimieren. <br>

3. Booking Service → Payment Service <br>
- Eventing (asynchron, über Message Broker): <br>
> z. B. Event BookingCreated <br>
> Buchung erzeugt Event, Zahlung erfolgt asynchron (User muss evtl. noch bezahlen). <br>
> Loose Coupling, keine Wartezeit nötig. <br>
> Technologien: Azure Service Bus, RabbitMQ, Apache Kafka <br>

4. Payment Service → Invoice Service <br>
- Eventing (asynchron): <br>
> z. B. Event PaymentCompleted <br>
> Die Rechnung wird nach erfolgreicher Zahlung erzeugt. <br>
> Lose Kopplung, leicht zu erweitern (z. B. weitere Services wie Loyalty Points hinzufügen). <br>
> Resilient gegen temporäre Ausfälle (z. B. Invoice-Queue puffert Event). <br>

5. Invoice Service → Notification Service <br>
- Messaging (asynchron): <br>
> Rechnung wurde erzeugt → E-Mail mit Anhang versenden. <br>
> Kein Rückgabewert nötig, Entkopplung erhöht Resilienz. <br>

6. Customer Service ↔ Booking/Payment <br>
- REST oder Eventing: <br>
> REST: GET /customers/{id} → Daten abrufen <br>
> Eventing: CustomerDeleted → Storniere alle offenen Buchungen <br>
> CRUD-Anfragen synchron, systemische Änderungen (z. B. Löschung) über Eventing <br>

7. Auth Service <br>
- REST (synchron) – nur mit API Gateway: <br>
> Authentifizierung muss direkt und zuverlässig beim Login erfolgen. <br>
> JWT-Token-Verifikation erfolgt im API Gateway, Services verifizieren ggf. Signatur lokal (kein erneuter Aufruf nötig → Performance). <br>

<h2> Beispiel </h2>

User → API Gateway → POST /booking

1. Booking Service:
> prüft Zimmerverfügbarkeit via REST → Room Service  
> speichert Buchung lokal  
> sendet Event `BookingCreated`

2. Payment Service:
> empfängt Event `BookingCreated`  
> startet Bezahlprozess  
> bei Erfolg: sendet Event `PaymentCompleted`

3. Invoice Service:
> empfängt Event `PaymentCompleted`  
> erzeugt Rechnung  
> sendet Event `InvoiceReady`

4. Notification Service:
> empfängt Event `InvoiceReady`  
> versendet E-Mail mit PDF
