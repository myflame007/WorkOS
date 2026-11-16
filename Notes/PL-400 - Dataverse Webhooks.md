---
categories:
  - "[[PL-400]]"
tags:
  - Dataverse
  - Integrations
created: 2026-11-16
---
# Dataverse Webhooks

Ein **Webhook** sendet bei einem Dataverse-Pipeline-Ereignis einen HTTP-Request an einen externen Endpoint. So kann Dataverse beispielsweise einen externen Service über eine erstellte oder geänderte Zeile informieren, ohne dass dafür zwingend ein eigenes C#-Plug-in die Integration implementieren muss.

## Registrierung

Ein Webhook lässt sich mit dem Plug-in Registration Tool registrieren. Dazu werden Endpoint und Authentifizierung konfiguriert. Anschließend wird ein Step mit Message, Tabelle, Filtering Attributes, Stage und Ausführungsmodus registriert.

Beispiel für eine asynchrone Benachrichtigung bei einer Änderung an einem Consent-Feld:

- **Message:** `Update`
- **Primary Entity:** `contact`
- **Filtering Attributes:** `new_newsletterconsent`
- **Stage:** `PostOperation`
- **Execution Mode:** `Asynchronous`

Filtering Attributes prüfen, ob ein Attribut im Update-Request enthalten ist. Sie prüfen **nicht automatisch**, ob sich der Wert tatsächlich geändert hat.

## Request-Kontext und Images

Der Request enthält Informationen aus dem Plug-in Execution Context, beispielsweise Message-Name, Tabellenname und Zeilen-ID sowie Input Parameters und registrierte Pre- oder Post-Images. Images müssen passend registriert werden; ein Post Image kann etwa E-Mail-Adresse und Consent-Wert enthalten.

Falls die benötigten Felder nicht im Kontext enthalten sind, kann der externe Service weitere Daten über die Dataverse API abrufen.

## Webhook, Plug-in und Service Bus

- Ein **Plug-in** führt Business-Logik innerhalb der Dataverse-Pipeline aus.
- Ein **Webhook** sendet eine HTTP-Benachrichtigung an einen externen Service. Dieser übernimmt die weitere Verarbeitung.
- **Asynchrone Webhooks** laufen über den Dataverse Async Service, bieten aber keine eigene dauerhafte Queue-Verwaltung.
- [[Azure Service Bus]] entkoppelt Sender und Empfänger. Nachrichten können zwischengespeichert und später verarbeitet werden; das eignet sich für robuste Integrationen.

## Merksätze

- **Webhook:** HTTP-Benachrichtigung bei einem Dataverse-Ereignis.
- **PostOperation:** Die Dataverse-Operation wurde in der Pipeline bereits ausgeführt.
- **Asynchronous:** Der externe Aufruf blockiert nicht die ursprüngliche Benutzeroperation.
- **Service Bus:** Queue-basierte, entkoppelte Nachrichtenverarbeitung statt direktem HTTP-Aufruf.
