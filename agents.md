# 🎯 Projekt- und Agenten-Spezifikation

## 1. Projektübersicht

**Projektname**: Open Notebook  
**Ziel**: 100% lokale, KI-gestützte Wissensdatenbank mit RAG (Retrieval Augmented Generation)  
**Dev-Host**: Proxmox VE (192.168.178.142)  
**LLM-Host**: Strix Halo + RTX Pro 6000 Blackwell (192.168.178.144)  
**Storage**: Proxmox SSD (/mnt/pve/data_2/filebrowser/open-notebook/)  
**Domain**: https://nb.hcmtech.de  
**Privacy**: 100% lokal - nichts verlässt das Netzwerk

### 1.1 Kernaufgaben
- Dokumenten-Verarbeitung und -Indexierung
- Speech-to-Text (WhisperX)
- Dokumenten-Embeddings (Qwen3-Embedding-0.6B)
- RAG-basierte Wissensabfrage
- Graph-basierte Wissensvernetzung (SurrealDB)

---

## 2. Architektur

### 2.1 Komponenten

| Komponente | Image/Version | Port | Beschreibung |
|------------|---------------|------|--------------|
| **SurrealDB** | `surrealdb/surrealdb:v2` | 8001 | Vektor-Datenbank + Graph |
| **Open Notebook App** | `lfnovo/open_notebook:v1-latest` | 3001 (UI), 5056 (API) | Hauptanwendung |
| **llama-swap** | - | 8082 | LLM-Proxy (Gemma4-31B) |
| **WhisperX** | - | 9000 | Speech-to-Text |
| **Qwen3-Embedding** | - | 9000+ | Embedding-Modell (neu) |
| **Docling** | - | 1002 | Dokumenten-Konvertierung |

### 2.2 Netzwerk-Konfiguration
- **Modus**: Bridge (vermeidet Port-Konflikte)
- **Container-Namen**: `open-notebook-surrealdb`, `open-notebook-app`
- **Direkter Zugriff**: 192.168.178.142 (Docker Host)
- **Nginx Reverse Proxy**: nb.hcmtech.de → 192.168.178.142:3001

---

## 3. Technische Stack

### 3.1 Backend
- **Datenbank**: SurrealDB v2 (RocksDB Persistence)
- **API**: Python FastAPI (Open Notebook)
- **Embeddings**: Qwen3-Embedding-0.6B (32K Kontext)
- **LLM**: Gemma4-31B (via llama-swap)
- **STT**: WhisperX (NVIDIA GPU)

### 3.2 Frontend
- **Framework**: Next.js (im Open Notebook Container)
- **Zugriff**: https://nb.hcmtech.de/notebooks

### 3.3 Infrastruktur
- **Container**: Docker + Docker Compose
- **GPU**: NVIDIA RTX Pro 6000 Blackwell
- **Context Size**: 256k (262144 tokens)

---

## 4. Umgebungsvariablen (Critical)

```yaml
# Database
SURREAL_URL=ws://192.168.178.142:8001/rpc
SURREAL_USER=root
SURREAL_PASSWORD=root
SURREAL_NAMESPACE=open_notebook
SURREAL_DATABASE=open_notebook

# API
API_URL=http://192.168.178.142:5056
INTERNAL_API_URL=http://localhost:5055
API_CLIENT_TIMEOUT=1800

# LLM (OpenAI-Compatible)
OPENAI_COMPATIBLE_BASE_URL_LLM=http://192.168.178.144:8082/v1
OPENAI_COMPATIBLE_API_KEY_LLM=not-needed

# STT (OpenAI-Compatible)
OPENAI_COMPATIBLE_BASE_URL_STT=http://192.168.178.144:9000
OPENAI_COMPATIBLE_API_KEY_STT=not-needed

# Chunking (CRITICAL - Tokenizer-Mismatch Problem)
OPEN_NOTEBOOK_CHUNK_SIZE=450  # Wird auf 300 geändert für Qwen3
OPEN_NOTEBOOK_CHUNK_OVERLAP=50

# Timeouts
ESPERANTO_LLM_TIMEOUT=1200
ESPERANTO_STT_TIMEOUT=3600

# CORS
CORS_ORIGINS=https://nb.hcmtech.de,http://nb.hcmtech.de
```

