---
categories:
  - "[[PL-400]]"
tags:
  - Dataverse
  - Solutions
created: 2026-11-16
---
# Dataverse Solution Layers

**Solution Layers** zeigen, wie Dataverse Anpassungen derselben Komponente aus verschiedenen Solutions verwaltet. Das betrifft zum Beispiel Forms, Views, Tabellen und Attribute.

## Managed Solution Layers

Managed Solutions bilden eine Layer-Reihenfolge. Werden mehrere Managed Solutions mit Anpassungen derselben Komponente importiert, liegt die zuletzt importierte Solution in der Regel über den früheren Managed Layers.

Welche Anpassungen wirksam sind, hängt vom Komponententyp ab. Bei Forms gibt es Merge-Verhalten; die oberste Managed Layer ersetzt daher nicht automatisch alle Anpassungen der darunterliegenden Layers.

## Active Unmanaged Layer

Eine direkt in einer Umgebung veröffentlichte, nicht verwaltete Anpassung kann eine **Active Unmanaged Layer** erzeugen. Diese liegt über den Managed Layers und kann deren Änderungen überlagern.

Beispiel:

1. Eine Managed Solution verschiebt ein Feld auf einem Form.
2. In der Zielumgebung wird das Feld direkt am Form erneut verschoben.
3. Die direkte Änderung liegt in der Active Unmanaged Layer.
4. Beim Import einer neuen Managed-Version wird zwar der Managed Layer aktualisiert, die Active Unmanaged Layer bleibt jedoch bestehen und kann die neue Anpassung weiterhin überlagern.

## Praktische Konsequenz

Bei unerwarteten Abweichungen zwischen einer importierten Solution und der sichtbaren Anpassung lohnt sich ein Blick auf die Layer der betroffenen Komponente. Direkte Anpassungen in einer Zielumgebung können verwaltete Änderungen verdecken; Änderungen sollten daher kontrolliert über die vorgesehenen Solutions transportiert werden.

## Merksätze

- **Solution Layers:** bestimmen, wie Anpassungen derselben Komponente zusammenwirken.
- **Managed Layers:** werden durch importierte Managed Solutions aufgebaut und aktualisiert.
- **Active Unmanaged Layer:** liegt über Managed Layers und kann deren Änderungen überlagern.
- Ein Managed-Solution-Import entfernt eine bestehende Active Unmanaged Layer nicht automatisch.
