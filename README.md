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

Gute Frage – die Antwort: **Nein, es gibt keine eingebaute Split-Spalte pro Zeile.** Die Zugehörigkeit zu einem Split wird rein über die **Datei** bestimmt, nicht über ein Feld im Sample selbst.

## Split-Zugehörigkeit = Dateizuordnung, nicht Zeileninhalt

Wenn eine Zeile in `train-00000-of-00001.parquet` liegt, ist sie automatisch Teil des `train`-Splits. Es gibt standardmäßig **kein** `split`-Feld, **kein** automatisches `id`/`idx`-Feld und **keine** Subset-Kennung im Sample selbst – außer du fügst sowas explizit als eigene Spalte in deine Daten ein.

Einzige Ausnahme: Bei bild-/audio-basierten Ordnerstrukturen (Klassifikation) wird der Ordnername automatisch als `label`-Spalte in jede Zeile geschrieben (kannst du mit `drop_labels: true` abschalten) – das ist aber Sonderfall, kein generelles Prinzip.

## Was pro Sample tatsächlich gespeichert wird

Nur die Feature-Spalten, wie sie im Schema definiert sind (bei Parquet: die Parquet-Columns selbst). Für dein Beispiel `sentence1`, `sentence2`, `label` – jede Zeile enthält exakt diese Werte, nichts weiter.

## Aber: Es gibt Metadaten auf Dateiebene (Parquet-Footer)

Hier wird's interessant: `datasets` schreibt beim Konvertieren zu Parquet das komplette Feature-Schema als **Key-Value-Metadata in den Parquet-Footer** der jeweiligen Datei – unter dem Key `"huggingface"`. Das sieht z. B. so aus:

```python
import pyarrow.parquet as pq
metadata = pq.read_metadata("train-00000-of-00001.parquet")
json.loads(metadata.metadata[b"huggingface"])
# {'info': {'features': {'sentence1': {'dtype': 'string', '_type': 'Value'},
#                         'label': {'names': ['entailment', 'not_entailment'],
#                                   '_type': 'ClassLabel'}}}}
```

Das ist **Datei-Metadaten**, nicht Zeilen-Metadaten – aber es ist der Mechanismus, mit dem `datasets` beim Laden prüft, ob das Schema zur README-YAML-Deklaration (`dataset_info.features`) passt. Passt es nicht überein, gibt's genau den `CastError`, den man häufig im Dataset Viewer sieht, wenn Spaltennamen nicht übereinstimmen.

## Zusammenfassung

| Ebene | Was steht da |
|---|---|
| **Zeile/Sample** | Nur die Feature-Werte selbst (keine Split-/Subset-Info) |
| **Datei** (Parquet-Footer, `huggingface`-Key) | Feature-Schema (Typen, ClassLabel-Namen etc.) |
| **Dateiname/Pfad** | Bestimmt Split-Zugehörigkeit (per Konvention oder YAML `data_files`) |
| **README.md YAML** (`configs.data_files`) | Explizite Zuordnung Datei → Split, Datei → Subset |
| **README.md YAML** (`dataset_info`) | Erwartetes Schema + Statistiken (Soll-Zustand zum Abgleich) |

Für dein eigenes kompatibles Format heißt das praktisch: Du musst pro Zeile **nichts extra** speichern – du musst nur konsequent sein bei **Dateibenennung/-ablage** (welche Datei = welcher Split/Subset) und das im README-YAML korrekt deklarieren. Willst du, dass ich dir zeige, wie du beim Schreiben mit `datasets.Dataset.to_parquet()` oder `push_to_hub()` dieses Footer-Schema automatisch mitgeneriert bekommst, statt es manuell zu pflegen?

Wichtige Unterscheidung zuerst: Es gibt zwei völlig unterschiedliche "Arrow"-Kontexte bei `datasets`, und das solltest du bei deinem Nachbau nicht verwechseln.

## 1. Wie Audio überhaupt strukturell gespeichert wird (unabhängig von Arrow vs. Parquet)

Die `Audio`-Feature ist intern als **Struct-Spalte** definiert:

```python
pa_type = pa.struct({"bytes": pa.binary(), "path": pa.string()})
```

Jedes Audio-Sample ist also **eine Zelle** in der Tabelle mit zwei Unterfeldern:
- `bytes`: die rohen Audiodaten (z. B. WAV/MP3-Bytes) direkt eingebettet, binär
- `path`: optionaler Dateiname/Pfad (z. B. `"clip_0001.wav"`), rein informativ

Das gilt **gleichermaßen** in Arrow- und in Parquet-Dateien, weil ein Arrow-Struct 1:1 auf einen Parquet-Struct-Type abbildet. Die Audiodaten liegen also direkt **in der Tabelle**, nicht als separate Dateien daneben – es sei denn, du entscheidest dich bewusst für externe Referenzierung (dann ist `bytes` null und nur `path` gesetzt, was aber offline nicht ladbar ist ohne die referenzierten Dateien).

## 2. `.arrow`-Dateien = lokales Cache-Format, NICHT das Hub-Upload-Format

Wenn du lokal `dataset.save_to_disk("mein_pfad")` aufrufst, entsteht eine völlig andere Struktur als beim Hub-Upload:

```
mein_pfad/
├── dataset_dict.json          # listet die Splits
├── train/
│   ├── dataset_info.json      # Features, Beschreibung, Split-Infos
│   ├── state.json             # welche .arrow-Dateien, Fingerprint
│   └── data-00000-of-00001.arrow
└── test/
    ├── dataset_info.json
    ├── state.json
    └── data-00000-of-00001.arrow
```

Das ist das **Arrow-IPC/Feather-Format**, optimiert für schnelles Memory-Mapping lokal – nicht für Web-Hosting gedacht.

## 3. Wichtig: `push_to_hub()` konvertiert IMMER zu Parquet

Wenn du ein Dataset via `push_to_hub()` hochlädst, wandelt `datasets` die Arrow-Tabellen **automatisch in Parquet** um. Auf dem Hub selbst liegen so gut wie nie `.arrow`-Dateien – der Standard ist Parquet (wie in meiner vorherigen Antwort beschrieben), auch für Audio/Bild-Datensätze. Der Struct mit `bytes`+`path` wandert einfach unverändert in die Parquet-Spalte.

## 4. Kannst du trotzdem `.arrow`-Dateien auf dem Hub hosten?

Ja, technisch geht das – `datasets` hat auch einen Arrow-Loader (`packaged_modules/arrow`), analog zu CSV/JSON/Parquet. Du könntest in der README-YAML einfach auf `.arrow`-Dateien verweisen:

```yaml
configs:
  - config_name: default
    data_files:
      - split: train
        path: "data/train-*.arrow"
```

Aber: Das ist unüblich, wird vom Dataset Viewer schlechter unterstützt (kein Streaming-Vorteil wie bei Parquet, größere Dateien, keine Kompression), und **kein einziges** offizielles HF-Dataset macht das so. Für "genauso wie die Open-Source-Datasets" solltest du also bei **Parquet** bleiben.

## Zusammenfassung

| Frage | Antwort |
|---|---|
| Wo liegen Audio-Bytes? | Direkt als `bytes`-Feld im Struct, in derselben Zeile/Tabelle |
| Braucht's separate Audiodateien neben der Tabelle? | Nein, wenn `bytes` gesetzt ist – alles ist eingebettet |
| Ist `.arrow` das Hub-Format? | Nein – `.arrow` ist lokales Cache-Format (`save_to_disk`), Hub nutzt Parquet |
| Metadaten bei `.arrow`? | `dataset_info.json` + `state.json` pro Split, `dataset_dict.json` auf oberster Ebene |
| Soll ich für Hub-Kompatibilität `.arrow` nutzen? | Nein, konvertiere zu Parquet (z. B. via `push_to_hub()` oder `to_parquet()`) |

Willst du, dass ich dir zeige, wie du ein bestehendes Audio-Dataset (z. B. Ordner mit `.wav`-Dateien + Metadaten-CSV) direkt in dieses Struct-Format packst und als Parquet exportierst?