---

## 5. Bekannte Probleme & Lösungen

### 5.1 Embedding Tokenizer-Mismatch (Gelöst → Wechsel zu Qwen3)

**Problem**: Open Notebook nutzt `tiktoken` (o200k_base), mxbai-embed-large nutzt BERT WordPiece.  
**Symptom**: 450 konfigurierte Token → 607 tatsächliche Token → Fehler "input larger than max context size (512)"  
**Fehler**: `input (607 tokens) is larger than the max context size (512 tokens). skipping`

**Lösung**: Wechsel zu **Qwen3-Embedding-0.6B**
- 32K Kontext-Fenster (statt 512)
- Bessere Token-Effizienz
- Multilingual (100+ Sprachen)
- Größe: 639 MB (ähnlich wie mxbai)

**Status**: Qwen3-Embedding-0.6B wird vom User installiert.

### 5.2 Frontend Health Check Bug

**Problem**: Open Notebook wartet auf `/api/health` (404), korrekter Endpoint ist `/health`  
**Workaround**: Health Check ignoriert, Container-Status manuell prüfen

### 5.3 CORS-Konfiguration

**Problem**: Nginx Reverse Proxy verursacht CORS-Fehler  
**Lösung**: `CORS_ORIGINS=https://nb.hcmtech.de,http://nb.hcmtech.de` in docker-compose.yml

---

## 6. Dateistruktur

```
/mnt/pve/data_2/filebrowser/open-notebook/
├── docker-compose.yml          # Container-Konfiguration
├── .env                        # Umgebungsvariablen (optional)
├── surreal_data/               # SurrealDB Persistent Storage
└── notebook_data/              # Open Notebook Persistent Storage
```

**Development Directory**: `/mnt/c/Users/T11/SynologyDrive/LLM/open-notebook/`

---

## 7. Agenten-Verfügbarkeit

### 7.1 Build Subagent
- **Zweck**: Standard-Ausführung
- **Tools**: Alle verfügbaren Tools (basierend auf Permissions)

### 7.2 Explore Subagent
- **Zweck**: Codebase-Erkundung, Pattern-Suche
- **Tools**: Grep, AST-grep, LSP

### 7.3 Librarian Subagent
- **Zweck**: Externe Dokumentation, GitHub-Beispiele
- **Tools**: Context7, Web Search, GitHub CLI

### 7.4 Oracle Subagent
- **Zweck**: Komplexe Architektur, Debugging
- **Status**: Read-only, teuer, hochwertige Reasoning

---

## 8. Entwicklungsroutinen

### 8.1 Container-Management
```bash
# Status prüfen
ssh docker-a "docker ps --filter 'name=open-notebook'"

# Logs ansehen
ssh docker-a "docker logs open-notebook-app --tail 200"

# Neu starten
ssh docker-a "cd /mnt/pve/data_2/filebrowser/open-notebook && docker compose down && docker compose up -d"
```

### 8.2 Konfiguration ändern
```bash
# docker-compose.yml bearbeiten
ssh docker-a "nano /mnt/pve/data_2/filebrowser/open-notebook/docker-compose.yml"

# Änderungen anwenden
ssh docker-a "cd /mnt/pve/data_2/filebrowser/open-notebook && docker compose up -d"
```

### 8.3 Nginx-Proxy
```bash
# Konfiguration
/etc/nginx/sites-available/nb.hcmtech.de

# Neu laden
sudo systemctl reload nginx
```

---

## 9. Sicherheit & Privacy

### 9.1 Datenschutz
- **100% lokal**: Keine Cloud-APIs
- **Maximum Privacy**: Nichts verlässt das Netzwerk
- **Verschlüsselung**: `OPEN_NOTEBOOK_ENCRYPTION_KEY` muss gesetzt werden

