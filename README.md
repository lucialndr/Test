# Test


Basierend auf der aktuellen Hugging Face Doku hier die Struktur, die du nachbauen musst:

## Grundprinzip

Ein HF-Dataset-Repo ist im Kern ein Git-Repo mit Datendateien plus einer `README.md`, deren **YAML-Frontmatter** (der Block zwischen `---`) die gesamte Metainformation zu Subsets/Configs, Splits und Schema enthält. Alles andere (Namenskonventionen der Dateien, Ordnerstruktur) ist optional automatisierbar, wenn du dich an Muster hältst – aber die YAML-Konfiguration ist die robuste, explizite Variante, die du für "genauso wie HF" nachbauen willst.

## 1. Automatische Erkennung (ohne YAML)

Wenn Dateinamen bestimmten Mustern folgen, erkennt `load_dataset()` Splits automatisch:

```
my_dataset/
├── README.md
├── train.csv
└── test.csv
```

Das ergibt automatisch zwei Splits `train` und `test`. Ordner/Dateinamen mit `train`, `test`, `validation` (auch `val`, `dev`) im Namen werden als Split-Marker erkannt.

## 2. Manuelle Konfiguration über YAML (`configs`-Feld)

Das ist der Kern für **Subsets** (= "configs" im HF-Jargon) **und Splits**:

```yaml
---
configs:
  - config_name: default
    data_files:
      - split: train
        path: "data/train-*.parquet"
      - split: test
        path: "data/test-*.parquet"
---
```

Für mehrere **Subsets**, definierst du mehrere Einträge unter `configs`:

```yaml
---
configs:
  - config_name: catalan
    data_files:
      - split: train
        path: "catalan/train-*"
      - split: test
        path: "catalan/test-*"
      - split: validation
        path: "catalan/validation-*"
  - config_name: spanish
    data_files:
      - split: train
        path: "spanish/train-*"
      - split: test
        path: "spanish/test-*"
    default: true
---
```

Jedes Subset bekommt so ein eigenes Dropdown im Dataset Viewer und wird separat geladen: `load_dataset("repo", "catalan")`.

## 3. Übliche Ordnerstruktur (Konvention der offiziellen HF-Datasets)

Die meisten offiziellen Datasets, die zu Parquet konvertiert wurden, sehen so aus:

```
my_dataset_repository/
├── README.md
├── default/                          # oder Subset-Name als Ordner
│   ├── train-00000-of-00007.parquet
│   ├── train-00001-of-00007.parquet
│   ├── test-00000-of-00001.parquet
│   └── validation-00000-of-00001.parquet
└── subset2/
    ├── train-00000-of-00001.parquet
    └── test-00000-of-00001.parquet
```

Namenskonvention für gesharded Dateien: `{split}-{shard_index:05d}-of-{num_shards:05d}.parquet`

## 4. Zusätzliche Metadaten im YAML: `dataset_info`

Neben `configs` (Dateizuordnung) gibt es `dataset_info` – das beschreibt Schema, Feature-Typen und Statistiken pro Split/Subset:

```yaml
dataset_info:
  - config_name: catalan
    features:
      - name: id_str
        dtype: string
      - name: label
        dtype:
          class_label:
            names:
              '0': AGAINST
              '1': FAVOR
              '2': NEUTRAL
    splits:
      - name: train
        num_bytes: 976635
        num_examples: 6028
      - name: test
        num_bytes: 331730
        num_examples: 2010
    download_size: 1070281
    dataset_size: 1308365
```

Dieser Block ist **optional für das Laden**, wird aber von `datasets` genutzt, um Schema/Größe anzuzeigen, ohne die Daten erst herunterzuladen (Streaming/Viewer-Vorschau).

## 5. Weitere YAML-Metadaten-Felder (Dataset Card)

Zusätzlich üblich, aber nicht ladelogik-relevant, eher für den Hub-Viewer/Discovery:
- `license`, `language`, `pretty_name`, `tags`, `task_categories`, `size_categories`, `annotations_creators` etc.

## Zusammenfassung: Was du wo speichern musst

| Info | Speicherort |
|---|---|
| Subsets (Configs) | `configs:` Liste in README.md YAML, je ein `config_name` |
| Splits & Dateizuordnung | `data_files:` innerhalb jedes `configs`-Eintrags |
| Schema/Feature-Typen | `dataset_info.features` in README.md YAML |
| Split-Statistiken (Größe, Anzahl) | `dataset_info.splits` in README.md YAML |
| Eigentliche Daten | Parquet/CSV/JSONL-Dateien im Repo, in Ordnern je Subset |
| Dataset Card (Beschreibung) | Freitext-Markdown unterhalb des YAML-Blocks |

**Wichtig:** Das früher genutzte `dataset_infos.json` als separate Datei ist mittlerweile **deprecated/legacy** – bei aktuellen Datasets ist alles ins README-YAML gewandert. Für maximale Kompatibilität solltest du dich also rein auf README.md-YAML (`configs` + `dataset_info`) plus Parquet-Dateien stützen, ganz ohne separates JSON und ohne Loading-Script (`.py`-Dateien sind ebenfalls veraltet).

Möchtest du, dass ich dir ein konkretes Vorlagen-Template (README.md + Ordnerstruktur) für dein spezifisches Dataset erstelle?
