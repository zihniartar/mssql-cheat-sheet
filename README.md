# Automatisierte Wartung mit dynamischem T-SQL
Reorganize indexes in Smartstore DB tables

## Skript zur Analyse und Behandlung der Fragmentierung
Das folgende Skript prüft den Fragmentierungsgrad und wendet REORGANIZE oder REBUILD je nach Schwellenwert an:
```sql
DECLARE @TableName NVARCHAR(128);
DECLARE @IndexName NVARCHAR(128);
DECLARE @SchemaName NVARCHAR(128);
DECLARE @SQL NVARCHAR(MAX);

DECLARE IndexCursor CURSOR FOR
SELECT 
    s.name AS SchemaName,
    t.name AS TableName,
    i.name AS IndexName
FROM 
    sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') AS ips
    INNER JOIN sys.indexes AS i
        ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    INNER JOIN sys.tables AS t
        ON i.object_id = t.object_id
    INNER JOIN sys.schemas AS s
        ON t.schema_id = s.schema_id
WHERE 
    ips.avg_fragmentation_in_percent > 5; -- Fragmente über 5% analysieren

OPEN IndexCursor;

FETCH NEXT FROM IndexCursor INTO @SchemaName, @TableName, @IndexName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = '';
    -- Ermitteln des Fragmentierungsgrades
    DECLARE @FragmentationPercent FLOAT;
    SELECT @FragmentationPercent = ips.avg_fragmentation_in_percent
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
    INNER JOIN sys.indexes i
        ON ips.object_id = i.object_id AND ips.index_id = i.index_id
    WHERE i.name = @IndexName;

    -- Aktion basierend auf Fragmentierungsgrad
    IF @FragmentationPercent BETWEEN 5 AND 30
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REORGANIZE;';
    ELSE IF @FragmentationPercent > 30
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REBUILD;';

    IF @SQL <> ''
        EXEC sp_executesql @SQL;

    FETCH NEXT FROM IndexCursor INTO @SchemaName, @TableName, @IndexName;
END;

CLOSE IndexCursor;
DEALLOCATE IndexCursor;
```
Fragmentierungsgrad:
Zwischen 5% und 30%: Reorganize.
Über 30%: Rebuild.

## Wartungsplan mit SQL Server Agent
Automatisieren Sie den Prozess mit einem Wartungsplan oder SQL Server Agent:

SQL Server Management Studio (SSMS):

Navigieren Sie zu Management > Maintenance Plans > New Maintenance Plan.
Fügen Sie die Aufgabe Rebuild Index Task oder Reorganize Index Task hinzu.
Wählen Sie die Ziel-Datenbanken und die zu bearbeitenden Indizes aus.
Planen Sie den Wartungsplan (z. B. wöchentlich in der Nacht).
Mit SQL Server Agent (T-SQL-Job):

Erstellen Sie einen neuen Job im SQL Server Agent.
Verwenden Sie das oben angegebene Skript oder die Befehle REORGANIZE und REBUILD.
Planen Sie den Job (z. B. einmal pro Woche).


# Kompatibilitäts-Level einer DB über eine Abfrage anzeigen und ändern

### Anzeigen:
```sql
SELECT name, compatibility_level 
FROM sys.databases 
WHERE name = 'datenbankname';
```

### Ändern:
```sql
ALTER DATABASE [datenbankname]
SET COMPATIBILITY_LEVEL = 130;
```

Level 130 entspricht SQL Server 2016. Hier sind die Levels für verschiedene SQL Server-Versionen:

150: SQL Server 2019
140: SQL Server 2017
130: SQL Server 2016
120: SQL Server 2014
110: SQL Server 2012

# Eigenschaft einer Spalte in Microsoft SQL Server von NOT NULL auf NULL ändern

### Syntax für die Tabele Setting und für die Spalte Value
```sql
ALTER TABLE Setting ALTER COLUMN Value NVARCHAR(MAX) NULL;
```

