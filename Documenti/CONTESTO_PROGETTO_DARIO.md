# Contesto progetto Scout Volley CV & NLP

**Aggiornato al 17 novembre 2026**  
Cartella di lavoro: `C:\Users\SavarDar\OneDrive - Electrolux\Desktop\Scout_Volley_CV&NLP`

Questo documento riassume lo stato verificabile dei notebook, delle configurazioni e delle run presenti. Non è una dichiarazione di accuratezza generale né un versionamento del progetto. Le metriche di detection sono distinte dai test sintetici e dalle prove video. Per gli output modificabili dall’utente fa fede la configurazione contenuta nella relativa run; gli output incorporati nei notebook possono essere storici.

## Obiettivo e perimetro

Il progetto prepara strumenti separati per:

- fine-tuning e tracking 2D del pallone;
- fine-tuning e tracking dei giocatori;
- omografia del pavimento, zone del campo e calibrazione camera sperimentale.

Il video di riferimento è `Annotazioni\ITA_POL_VNL2024_30fps.mp4`, ripresa fissa a 30 fps e 1280×720. I notebook non forniscono, allo stato attuale, una ricostruzione 3D valida della camera o del volo della palla.

## Dataset e fonti

| Uso | Dati presenti nella cartella | Classi/split | Licenza e provenienza |
|---|---|---|---|
| Pallone | `Volleyball Ball Tracking.v1i.yolov8` | `ball`; 1.647 train, 185 validation, 192 test immagini | YAML locale: MIT; progetto Roboflow `savaresi-dario-gmail-com/volleyball-ball-tracking-kidsa`, versione 1. La pagina Roboflow è indicata nel dataset come [fonte](https://app.roboflow.com/savaresi-dario-gmail-com/volleyball-ball-tracking-kidsa/1). |
| Giocatori | `Tracking Volleyball Players.v2i.yolov8` | `player`, `refree`; 1.155 train, 330 validation, 164 test immagini | YAML locale: CC BY 4.0; progetto Roboflow `savaresi-dario-gmail-com/tracking-volleyball-players-hnhab`, versione 2. La classe `refree` mantiene l’ortografia originale del dataset. [Fonte](https://app.roboflow.com/savaresi-dario-gmail-com/tracking-volleyball-players-hnhab/2). |

Il dataset giocatori inizialmente non era presente quando fu predisposto il notebook; in seguito è stato scaricato e utilizzato per un training reale. Non è il dataset palloni rinominato o riutilizzato. Un ulteriore candidato discusso, ma non scaricato qui, è [Volleyball Player](https://universe.roboflow.com/yan-personal-dataset/volleyball-player): circa 125 immagini, classi `person` e `sports ball`, CC BY 4.0. È una fonte da valutare, non una garanzia di qualità.

## Fine-tuning YOLOv8m

I due notebook sono distinti: `Fine_tuning_palloni_YOLOv8.ipynb` e `Fine_tuning_giocatori_YOLOv8.ipynb`. Entrambi usano il checkpoint locale `yolov8m.pt` invece di dipendere da un download durante l’avvio del training. Un precedente download dei pesi da GitHub ha incontrato un errore TLS `CERTIFICATE_VERIFY_FAILED`; non sono stati disabilitati i controlli TLS né modificati globalmente i certificati. La disponibilità locale evita quel download; non identifica la causa dell’errore TLS originale.

I valori seguenti provengono dai CSV di valutazione conservati nelle run. Sono metriche di **object detection** (`precision`, `recall`, `mAP`), non classification accuracy.

| Modello / split | Precision | Recall | mAP50 | mAP50–95 | Immagini |
|---|---:|---:|---:|---:|---:|
| Pallone, validation | 0,834290 | 0,952500 | 0,960817 | 0,515068 | 185 |
| Pallone, test | 0,920887 | 0,904676 | 0,951795 | 0,540161 | 192 |
| Giocatori, validation, media Ultralytics | 0,986589 | 0,982647 | 0,993366 | 0,735035 | 330 |
| Giocatori, validation, `player` | 0,985397 | 0,978500 | 0,993824 | 0,725760 | 330 |
| Giocatori, test, media Ultralytics | 0,988031 | 0,982202 | 0,991844 | 0,737085 | 164 |
| Giocatori, test, `player` | 0,985559 | 0,974018 | 0,990760 | 0,717861 | 164 |

Il dataset giocatori conserva entrambe le classi: `refree` non è convertita in giocatore e il training non usa `single_cls`. Le metriche per `refree` sono disponibili in `runs\volleyball_players\yolov8m_players_20261117_083439\metriche_detection_per_classe.csv`. Un detector `person` o una classe `player` non determina da solo il ruolo: allenatori, arbitri o persone a bordo campo possono essere rilevati come persone/giocatori se le annotazioni del dataset non li distinguono coerentemente.

Checkpoint:

- Pallone: `runs\volleyball_ball\yolov8m_ball_20261114_172150\weights\best.pt`.
- Giocatori: `runs\volleyball_players\yolov8m_players_20261117_083439\weights\best.pt`.
- Pesi iniziali locali: `yolov8m.pt`.

Le configurazioni di training riportano YOLOv8m pretrained, CUDA, 640 pixel, batch 16 e `workers=0`. Il training palla è configurato per 100 epoche e `results.csv` contiene 100 righe; per i giocatori la configurazione indica 100 epoche mentre `results.csv` contiene 73 righe. Le metriche della tabella provengono dai file valutazione val/test, non sono inferite dal numero di epoche.

## Tracking del pallone

Notebook: `Tracking_pallone_YOLO_Kalman.ipynb`.

- Detector YOLO e tracker monopalla: seleziona la detection con confidence più alta per frame; non fa re-identificazione e può saltare fra più palloni, ad esempio in riscaldamento.
- La posizione `OSS` è la detection YOLO grezza. Il filtro Kalman CA lavora in coordinate immagine `[x, y, vx, vy, ax, ay]`, non in coordinate fisiche 3D. `PRED` viene emessa solo quando manca una detection.
- Configurazione attualmente salvata nel notebook: confidence 0,77; stride 1; `MAX_MISSED_SECONDS=0,75`, `MAX_PREDICTION_SECONDS=0,15`, deviazione standard massima della posizione prevista 30 px; limite di Mahalanobis al quadrato 25. Una run successiva registrò confidence 0,765 e orizzonte PRED 0,3 s: per riprodurre un output va letto il suo `configurazione.json`, non assunto che coincida con i valori correnti del notebook.
- La confidence non è usata come varianza spaziale; il filtro non rileva tocchi nascosti, non modella la traiettoria balistica e non stima il punto di contatto.
- La proiezione omografica al pavimento aggiunta in un esperimento è stata successivamente rimossa su richiesta. Il notebook di tracking palla corrente non deve essere interpretato come produttore di coordinate di campo o 3D.
- La preview ordinaria viene salvata dal **primo frame processato dell’intervallo**, anche se non contiene detection. Per esempio, nella run che inizia a 4740 s le prime decine di frame non avevano detection; la preview iniziale può quindi apparire vuota e non essere rappresentativa.

Esistono annotazioni manuali del pallone in `Annotazioni\ITA_POL_VNL2024_30fps_FP_palla.csv` e valutazioni pixel in `runs\ball_tracking\...`. Per la valutazione sull’intervallo 1150–1200 s sono documentati 132 frame visibili, tolleranza spaziale 25 px:

- output YOLO osservato: 106/132 frame GT coperti; recall entro tolleranza 80,30%. Tutte le 106 osservazioni confrontate erano entro 25 px; errore medio 3,82 px, RMSE 4,36 px;
- output PRED: 22 predizioni confrontabili, 16 entro tolleranza; errore medio 69,73 px, mediana 11,71 px, RMSE 170,36 px;
- combinando OSS e PRED: 122/132 frame visibili entro tolleranza (92,42%); 4 frame senza output.

Questi numeri descrivono quella specifica annotazione/intervallo e non sono una precision globale: il file GT non certifica esplicitamente l’assenza di pallone negli altri frame, quindi la precisione/falsi positivi non è stata valutata. Alcuni file metriche ripetono lo stesso confronto; non sono prove indipendenti.

Run notevoli: `runs\ball_tracking\ITA_POL_VNL2024_30fps_20261115_131753` (estratto 1150–1200 s) e `runs\ball_tracking\ITA_POL_VNL2024_30fps_20261116_081527` (elaborazione più lunga; la valutazione GT usa comunque i frame annotati). È presente anche `runs\ball_tracking\ITA_POL_VNL2024_30fps_20261117_081248`, 6.211 frame nell’intervallo 4740–4947 s, confidence 0,77, GPU. CSV/config e video annotati sono salvati nelle rispettive run.

## Campo, zone e calibrazione

Notebook: `Zone_competenza_pallavolo.ipynb`. Calibrazione confermata dall’utente: `zone_campo_output\calibrazione_ITA_POL_VNL2024_30fps.json`, schema 3, video e risoluzione 1280×720 corrispondenti. L’omografia è una trasformazione **piana del pavimento**:

- coordinate campo: X trasversale 0–9 m, Y longitudinale 0–18 m; rete a Y=9 m;
- vertici immagine nell’ordine memorizzato: angoli alto-sinistra/alto-destra (fondo lontano), basso-destra/basso-sinistra (fondo vicino);
- discretizzazione zone: X `[0, 3, 6, 9]` m e Y `[0, 4,5, 9, 13,5, 18]` m; zone P1–P6 sui due lati;
- estensione avversaria oltre il fondocampo: **5 m**, valore personalizzato dell’utente preservato, non misura regolamentare;
- rete femminile configurata a 2,24 m al centro; bordo inferiore nominale 1,24 m, antenna 0,80 m sopra il nastro e punta nominale 3,04 m.

La calibrazione camera 3D nel JSON è **`not_available`** e non va usata per ricostruire posizione/altezza della palla. Il fit pinhole a 10 corrispondenze ha intrinseci al limite fisico, incertezze molto alte, RMS di riproiezione **139,38 px** e massimo **310,90 px**; pertanto il modello è rifiutato, benché l’omografia del pavimento resti disponibile. Residui registrati per punto: CORNER-1 6,59 px; CORNER-2 297,38; CORNER-3 310,90; CORNER-4 86,13; TOP-L 2,33; TOP-C 3,52; TOP-R 2,35; BOT-L 24,10; BOT-C 23,73; BOT-R 23,38. Gli errori elevati agli angoli e al bordo inferiore, i parametri al bound e le quote nominali della rete/sag impediscono di dichiarare una ricostruzione 3D valida. La camera 3D non è stata forzata disponibile.

Output utili: `zone_campo_output\calibrazione_ITA_POL_VNL2024_30fps.json`, `maschere_zone_ITA_POL_VNL2024_30fps.npz` e overlay campionati `overlay_zone_...png`. I click/configurazioni effettivi dell’utente devono essere preservati; non modificare il JSON per “migliorare” il fit senza nuove corrispondenze confermate.

## Tracking giocatori

Notebook: `Tracking_giocatori_YOLO_ByteTrack.ipynb`. Checkpoint reale giocatori, class map `{0: player, 1: refree}`; il tracking seleziona `player`. ByteTrack mantiene ID temporanei e usa un filtro interno di Kalman; gli ID non sono numeri di maglia o identità fisiche.

Configurazione attualmente salvata nel notebook:

- intervallo 360–4970 s, stride 1; GPU richiesta;
- `ROI_MODE='floor'`, omografia del pavimento della stessa camera, margine 0,25 m;
- gate solo sul lato sinistro: se X < −0,25 m per 10 update consecutivi, l’ID è escluso; a 30 fps/stride 1 sono circa 0,33 s. X ≥ −0,25 m al rientro azzera il contatore e riabilita l’ID. Destra e fondocampo non attivano questo gate;
- cap attivo: massimo 6 ID per metà geometrica (A: Y<9 m; B: Y≥9 m), quindi massimo 12 track restituite per frame. Il cap non fabbrica persone e non garantisce che l’identità sia quella corretta.

Le run correnti registrano `outside_gate_version=post_association_left_sideline_only_v3` e `side_cap_enabled=true`.

Il cap richiede coordinate pavimento valide e `ROI_MODE='floor'`; con modalità `image` o `none` occorre disabilitarlo esplicitamente, perdendo la garanzia 6/12. Se più di 6 candidati validi ricadono su una metà, la priorità è: ID ammessi nel frame precedente **sulla stessa metà corrente**, poi confidence della detection corrente decrescente, anzianità della traccia, ID crescente. I piedi sono stimati dal centro del bordo inferiore della bounding box: salti, occlusioni e bounding box imprecise possono spostare coordinate e metà. Le metà sono coordinate geometriche, non squadre riconosciute da divise o tattica. Il gate/cap non identificano un allenatore; possono escludere un atleta fuori dal lato sinistro o preferire un falso positivo stabile.

Sono state eseguite prove video brevi di 3 s (90 frame) per il gate laterale e per il cap, con override dell’intervallo in memoria; nei controlli effettuati il coach rilevato a sinistra è stato escluso dopo il decimo update e i conteggi sono rimasti entro 6 per metà. Le rispettive cartelle di prova non risultano più presenti nella cartella al momento di questo riepilogo: i relativi output non sono quindi allegati né presentati come run attualmente riproducibili.

Run breve attualmente conservata: `runs\player_tracking\ITA_POL_VNL2024_30fps_20261117_121636_6330a7`, 1.500 frame nell’intervallo 1150–1200 s. Run completa registrata dall’utente: `runs\player_tracking\ITA_POL_VNL2024_30fps_20261117_122249_a87973`, 138.300 frame nell’intervallo 360–4970 s, 1.125.184 osservazioni nel CSV principale; entrambe le configurazioni registrano cap attivo e margine 0,25 m. La run completa ha un video di circa 5,8 GB; il suo `configurazione.json` indica stato `complete`, ma non è stata decodificata nuovamente integralmente durante la redazione di questo documento. I conteggi salvati variano da 0 a 12 e non impongono dodici giocatori quando le detection non bastano. Per riprodurre risultati, controllare la configurazione effettiva nella run.

Le prove video e i test sintetici non misurano IDF1, MOTA, ID switches o accuratezza di tracking: non è presente ground truth di giocatori. Il cap limita i risultati, ma non certifica che i 6 selezionati per metà siano i 6 atleti reali.

## Ambiente e uso

Runtime registrato nelle run: PyTorch `2.14.1+cu130`, torchvision `0.29.1+cu130`, Ultralytics `8.4.212`, CUDA/NMS verificati; GPU rilevata nelle sessioni precedenti: NVIDIA RTX PRO2000 Blackwell Laptop. I notebook di training/tracking usano workers 0 per compatibilità Windows.

Durante una precedente ispezione dell’API ByteTrack, l’autoinstall di Ultralytics ha installato `lap==0.5.14` nell’ambiente Python corrente. È stato un effetto collaterale non intenzionale; non è stato disinstallato né sono seguite altre installazioni globali. Il notebook tracking giocatori attuale imposta `YOLO_AUTOINSTALL=false` prima di importare Ultralytics e fa preflight delle dipendenze: se ne manca una, si ferma invece di installarla autonomamente. CUDA e NMS sono testate; il fallback CPU è esplicito nei notebook che lo supportano.

Per ottenere nuovi risultati:

1. Aprire il notebook aggiornato dalla cartella di lavoro con il kernel Python che contiene le dipendenze; riavviare il kernel ed eseguire le celle dall’inizio.
2. Controllare percorso video, checkpoint, split/config dataset, soglie e intervallo prima di lanciare training o inferenza. Il tracking giocatori è attualmente impostato su 360–4970 s: per una prova breve ridurre l’intervallo.
3. Verificare che CUDA/NMS siano effettivamente disponibili; scegliere CPU solo come fallback intenzionale.
4. Leggere la configurazione generata nella nuova directory `runs\...`; le run non riutilizzano automaticamente CSV precedenti. Per il cap giocatori verificare `side_cap_enabled`, margine, gate version e conteggi per frame.
5. Valutare un checkpoint con gli split appropriati; non usare il test per scegliere il modello. Metriche tracking richiedono annotazioni GT specifiche.

Questo riepilogo è stato creato il 17 novembre 2026 leggendo notebook, YAML, JSON, CSV e percorsi presenti. Nessun notebook, dataset, checkpoint, calibrazione o output di run è stato modificato per redigere il documento.
