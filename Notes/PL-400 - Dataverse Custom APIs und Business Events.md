---
categories:
  - "[[PL-400]]"
tags:
  - Dataverse
  - Custom APIs
created: 2026-11-16
---
# Dataverse Custom APIs und Business Events

Eine **Custom API** definiert eine eigene Dataverse-Operation, die explizit von anderen Komponenten aufgerufen werden kann. Sie kann eine Plug-in-Implementierung für zentrale Business-Logik besitzen, muss es aber nicht.

## Custom API mit Plug-in

Eine Custom API eignet sich, wenn mehrere Aufrufer dieselbe Business-Logik verwenden sollen, zum Beispiel zum Validieren und Genehmigen eines Accounts. Die API kann Eingaben entgegennehmen und ein Plug-in ausführen, das Dataverse-Daten aktualisiert oder weitere Aktionen ausführt.

```csharp
var request = new OrganizationRequest("new_ApproveAccount")
{
    ["AccountId"] = accountId
};

service.Execute(request);
```

Das Plug-in wird als Implementierung der Custom API registriert. Dadurch rufen Model-Driven Apps, Power Automate oder externe Anwendungen dieselbe Operation auf, statt die Logik jeweils selbst nachzubilden.

## Custom API als Business Event

Eine Custom API kann auch ein Ereignis bereitstellen, auf das andere Komponenten reagieren, etwa `new_AccountApproved`. In diesem Fall ist keine eigene Plug-in-Implementierung zwingend erforderlich. Ein Aufrufer löst das Ereignis aus; beispielsweise kann ein Flow daraufhin E-Mails versenden oder ein externes System informieren.

Damit das Ereignis im Dataverse-Trigger **When an action is performed** auswählbar ist, werden Catalog und Category verwendet. Die Category gruppiert das Business Event; eine `CatalogAssignment`-Zuordnung verknüpft die Custom API mit der Category.

Der Flow wird mit Catalog, Category, Tabelle und Action konfiguriert. Für einen Tabellenbezug muss die Custom API als Entity-bound Action definiert sein.

```csharp
var request = new OrganizationRequest("new_AccountApproved")
{
    ["Target"] = new EntityReference("account", accountId)
};

service.Execute(request);
```

## Abgrenzung zum Plug-in

- Ein normales [[Dataverse Plug-ins|Plug-in]] reagiert auf ein Dataverse-Pipeline-Ereignis, zum Beispiel ein Update.
- Eine Custom API definiert eine eigene Operation, die ein Aufrufer gezielt startet.
- Wird die Custom API mit einem Plug-in implementiert, dient sie als klarer, wiederverwendbarer Einstiegspunkt für die Business-Logik.
- Als Business Event macht eine Custom API eine Operation für weitere Komponenten wie [[Power Automate]] verfügbar.

## Merksätze

- **Custom API:** eigene, explizit aufrufbare Dataverse-Operation.
- **Custom API mit Plug-in:** zentrale Business-Logik für mehrere Aufrufer.
- **Custom API als Business Event:** Ereignis, auf das andere Komponenten reagieren können.
- **Catalog und Category:** machen katalogisierte Business Events im Trigger **When an action is performed** verfügbar.
