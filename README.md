# Einfluss der Semantik auf Merkmalsextraktoren und Matcher

Bachelorarbeit im Studiengang Software Engineering, Universität Duisburg-Essen (Note 1,3).

*English summary: Evaluation pipeline that compares 6 feature detectors and 5 matching methods (30 combinations) on the HPatches dataset and tests whether filtering matches with semantic object annotations (COCO format) improves their quality.*

## Fragestellung

Klassische Verfahren zur Merkmalsextraktion und zum Matching bewerten Bildpunkte nur nach ihrem Aussehen. Dieses Projekt untersucht, ob zusätzliche **semantische Information** hilft: Ein Match gilt nur dann als plausibel, wenn beide Punkte zu derselben Objektklasse gehören.

## Methode

![Arbeitsablauf der Evaluierungspipeline](docs/pipeline_overview.png)

1. **Daten:** HPatches-Bildsequenzen mit Ground-Truth-Homographien, auf 480 × 640 Pixel vereinheitlicht.
2. **Merkmalsextraktion (CPU):** SIFT, ORB, BRISK, SuperPoint, DISK, KeyNetHardNet. Keypoints, Deskriptoren und Laufzeiten werden gespeichert.
3. **Ergänzende GPU-Messung:** Laufzeit der CNN-Detektoren auf der GPU. Sie gehört nicht zum Hauptvergleich.
4. **Geometrisches Matching:** BruteForce, FLANN, KNN+Lowe (Ratio 0,75), FAISS und CrossCheck. Die Matches werden mit der Ground-Truth-Homographie bewertet (TP, FP, FN, TN, daraus Accuracy, Precision, Recall und F1).
5. **Matching mit semantischer Filterung:** Dieselben gespeicherten Extraktionsdaten werden verwendet. Ein Match bleibt nur erhalten, wenn beide Keypoints in Annotationsboxen **derselben Klasse** liegen (oder beide außerhalb aller Boxen). Danach folgt dieselbe geometrische Bewertung.
6. **Vergleich:** Geometrisch gegenüber geometrisch + Semantik, als Diagramme für alle 30 Kombinationen.

**Bewertung:** Ein Match zählt als korrekt, wenn der Punkt nach Projektion mit der Ground-Truth-Homographie weniger als 3 Pixel (Beleuchtungs-Szenen) bzw. 30 Pixel (Blickwinkel-Szenen) vom Partnerpunkt entfernt liegt.

## Datensatz

- **HPatches** (116 Szenen: 57 mit Beleuchtungswechsel `i_*`, 59 mit Blickwinkelwechsel `v_*`): [github.com/hpatches/hpatches-dataset](https://github.com/hpatches/hpatches-dataset). Die Bilder liegen **nicht** in diesem Repository.
- **Semantische Annotationen:** In `data/hpatches-yolo-jpg/` liegen pro Szene die selbst erstellten Objektannotationen im COCO-Format (`_annotations.coco.json`, Annotation mit Roboflow).

## Ergebnisse

Hauptvergleich über alle 30 Kombinationen (6 Detektoren × 5 Matcher):

| Kennzahl | Wirkung der semantischen Filterung |
|---|---|
| **Precision** | steigt in allen 30 Kombinationen |
| **Anteil falscher Matches (FP)** | sinkt |
| **Recall** | sinkt |
| **F1** | sinkt |
| **Laufzeit** | steigt durch den Filterschritt |

Beispiele: SIFT mit FAISS: Precision 16,3 % → 20,8 %, Anteil falscher Matches 62,8 % → 41,7 %, Recall 44,4 % → 27,7 %. DISK mit KNN+Lowe: Precision 65,0 % → 65,9 %.

![Precision](results/compare_geo_vs_sem_precision.png)
![Recall](results/compare_geo_vs_sem_recall.png)
![F1-Score](results/compare_geo_vs_sem_f1.png)
![Anteil falscher Matches](results/compare_geo_vs_sem_pct_b.png)
![Laufzeit](results/compare_geo_vs_sem_time_matching_plus_filter.png)

Schraffierte Balken zeigen die Variante mit semantischer Filterung. Weitere Diagramme (Accuracy, TP, FN, TN, Extraktion und CPU/GPU-Vergleich) liegen ebenfalls in `results/`.

## Fazit

Die semantische Filterung **tauscht Recall gegen Precision**: Es bleiben weniger, dafür zuverlässigere Matches. Das lohnt sich, wenn falsche Korrespondenzen teuer sind. Wo möglichst viele richtige Matches gebraucht werden, überwiegt der Recall-Verlust. Außerdem kostet der Filterschritt zusätzliche Laufzeit.

**Grenzen:** Die Annotationen sind Bounding-Boxen und decken Objekte nicht pixelgenau ab. Punkte außerhalb aller Boxen erhalten die Klasse „background". Die Ergebnisse gelten für HPatches und die gewählten Parameter.

## Repository-Struktur

```
Bachelorarbeit_Pipeline.ipynb   Pipeline (Extraktion, Matching, Auswertung, Diagramme)
requirements.txt                Python-Pakete
data/hpatches-yolo-jpg/         COCO-Annotationen pro Szene
results/                        Ergebnis-Diagramme
docs/                           Ablaufdiagramm, Präsentation
```

## Ausführen

1. Pakete installieren (Python 3.10 oder neuer empfohlen):
   ```
   pip install -r requirements.txt
   ```
   Für die GPU-Messung PyTorch passend zur CUDA-Version installieren (Hinweis in `requirements.txt`).
2. HPatches herunterladen und so ablegen, dass der Ordner `data/hpatches-sequences-release/` entsteht. Die Annotationen liegen bereits in `data/hpatches-yolo-jpg/`.
3. `Bachelorarbeit_Pipeline.ipynb` öffnen und die Zellen von oben nach unten ausführen. Die Pfade lassen sich über die Umgebungsvariablen `DATA_DIR` und `OUTPUT_DIR` ändern. Ergebnisse und Zwischenspeicher landen in `outputs/`.

Hinweis: Die Extraktion über alle Bilder und Detektoren dauert lange und erzeugt einen großen Zwischenspeicher (`.pkl`). Er wird nicht ins Repository aufgenommen.

## Verwendete Bibliotheken

Python, OpenCV, PyTorch, Kornia (DISK, KeyNetHardNet), LightGlue (SuperPoint), FAISS, pandas, NumPy, matplotlib, seaborn.

## Quellen und Lizenz

- HPatches: Balntas et al., *HPatches: A benchmark and evaluation of handcrafted and learned local descriptors*, CVPR 2017.
- Code und Annotationen: MIT-Lizenz (siehe `LICENSE`).
