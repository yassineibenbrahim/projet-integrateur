# mars-collect : collecte et formatage (MARS Axe 1)

Couche de collecte qui transforme des sources hétérogènes (API YouTube, transcriptions, CSV) en un
**corpus JSONL au schéma commun**, pseudonymisé dès l'ingestion. Seuls les *collectors* et les
*normalizers* dépendent de la plateforme ; tout le reste est commun.

```
cible ─► Collector ─► Normalizer ─► clean_text + PrivacyGuard ─► Document (Pydantic) ─► JSONL
(ID vidéo,  (par source)  (vers schéma)   (NFC, masquage PII, HMAC)    (validation)      + journal
 CSV)
```

## Installation et lancement

```bash
pip install -e ".[transcripts,dev]"
export MARS_ANONYMIZATION_SALT="une-longue-cle-secrete-de-16-caracteres-minimum"   # jamais dans Git
export YOUTUBE_API_KEY="..."
python -m mars_collect run --config config/example.yaml        # ou : mars-collect run --config ...
python -m mars_collect list-collectors
pytest                                                          # 58 tests, aucun accès réseau


# poweshell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e ".[transcripts,dev]"
$env:MARS_ANONYMIZATION_SALT = "une-cle-de-16-caracteres-minimum"
$env:YOUTUBE_API_KEY="..."
python -m mars_collect run --config config\example.yaml
pytest
```

Sorties dans `output_dir` : `corpus.jsonl`, `run_<date>.json` (journal et rapport de couverture),
`checkpoint.json`.

## Schéma commun (`schema.py`, version 1.0)

| Champ | Description |
|---|---|
| `doc_id` | `plateforme:type:ref_<HMAC>`, stable (idempotence). L'identifiant externe (ID de commentaire, de vidéo) n'est **jamais** écrit : il permettrait de retrouver le contenu et son auteur |
| `source` | plateforme ou fichier d'origine |
| `type` | `post` (titre + description), `transcript`, `comment`, `reply` |
| `text` | texte nettoyé (NFC, espaces) et purgé des PII ; **émojis et ponctuation conservés** |
| `author_pseudo` | `actor_<HMAC>` ou `null` si l'auteur est inconnu |
| `date` | ISO 8601 UTC, **arrondie au jour par défaut** (`privacy.date_precision`), ou `null` |
| `parent_id` | `doc_id` du parent (haché lui aussi) : reconstitue le fil réponse → commentaire → post |
| `collected_at`, `tool`, `tool_version` | traçabilité de la collecte |
| `metadata` | champs propres à la plateforme (langue, tags…) ; compteurs (likes, vues) **en tranches** par défaut ; pas d'ID de vidéo ; PII masquées |
| `schema_version` | version du schéma |

## Politique de confidentialité (`privacy` dans le YAML)

La pseudonymisation des auteurs ne suffit pas : retrouver un commentaire par son identifiant ou par son
texte ramène à son auteur. Défauts (protection maximale compatible avec l'analyse) :

| Clé | Défaut | Effet |
|---|---|---|
| `id_policy` | `hashed` | identifiants de vidéos et de commentaires remplacés par un HMAC (`clear` : pour le débogage, jamais pour un corpus partagé) |
| `date_precision` | `day` | dates arrondies au jour (`exact` possible) |
| `coarse_counts` | `true` | likes, vues, réponses en tranches (`21-100`), durée de transcription à la minute |

**Reste vrai malgré tout** : le texte des commentaires, et le titre et la description des vidéos, restent dans
le corpus (c'est l'objet de l'analyse) et peuvent être recherchés. Le corpus ne doit donc pas être publié, et
aucun commentaire ne doit être cité tel quel dans un rapport. Le journal `run_*.json` et `checkpoint.json`
contiennent la liste des cibles (ID de vidéos en clair) : ce sont des fichiers internes, à ne pas diffuser avec le
corpus. Pour effacer un contenu à la demande de son auteur, recalculer le haché de son identifiant (même clé)
suffit : aucune table de correspondance n'est nécessaire, et il n'y en a pas.

## Éléments

| Fichier | Rôle |
|---|---|
| `schema.py` | modèle Pydantic `Document` : le contrat avec le NLP, IRaMuteQ et le front |
| `privacy.py` | HMAC-SHA256 **par plateforme** (non reliable entre plateformes), masquage e-mail/URL/mention/téléphone |
| `base.py`, `registry.py` | interface `Collector` (flux d'enregistrements, capacités, statuts) et registre |
| `collectors/youtube_api.py` | API v3 : vidéo, commentaires, réponses, pagination, quota, reprises sur 5xx |
| `collectors/youtube_transcript.py` | transcriptions (youtube-transcript-api), optionnel, sans contournement |
| `collectors/csv_import.py` | source MVP, rejette les lignes invalides avec leur numéro |
| `normalizers/` | un fichier par plateforme : brut → `Candidate` |
| `pipeline.py` | orchestration par cible, rapport de couverture, checkpoint |
| `writer.py` | JSONL idempotent, reprise après arrêt brutal |

## Ajouter une plateforme

1. `collectors/reddit.py` : une classe `@register` avec `name`, `platform`, `capabilities` et
   `collect(target)` qui produit des `RawRecord` et appelle `set_status(...)`.
2. `normalizers/reddit.py` : des fonctions `@normalizer("reddit", "<kind>")` qui renvoient un `Candidate`.
3. Ajouter la source dans le YAML. Les tests de `tests/contract.py` valident n'importe quel corpus.

## Statuts du rapport de couverture

`ok`, `comments_disabled`, `no_captions`, `captions_disabled`, `not_found`, `invalid_rows`
(terminaux : la cible n'est pas rejouée) ; `quota_exceeded`, `blocked`, `error` (rejouables à la
relance suivante) ; `skipped_done`, `skipped_quota`.

## Limites connues

- **Pseudonymisation, pas anonymisation** : un HMAC déterministe reste une donnée personnelle (RGPD) ; la rotation de la clé change tous les pseudonymes et toutes les références.
  Les noms propres écrits dans un texte ne sont pas détectés par les regex.
- Reprise **par cible** (pas par page) : un arrêt en cours de vidéo relit cette vidéo ; les doublons
  sont évités par `doc_id`, mais le quota est reconsommé.
- Les transcripts n'ont pas d'auteur (`author_pseudo = null`) et la bibliothèque est non officielle :
  endpoints internes, blocages d'IP cloud, sous-titres automatiques peu fiables sur l'argot.
- Le texte des CSV n'est pas déséchappé du HTML (seul YouTube l'est).
- Non couvert ici : déduplication textuelle, décodage de l'algospeak, export IRaMuteQ/Gephi
  (étapes de prétraitement et d'export, à venir).
