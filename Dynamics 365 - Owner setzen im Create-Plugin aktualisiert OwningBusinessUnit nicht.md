---
categories:
  - "[[D365]]"
  - "[[Help]]"
tags:
  - Plugin
  - Dataverse
  - Ownership
created: 2026-08-26
---
# Owner setzen im Create-Plugin aktualisiert OwningBusinessUnit nicht

## Problem

Wenn man in einem Plugin auf **Create** den `ownerid` eines neuen Records direkt als Attribut setzt (egal ob in PreValidation, PreOperation oder als Attribut-Mutation in PostOperation), wird `owningbusinessunit` von der Plattform **nicht zuverlässig neu berechnet**. Der Record landet mit der Business Unit des erstellenden Users (bzw. einer Default-BU) statt der BU des tatsächlich zugewiesenen Owners (z. B. eines Teams).

Beobachtet an zwei fast identischen `ava_dailylist`-Records (Avanade.BBU.CSM, Tagesliste-Plugin, Owner sollte bei `ava_regardingbusinessunit = RKS` automatisch auf ein RKS-Team gesetzt werden):

| | funktioniert | funktioniert nicht |
|---|---|---|
| `createdon` | 13:38:44 | 13:32:47 |
| `modifiedon` | 13:39:06 (**später separat geändert**) | 13:32:47 (**identisch mit createdon**) |
| `ownerid` | RKS-Team ✅ | RKS-Team ✅ |
| `owningbusinessunit` | RKS (Rückkehrberatung) ✅ | Root-BU der Organisation ❌ |

In **beiden** Fällen war `ownerid` korrekt gesetzt. Der einzige Unterschied: Der funktionierende Record wurde nach dem Create noch **einmal separat geändert** (z. B. manuell in der UI der Owner erneut gesetzt) – das hat die BU nachträglich korrigiert.

## Ursache

`owningbusinessunit` ist im Early-Bound-Code bewusst **ohne Setter** generiert (systemverwaltetes Feld). Die Plattform berechnet es aus dem Owner – aber diese Neuberechnung läuft zuverlässig nur bei einem **Update/Assign auf einen bereits existierenden Record**, nicht wenn `ownerid` während des **Create selbst** von einem Plugin verändert wird. Ein reines Attribut-Set auf `target.OwnerId` im Create-Kontext wird für die Ownership-Kaskade (inkl. `owningbusinessunit`) nicht zuverlässig berücksichtigt.

## Fix

Owner nicht direkt als Attribut im Create setzen, sondern **nach** dem Create per `AssignRequest` zuweisen – das ist derselbe Vorgang wie ein manueller Owner-Wechsel in der UI und triggert die volle Ownership-Kaskade zuverlässig:

```csharp
// Core/Entities/ava_dailylist.cs (partial class)
public void SetOwnerToRKSTeam(XrmServiceContext ctx, ava_dailylist target, ITracingService trace = null)
{
    if (this.ava_RegardingBusinessUnitEnum != ava_regardingbusinessunit.RKS)
        return;

    var teamId = new EnvironmentVariableHelper(ctx).RetrieveEnvironmentVariable("ava_TageslisteRKSTeamID");
    if (string.IsNullOrEmpty(teamId))
        return;

    ctx.OrganizationService.Execute(new AssignRequest
    {
        Assignee = new EntityReference(Team.EntityLogicalName, Guid.Parse(teamId)),
        Target = target.ToEntityReference(),
    });
}
```

Registrierung dafür auf **PostOperation** (nicht PreOperation!), weil `AssignRequest` einen bereits existierenden, committeten Record braucht – siehe [[Dynamics 365 - Plugin Images & Stages]]:

```csharp
[CrmPluginRegistration(MessageNameEnum.Create, ava_dailylist.EntityLogicalName, StageEnum.PostOperation,
    ExecutionModeEnum.Synchronous, "", "...PostCreate", 1, IsolationModeEnum.Sandbox)]
public class PostCreate : XrmPluginBase<ava_dailylist>, IPlugin
{
    protected override void ExecuteCrmPlugin(XrmPluginContext<ava_dailylist> context)
    {
        context.Subject.SetOwnerToRKSTeam(context.CrmSystemContext, context.Target, context.TracingService);
    }
}
```

## Wichtig: System- vs. User-Context

Sobald die Zuweisung über `ctx.OrganizationService.Execute(AssignRequest)` läuft (statt reiner Attribut-Mutation), macht der übergebene Context einen echten Unterschied:

- **CrmUserContext**: `AssignRequest` läuft mit den Privilegien des erstellenden Users. Fehlt diesem die "Assign"-Berechtigung auf die Entität oder die nötige Stufe (User/BU/Parent-Child/Organisation) für eine Cross-Business-Unit-Zuweisung an ein Team, schlägt es mit einer Security-Exception fehl.
- **CrmSystemContext**: läuft mit vollen Systemrechten, die Zuweisung klappt unabhängig von der Sicherheitsrolle des erstellenden Users.

→ Für automatisierte Business-Rule-Zuweisungen (wie "Owner = Team X wenn Feld Y = Z") **CrmSystemContext** verwenden, nicht CrmUserContext.

## Faustregel

> [!tip]
> Owner eines **neuen** Records auf ein Team/einen User setzen, dessen Business Unit sich vom Ersteller unterscheidet → niemals `target.OwnerId = ...` im Create, sondern `AssignRequest` in PostOperation mit `CrmSystemContext`.

## Tags
#dynamics365 #plugin #dataverse #ownership