### 9.2 Netzwerk
- **Isoliert**: Nur im lokalen Netzwerk (192.168.178.x)
- **Reverse Proxy**: Nginx mit SSL (Let's Encrypt)
- **CORS**: Explizit auf nb.hcmtech.de beschränkt

---

## 10. Monitoring & Logging

### 10.1 Container-Logs
```bash
ssh docker-a "docker logs open-notebook-app --tail 200"
ssh docker-a "docker logs open-notebook-surrealdb --tail 100"
```

### 10.2 Health Checks
- **Open Notebook**: `http://192.168.178.142:5055/health`
- **SurrealDB**: Manuell prüfen (kein Health Check im Container)

### 10.3 Nginx-Logs
```bash
ssh docker-a "tail -f /var/log/nginx/nb.hcmtech.de.error.log"
```

---

## 11. Test & Verification

### 11.1 UI Zugriffs-Test
```bash
curl -I https://nb.hcmtech.de/notebooks
# Erwartet: HTTP/2 200
```

### 11.2 API Health-Test
```bash
curl -s http://192.168.178.142:5055/health
# Erwartet: {"status":"ok"} oder ähnlich
```

### 11.3 LLM-Verbindung
```bash
curl -s http://192.168.178.144:8082/v1/models
# Erwartet: Liste verfügbarer Modelle (Gemma4-31B)
```

### 11.4 Embedding-Verbindung
```bash
curl -s http://192.168.178.144:9000/v1/models
# Erwartet: Qwen3-Embedding-0.6B (nach Installation)
```

---

## 12. Performance-Optimierung

### 12.1 Chunking-Parameter
- **Aktuell**: `OPEN_NOTEBOOK_CHUNK_SIZE=450`
- **Geplant**: `OPEN_NOTEBOOK_CHUNK_SIZE=300` (für Qwen3)
- **Overlap**: `OPEN_NOTEBOOK_CHUNK_OVERLAP=50`

### 12.2 Timeouts
- **API Client**: 1800s (30 Minuten)
- **LLM Operations**: 1200s (20 Minuten)
- **STT Operations**: 3600s (1 Stunde)

### 12.3 Batch-Verarbeitung
- **Max. Dateien pro Upload**: 40+ möglich (durch erhöhte Timeouts)
- **Parallelität**: Container-interne Worker-Threads

---

## 13. Known Issues & Workarounds

| Issue | Severity | Status | Workaround |
|-------|----------|--------|------------|
| Embedding Tokenizer-Mismatch | High | → Qwen3 | Chunk-Size reduzieren |
| Frontend Health Check Bug | Low | Ignoriert | Manuell prüfen |
| Bulk Upload Timeouts | Medium | Gelöst | Timeouts erhöht |

---

## 14. Nächste Schritte (Roadmap)

### Phase 1: Qwen3-Embedding-0.6B Integration (AKTUELL)
1. ✅ Qwen3-Embedding-0.6B installieren (durch User)
2. ⏳ `OPENAI_COMPATIBLE_BASE_URL_STT` auf neuen Port ändern
3. ⏳ Chunk-Size auf 300 anpassen (nicht mehr kritisch, aber empfohlen)
4. ⏳ Test-Upload durchführen

### Phase 2: Optimierung
5. Dokumenten-Verarbeitung testen (via Docling)
6. Speech-to-Text Integration testen (via WhisperX)
7. RAG-Abfragen validieren

### Phase 3: Production Hardening
8. Backup-Strategie für SurrealDB implementieren
9. Monitoring einrichten
10. Security-Hardening (Encryption Key rotieren)

---

## 15. Glossar & Referenzen

### 15.1 Begriffe
- **RAG**: Retrieval Augmented Generation
- **STT**: Speech-to-Text
- **LLM**: Large Language Model
- **Embedding**: Vektor-Repräsentation von Text
- **Chunking**: Aufteilung von Dokumenten in kleinere Einheiten

### 15.2 Referenzen
- **Open Notebook**: https://github.com/lfnovo/open_notebook
- **SurrealDB**: https://surrealdb.com
- **Qwen3-Embedding**: https://github.com/QwenLM/Qwen3-Embedding
- **mxbai-embed-large**: https://huggingface.co/mixedbread-ai/mxbai-embed-large-v1
- **llama-swap**: (lokale Installation auf 192.168.178.144)

---

## 16. Kontakt & Support

**Primärer Entwickler**: User (T11)  
**Dev-Host**: 192.168.178.142 (Proxmox)  
**LLM-Host**: 192.168.178.144 (Strix Halo + RTX Pro 6000)  
**Domain**: https://nb.hcmtech.de

---

**Letzte Aktualisierung**: 2026-06-15  
**Version**: 1.0  
**Status**: Active Development
