---
categories:
  - "[[PL-400]]"
tags:
  - Azure Functions
  - Workflows
created: 2026-11-16
---
# Azure Durable Functions

**Durable Functions** erweitern Azure Functions um zustandsbehaftete, langlebige Workflows. Der Orchestrierungsfortschritt wird persistent verwaltet. Ein Workflow kann deshalb auf ein Ereignis oder einen Zeitpunkt warten, ohne dass währenddessen ein Function-Prozess aktiv bleiben muss.

## Zentrale Komponenten

- **`DurableTaskClient`:** startet und verwaltet Orchestrierungen.
- **`TaskOrchestrationContext`:** steuert die Orchestrierung, ruft Activities auf und wartet auf Timer oder externe Ereignisse.
- **Activity Functions:** führen die eigentliche Arbeit aus, etwa Dataverse aktualisieren oder einen HTTP-Aufruf ausführen.

## Timer und externe Ereignisse

Ein Durable Timer eignet sich für längere Wartezeiten:

```csharp
await context.CreateTimerAsync(
    context.CurrentDateTimeUtc.AddDays(3),
    CancellationToken.None
);
```

`CreateTimerAsync()` wird von Durable Functions persistent verwaltet und übersteht Host-Neustarts. `Task.Delay()` ist dagegen kein persistenter Orchestrierungs-Timer.

Eine Orchestrierung kann außerdem auf ein externes Ereignis warten, zum Beispiel auf eine Genehmigung:

```csharp
var approved = await context.WaitForExternalEvent<bool>(
    "ApprovalReceived"
);
```

Eine andere Function kann dieses Ereignis an die laufende Orchestrierung senden. So lassen sich Genehmigungen, Benutzerbestätigungen oder externe Systemantworten abbilden.

## Orchestrator und Activity

Der **Orchestrator** koordiniert den Ablauf; **Activities** führen Seiteneffekte und eigentliche Arbeit aus. Orchestratoren müssen deterministisch sein, da Durable Functions sie beim Replay erneut ausführen, um den gespeicherten Zustand zu rekonstruieren.

Deshalb gehören HTTP-Aufrufe und Datenbankzugriffe in Activities, nicht direkt in den Orchestrator. Activities können bei Wiederholungen mehrfach ausgeführt werden; ihre Seiteneffekte sollten daher idempotent gestaltet werden.

## Dataverse-Integration und Einsatz

Ein Dataverse-Plug-in hostet keine Durable Function, kann aber eine Orchestrierung anstoßen, zum Beispiel über einen Aufruf an eine Azure Function. Für eine stärker entkoppelte Integration kann [[Azure Service Bus]] verwendet werden.

Typische Muster sind:

- **Function Chaining:** Activities nacheinander ausführen.
- **Fan-out/Fan-in:** Aufgaben parallel ausführen und Ergebnisse zusammenführen.
- **Human Interaction:** auf eine Genehmigung warten, optional mit Timeout und Eskalation.

[[Power Automate]] bietet einen visuellen Low-Code-Ansatz mit vielen Connectors und fertigen Approval Actions. Durable Functions sind codebasiert und bieten mehr Kontrolle über Ablauf, Timer, parallele Verarbeitung und Fehlerbehandlung. Für einfache Genehmigungen ist Power Automate häufig die pragmatischere Wahl.

## Merksätze

- **Durable Functions:** zustandsbehaftete, langlebige Serverless-Workflows.
- **`DurableTaskClient`:** Orchestrierungen von außen starten und verwalten.
- **`TaskOrchestrationContext`:** den Ablauf im Orchestrator steuern.
- **`CreateTimerAsync()`:** persistent verwaltete Wartezeit.
- **`WaitForExternalEvent()`:** auf ein externes Ereignis warten.
- **Activities:** Arbeit und Seiteneffekte ausführen; Wiederholungen berücksichtigen.
